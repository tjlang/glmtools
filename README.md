# glmtools

GOES Geostationary Lightning Mapper Tools

[![DOI](https://zenodo.org/badge/71308485.svg)](https://zenodo.org/badge/latestdoi/71308485)

## Installation
glmtools requires Python 3.5+ and provides a conda `environment.yml` for the key dependencies.

See the documentation in `docs/index.rst` for complete installation instructions.

## Description
Compatible data:
- NetCDF format Level 2 data, as described in the [Product Definition and Users Guide](https://www.goes-r.gov/resources/docs.html)

glmtools automatically reconstitutes the parent-child relationships implicit in
the L2 GLM data and adds traversal information to the dataset:

- calculating the parent flash id for each event
- calculating the number of groups and events in each flash
- calculating the number of events in each group

xarray's dimension-aware indexing lets you quickly reduce the dataset to 
flashes of interest, as described below.

glmtools can restore the GLM event geometry using a built-in corner-point lookup table,
which allows for gridding of the imagery at finer resolutions that accurately represent
the full footprint of each event, group, and flash.

## Some common tasks

### Create gridded NetCDF imagery

Use the script in `examples/grid/make_GLM_grids.py`. For instance, the following command
will grid one minute of data (3 GLM files) on the ABI fixed grid in the CONUS sector at 2
km resolution. These images will overlay precisely on the ABI cloud tops, and will have
parallax with respect to ground for all the same reasons ABI does.

```bash 
python make_GLM_grids.py -o /path/to/output/ --fixed_grid --split_events \
--goes_position east --goes_sector conus --dx=2.0 --dy=2.0 --ctr_lon 0.0 --ctr_lat 0.0 \
--start=2018-01-04T05:37:00 --end=2018-01-04T05:38:00 \
OR_GLM-L2-LCFA_G16_s20180040537000_e20180040537200_c20180040537226.nc \
OR_GLM-L2-LCFA_G16_s20180040537200_e20180040537400_c20180040537419.nc \
OR_GLM-L2-LCFA_G16_s20180040537400_e20180040538000_c20180040538022.nc \
```

To start with, look at the flash extent density and total energy grids.

`ctr_lon` and `ctr_lat` aren't used, but are required anyway. Fixing this would
make a nice first contribution!

Removing the --split_events flag and setting the grid to 10 km allows for gridding
of the raw point data, and will run much faster. Finer resolutions will cause gaps in
flash extent density because the point data are spaced about 8-12 km apart.
Note that these grids will either have gaps or will double-count events along
GLM pixel borders, because there is no one grid resolution which exactly
matches the GLM pixel size as it varies with earth distortion over the field
of view.

### Interactively plot raw flash data

See the examples folder. `plot_glm_test_data.ipynb` is a good place to start.

### Reduce the dataset to a few flashes

```python
from glmtools.io.glm import GLMDataset
filename = 'OR_GLM-L2-LCFA_G16_s20180040537000_e20180040537200_c20180040537226.nc'
glm =  GLMDataset(filename)
flash_id_list = glm.dataset.flash_id[20:30]
smaller_dataset = glm.get_flashes(flash_id_list)
```

### Filter out flashes geographically or by events/groups per flash

See `glmtools.io.glm.GLMDataset.subset_flashes`.

The logic implemented above is pretty simple, and below shows how to adapt it to find large flashes.

```python
from glmtools.io.glm import GLMDataset
filename = 'OR_GLM-L2-LCFA_G16_s20180040537000_e20180040537200_c20180040537226.nc'
glm =  GLMDataset(filename)
fl_idx = glm.dataset['flash_area'] > 2000
flash_ids = glm.dataset[{glm.fl_dim: fl_idx}].flash_id.data
smaller_dataset = glm.get_flashes(flash_ids)
print(smaller_dataset)
```

