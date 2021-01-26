---
jupytext:
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.12
    jupytext_version: 1.9.1
kernelspec:
  display_name: Python 3
  language: python
  name: python3
---

# specMACS cloudmask

The following script exemplifies the access and usage of specMACS data measured 
during EUREC4A.  

More information on the dataset can be found at https://macsserver.physik.uni-muenchen.de/campaigns/EUREC4A/products/cloudmask/. If you have questions or if you would like to use the data for a publication, please don't hesitate to get in contact with the dataset authors as stated in the dataset attributes `contact` and `author` list.

```{code-cell} ipython3
%pylab inline
```

## Get data
* To load the data we first load the EUREC4A meta data catalogue. More information on the catalog can be found [here](https://github.com/eurec4a/eurec4a-intake#eurec4a-intake-catalogue).

```{code-cell} ipython3
import eurec4a
```

```{code-cell} ipython3
cat = eurec4a.get_intake_catalog()
list(cat)
```

* We can funrther specify the platform, instrument, if applicable dataset level or variable name, and pass it on to dask.

*Note: have a look at the attributes of the xarray dataset `ds` for all relevant information on the dataset, such as author, contact, or citation infromation.*

```{code-cell} ipython3
ds = cat.HALO.specMACS.cloudmaskSWIR["HALO-0205"].to_dask()
ds
```

## Load HALO flight phase information
All HALO flights were split up into flight phases or segments to allow for a precise selection in time and space of a circle or calibration pattern. For more information have a look at the respective [github repository](https://github.com/eurec4a/halo-flight-phase-separation).

```{code-cell} ipython3
meta = eurec4a.get_flight_segments()
```

We further select the first straight leg on February 5 by it's `segment_id`.

```{code-cell} ipython3
segments = {s["segment_id"]: {**s, "flight_id": flight["flight_id"]}
             for platform in meta.values()
             for flight in platform.values()
             for s in flight["segments"]
            }
seg = segments["HALO-0205_sl1"]
```

We transfer the information from our flight segment selection to the specMACS data in the xarray dataset.

```{code-cell} ipython3
ds_selection = ds.sel(time=slice(seg["start"], seg["end"]))
```

## Plots
Figure 1: shows the SWIR camera cloud mask product along the flight track (x axis) for all observations in accross track directions (y axis).  

You can get a list of available variables in the dataset from `ds_selection.variables.keys()`  
*Note: fetching the data and displaying it might take a few seconds*

```{code-cell} ipython3
fig, ax = plt.subplots(figsize=(16,4))
ds_selection.cloud_mask.T.plot(ax=ax, cmap="gray")
```

## Conversion from camera view angles to latitude and longitude

```{code-cell} ipython3
import intake
cat_experimental = intake.open_catalog("https://raw.githubusercontent.com/d70-t/eurec4a-intake/cloudmaskSWIRv2/catalog.yml")
ds = cat_experimental.HALO.specMACS.cloudmaskSWIR["HALO-0202"].to_dask()
seg = segments["HALO-0202_c1"]
ds_selection = ds.sel(time=slice(seg["start"], seg["end"]))
```

The cloud mask is given on a `time x angle` grid where `angles` are the internal camera angles. Sometimes it is helpful to project the data onto a map. This means that we would like to know the corresponding **latitude** and **longitude** coordinates of the cloud mask.
We need five variables for this:
* position of the airplane: latitude, longitude and height (`ds.lat, ds.lon, ds.height`)
* viewing directions of the camera pixels: viewing zenith angle and viewing azimuth angle (`ds.vza, ds.vaa`)

Let's have a look a these variables

```{code-cell} ipython3
for key in ["lat", "lon", "height", "vza", "vaa"]:
    print(key, ds_selection[key].units)
    if "comment" in ds_selection[key].attrs:
        print(key, ds_selection[key].comment)
    print("\n")
```

* The position of the HALO is saved in ellipsoidal coordinates. It is defined by the lat/lon/height coordinates with respect to the WGS-84 ellipsoid.
* On the other hand the viewing zenith and azimuth angles are given with respect to the local horizon (lh) coordinate system at the position of the HALO. This system has its center at the lat/lon/height position of the HALO and the x/y/z axis point into North, East and down directions.
A convenient way to work with such kind of data is to transform it into the Earth-Centered, Earth-Fixed (ECEF) coordinate system. The origin of this coordinate system is the center of the Earth. The z-axis passes through true north, the x-axis through the Equator and the 0° longitude and the y-axis is orthogonal to x and z. This cartesian system makes computations of distances and angles very easy. 

In a first step we want to transform the position of the HALO into the ECEF coordinate system. We use the method presented here: https://gssc.esa.int/navipedia/index.php/Ellipsoidal_and_Cartesian_Coordinates_Conversion

```{code-cell} ipython3
def ellipsoidal_to_ecef(lat, lon, height):
    '''
    lat: latitude of ellipsoid [rad]
    lon: longitude of ellipsoid [rad]
    height: height above ellipsoid [m]
    see: https://gssc.esa.int/navipedia/index.php/Ellipsoidal_and_Cartesian_Coordinates_Conversion
    '''
    
    WGS_84_dict = {"axes": (6378137.0, 6356752.314245)} #m
    a, b = WGS_84_dict["axes"]
    
    e_squared = e2(a, b)
    N = calc_N(a, e_squared, lat)

    x = (N + height) * np.cos(lat) * np.cos(lon)
    y = (N + height) * np.cos(lat) * np.sin(lon)
    z = ((1 - e_squared)*N + height) * np.sin(lat)
    return np.array([x,y,z])

def calc_N(a, e_squared, lat):
    '''
    N: radius of curvature in the prime vertical
    e_squared: eccentricity
    lat: latitude [rad]
    '''
    return a/np.sqrt(1 - e_squared*(np.sin(lat))**2)

def e2(a, b):
    '''
    e: eccentricity
    a: semi-major axis
    b: the semi-minor axis b 
    f: flattening factor f=1−ba
    '''
    f = 1 - (b/a)
    e_squared = 2*f - f**2
    return e_squared
```

With these functions it is very easy to transform the lat/lon/height position of the HALO into the ECEF-coordinate system:

```{code-cell} ipython3
HALO_ecef = ellipsoidal_to_ecef(np.deg2rad(ds_selection["lat"]), 
                                np.deg2rad(ds_selection["lon"]), 
                                ds_selection["height"])
```

As a next step we want to set up the vector of the viewing direction. We will need the rotation matrices Rx, Ry and Rz for this.

```{code-cell} ipython3
def Rx(alpha):
    return np.array([[np.ones(alpha.shape), np.zeros(alpha.shape), np.zeros(alpha.shape)],
                     [np.zeros(alpha.shape), np.cos(alpha), np.sin(alpha)],
                     [np.zeros(alpha.shape), -np.sin(alpha), np.cos(alpha)]])
def Ry(alpha):
    return np.array([[np.cos(alpha), np.zeros(alpha.shape), -np.sin(alpha)],
                     [np.zeros(alpha.shape), np.ones(alpha.shape), np.zeros(alpha.shape)],
                     [np.sin(alpha), np.zeros(alpha.shape),  np.cos(alpha)]])
def Rz(alpha):
    return np.array([[ np.cos(alpha),  np.sin(alpha), np.zeros(alpha.shape)],
                     [-np.sin(alpha),  np.cos(alpha), np.zeros(alpha.shape)],
                     [ np.zeros(alpha.shape), np.zeros(alpha.shape), np.ones(alpha.shape)]])
```

Now we can calculate the viewing direction relative to the local horizon coordinate system.

```{code-cell} ipython3
ds_selection.load()
```

```{code-cell} ipython3
def vector_lh(ds):
    down = np.array([0,0,1]) #
    view_lh = np.einsum("ijlm,jlm->iml", Rz(np.deg2rad(-ds["vaa"]))[...], np.einsum("ijlm, j->ilm", Ry(-np.deg2rad(ds["vza"])), down)) #last step: -> iml then this is transposed by (0,2,1)  
    return view_lh/np.linalg.norm(view_lh, axis = 0)

view_lh = vector_lh(ds_selection)
```

Now we want to calculate the length of the vector which connects the HALO and the cloud. We need an approximation of the cloud top height and will use 1000 m as a first guess.

```{code-cell} ipython3
cth = 1000 #m cloud top height
viewpath_lh = view_lh * np.abs(ds_selection["height"].values - cth/view_lh[2])
```

Now we would like to transform this viewpath also into the ECEF coordinate system. We will stick to the method described here: https://gssc.esa.int/navipedia/index.php/Transformations_between_ECEF_and_ENU_coordinates

```{code-cell} ipython3
def lh_to_ecef(enu_vector, lat, lon):
    #lh: local horizon
    #following: https://gssc.esa.int/navipedia/index.php/Transformations_between_ECEF_and_ENU_coordinates adjusted to NED coordinate system.
    rot_matrix_inv = np.einsum('mn...,nd...->md...',Ry(np.deg2rad(np.ones(lon.shape)*90)), 
                               np.einsum('mn...,nd...->md...',Rx(-np.deg2rad(lon)),Ry(np.deg2rad(lat))))
    ecef_vector = np.einsum('mn...,nd...->md...',rot_matrix_inv, enu_vector)
    return ecef_vector

viewpath_ecef = lh_to_ecef(viewpath_lh, ds_selection["lat"], 
                           ds_selection["lon"])
```

If we add the viewpath to the position of the HALO the resulting point gives us the ECEF coordinates of the point on the cloud.

```{code-cell} ipython3
cloudpoint_ecef = HALO_ecef[..., np.newaxis] + viewpath_ecef.transpose(0,2,1)
x_cloud, y_cloud, z_cloud = cloudpoint_ecef
```

The last step is to transform these coordinates into ellipsoidal coordinates. Again we stick to the description here: https://gssc.esa.int/navipedia/index.php/Ellipsoidal_and_Cartesian_Coordinates_Conversion

```{code-cell} ipython3
def ecef_to_ellipsoidal(x, y, z, iterations=10):
    '''see: https://gssc.esa.int/navipedia/index.php/Ellipsoidal_and_Cartesian_Coordinates_Conversion
    x, y, z: cartesian coordinates (ECEF)
    iterations: increase this value if the change between two successive values 
                of latitude are larger than the precision required.
    '''
    WGS_84_dict = {"axes": (6378137.0, 6356752.314245)} #m
    a, b = WGS_84_dict["axes"] #semi-major and semi-minor axis a and b of the WGS-84 ellipsoid
        
    lon = np.arctan2(y,x) #radians    
    e_squared = e2(a, b)    
    p = np.sqrt(x**2 + y**2)     
    lat = np.arctan2(z, (1 - e_squared)*p)    
    
    for i in range(iterations):
        N = calc_N(a, e_squared, lat)
        height = p / np.cos(lat) - N
        lat = np.arctan2(z, (1-e_squared * (N/(N+height)))*p)
    return [np.rad2deg(lat), np.rad2deg(lon), height]


cloudlat, cloudlon, cloudheight = ecef_to_ellipsoidal(x_cloud, y_cloud, z_cloud, iterations=10)
ds_selection.coords["cloudlon"] = (("time", "angle"), cloudlon, {'units': 'degree east'})
ds_selection.coords["cloudlat"] = (("time", "angle"), cloudlat, {'units': 'degree north'})
```

`ds.cloudlon` and `ds.cloudlat` are the projected longitude and latitude coordinates of the cloud. Now it is possible to plot the cloudmask on a map!

```{code-cell} ipython3
import cartopy.crs as ccrs
from matplotlib import pyplot as plt

fig = plt.figure(figsize=(12, 6))
ax = plt.axes(projection=ccrs.PlateCarree())
img = ax.contourf(ds_selection["cloudlon"], ds_selection["cloudlat"], 
                  ds_selection["cloud_mask"], 
                  levels = [-1.5, -0.5, 0.5, 1.5, 2.5])
gl = ax.gridlines(crs=ccrs.PlateCarree(), draw_labels=True,
                  linewidth=2, color='gray', alpha=0.5, linestyle='--')
cbar_ax = fig.add_axes([0.4, 0.4, 0.05, 0.15])
cbar = fig.colorbar(img, cax = cbar_ax, ticks=ds_selection.cloud_mask.flag_values)
cbar.ax.set_yticklabels(ds_selection.cloud_mask.flag_meanings.split(" "))
```

```{code-cell} ipython3
#works also
plt.figure(figsize=(14,6))
ax = plt.axes(projection=ccrs.PlateCarree())
ds_selection.cloud_mask.plot.contourf(ax=ax, transform=ccrs.PlateCarree(), 
                                        x='cloudlon', y='cloudlat', add_colorbar=True)
gl = ax.gridlines(crs=ccrs.PlateCarree(), draw_labels=True,
                      linewidth=2, color='gray', alpha=0.5, linestyle='--')
```
