# How to EUREC4A

Code examples to get you started with EUREC4A data.

-> [current version of the book](https://eurec4a_staging.pages.gwdg.de/how_to_eurec4a/)

## Usage

### Building the book

If you'd like to develop on and build the How to EUREC4A book, you should:

- Clone this repository and run
- Run `pip install -r requirements.txt` (it is recommended you do this within a virtual environment)
- (Recommended) Remove the existing `How to EUREC4A/_build/` directory
- Run `jupyter-book build How to EUREC4A/`

A fully-rendered HTML version of the book will be built in `How to EUREC4A/_build/html/`.

### Enabling `jupytext`

`jupytext` is responsible to enable `MySt` formatted markdown within jupyter notebooks. It should be enabled automatically by installing `jupytext`, which is also included in the `requirements.txt`. So chances are high that you don't have to do anything special. However if that doesn't work out of the box, please have a look at their [install instructions](https://jupytext.readthedocs.io/en/latest/install.html). In particular, the command `jupyter serverextension enable jupytext` may help.

## Contributors

We welcome and recognize all contributions. You can see a list of current contributors in the [contributors tab](https://github.com/tmieslinger/how_to_eurec4a/graphs/contributors).

## Credits

This project is created using the excellent open source [Jupyter Book project](https://jupyterbook.org/) and the [executablebooks/cookiecutter-jupyter-book template](https://github.com/executablebooks/cookiecutter-jupyter-book).
