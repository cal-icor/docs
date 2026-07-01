# User Environments

## Resources

These are available to each user:

- **RAM:** 2 GB
- **CPU:** 1
- **Storage Limits:** 10G per user (configurable for special purposes)

## Kernels

The base image includes both Python and R kernels.

## Development Environments

- JupyterLab
- Jupyter Notebook
- VS Code
- RStudio
- Shiny

## Default Packages Installed For Users

Below is the list of base packages that are installed on the user images.  If you would like additional packages installed, it is *strongly* recommended that you [open an issue in the image repository](https://github.com/cal-icor/base-user-image/issues/new) and request we install it there.

When a user installs a package in to their home directory, it can take up significant amounts of space and push you close to your 10G storage limit.

### Python Packages

The following Python packages are pre-installed. The full list and available versions are found in the [base user image repository](https://github.com/cal-icor/base-user-image/blob/main/environment.yml):

- `datascience`
- `jupyter-shiny-proxy`
- `jupysql`
- `geopandas`
- `geopy`
- `openpyxl`
- `matplotlib`
- `numpy`
- `openpyxl`
- `otter-grader`
- `pandas`
- `plotly`
- `pytorch`
- `scikit-learn`
- `scipy`
- `seaborn`
- `statsmodels`
- `sympy`

### R Packages

The following R packages are pre-installed. Versions are available in the [repository](https://github.com/cal-icor/base-user-image/blob/main/install.R):

- `car`
- `colorspace`
- `devtools`
- `esquisse`
- `extrafont`
- `flexdashboard`
- `forcats`
- `forecast`
- `gdalcubes`
- `ggThemeAssist`
- `ggalluvial`
- `ggbeeswarm`
- `ggcorrplot`
- `GGally`
- `ggdist`
- `ggformula`
- `gghighlight`
- `ggmosaic`
- `ggpubr`
- `ggrepel`
- `ggridges`
- `ggtext`
- `ggthemes`
- `gridExtra`
- `gtsummary`
- `IRkernel`
- `janitor`
- `knitr`
- `leaflet`
- `Lock5Data`
- `lubridate`
- `magick`
- `mapgl`
- `mapproj`
- `maps`
- `minioclient`
- `mosaic`
- `naniar`
- `nycflights13`
- `openintro`
- `palmerpenguins`
- `plotly`
- `pwr`
- `quarto`
- `RColorBrewer`
- `remotes`
- `renv`
- `rmarkdown`
- `rstac`
- `scales`
- `see`
- `sf`
- `sjPlot`
- `socviz`
- `stars`
- `sweep`
- `terra`
- `tidymodels`
- `tidyquant`
- `tidyr`
- `tidyverse`
- `timetk`
- `viridis`
