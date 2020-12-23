---
jupytext:
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.12
    jupytext_version: 1.7.1
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

## Conversion from carmera view angles to latitude and longitude

* (we are working on a conversion and will post it here asap...)
