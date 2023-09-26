# setup-gdal

[![Validate workflow action](https://github.com/trundev/setup-gdal/actions/workflows/validate.yml/badge.svg)](https://github.com/trundev/setup-gdal/actions/workflows/validate.yml)

This action installs the [OSGeo/gdal](https://github.com/OSGeo/gdal) Python package, so the workflow can access it.

Simply running `pip install gdal` usually does not help, as [pypi.org](https://pypi.org/project/GDAL) does not include
any C extensions. This make the CI workflows where GDAL is required quite complicated, as the binaries must be build.
The action is intended to encapsulate all of these steps.

## Usage

See [action.yml](action.yml), also the [build-wheel.yml](.github/workflows/build-wheel.yml) workflow as usage example

```yaml
steps:
- uses: actions/setup-python@v4
- uses: actions/setup-gdal@main
  with:
    gdal-version: 'v3.7.2'
- run: python -m osgeo_utils.samples.gdalinfo --formats
- run: python -c "from osgeo import gdal; print(gdal.__name__)"
```

> Note: Under `macOS`, due to some issues with `DYLD_LIBRARY_PATH`, the default shell (`bash -e`) may not work.
> It is recomended to use `shell: pwsh` or `shell: python` (inlined python code).

## Inputs

All the input parameters are optional

- `gdal-version` - The GDAL version to be installed. This is actually a git-tag (or any reference) from
  [OSGeo/gdal](https://github.com/OSGeo/gdal/tags), but for Windows runners must also match the version
  to be installed by `pip`.
- `rebuild-cache` - When `true` the internal cache for specific configuration (if exists) will be discarded and
  a fresh one will be created
- `base-dir` - The base directory under `$GITHUB_WORKSPACE` to perform the build
- `cache-key-prefix` - Extra string to be added to the cache-key. To be used as a workaround, if the cache is
  incompatible between different versions of the same OS (like `macOS-11` and `macOS-12`).
- `use-conda` - Use conda, instead of GISInternals/system packages
- `extra-packages` - Install extra (optional) packages to be used by GDAL build process

**Default inputs example**
```yaml
steps:
- uses: actions/setup-gdal@main
  with:
    gdal-version: 'v3.7.2'
    rebuild-cache: 'false'
    base-dir: 'GDAL~'
    cache-key-prefix: ''
    use-conda: 'false'
    extra-packages: ''
```

## Outputs

### Github 'step' outputs

- `wheel-path` - Path to the pip-wheel generated by build process, which includes the binary modules and tools.
  This is the only file kept by the internal cache
- `osgeo-path` - Path to the `osgeo` folder under Python `importlib` structure
- `utils-path` - Path to the `osgeo_utils` folder under Python `importlib` structure. There are installed many
  of the [GDAL programs](https://gdal.org/programs/)

### Environment variables

- `GDAL_DRIVER_PATH` - Path to GDAL `plugins`, if available
- `PROJ_DATA` - Path to the [proj-data](https://proj.org/) package, if available
- `PATH` - _[Windows only]_ The variable is updated with `osgeo-path` to allow loading of `gdal.dll`
- `LD_LIBRARY_PATH` - _[Linux & Mac]_ The variable is updated with `osgeo-path` to allow loading of `libgdal.so`
- `DYLD_LIBRARY_PATH` - _[Mac only]_ The variable is updated with `osgeo-path` to allow loading of `libgdal.dylib`

**Use of outputs example**
```yaml
steps:
- uses: actions/setup-gdal@main
  id: gdal
- run: echo "wheel-path: ${{ steps.gdal.outputs.wheel-path }}"
- run: echo "osgeo-path: ${{ steps.gdal.outputs.osgeo-path }}"
- run: echo "utils-path: ${{ steps.gdal.outputs.utils-path }}"
```


## Operation details

The action checks-out the required GDAL version, then builds it for the specific Github runner environment.
This includes installation of necessary extra packages. The build process generates a complete
[Python wheel](https://wheel.readthedocs.io/) package for the specific runnable. It includes even the binaries
from the [GDAL programs](https://gdal.org/programs/) and optionally the GDAL driver plugins and
[PROJ](https://github.com/OSGeo/PROJ) package.

An [actions cache](https://github.com/actions/cache) is used to improve execution time. The cache keeps only
the generated wheel-file, which is intended to includes everything needed. In case of a cache hit, the
installed packages might be reduced only to the ones mandatory for loading of the Python modules.

> Note that, because of the reduction of installed packages, some GDAL modules may NOT work after a cached
> wheel was used, even if they were working at first run.
>
> For example `osgeo.gdal_array` needs `numpy`, but it is only installed in order to build the binaries. In
> all cases, such modules should be explicitly installed by separate workflow step, like `pip install numpy`.

For a purpose of investigation of any issues with this Python wheel-file, it can be obtained as an artifact
from the [build-wheel.yml](.github/workflows/build-wheel.yml) workflow, see
[Build GDAL python wheel](https://github.com/trundev/setup-gdal/actions/workflows/build-wheel.yml).
The "conda" builds are mostly self-contained and usually do not require extra installations, except the
Windows one which may require
[Microsoft ODBC Driver 17](https://learn.microsoft.com/en-us/sql/connect/odbc/download-odbc-driver-for-sql-server?view=sql-server-ver16#version-17).


### Setup using conda packages

> Activated by `use-conda: 'true'`

The setup process uses [setup-miniconda](https://github.com//conda-incubator/setup-miniconda)
action to install conda and some minimal set of the required external packages. Then, performs
a full `cmake` build of the sources from [GDAL](https://github.com/OSGeo/gdal) repository, the commit
is selected by `gdal-version`.
Finally, a wheel is build from the generated result via standard `setup.py bdist_wheel` procedure.

The `extra-packages` input parameter can be used to install additional external packages, before the `cmake`
build. This allows adding support for more geospatial data formats.

### Setup using GISInternals/system packages

> Activated by `use-conda: 'false'`

Preparation differs for Windows vs. Linux and MacOS runner OS:

- Windows: the prebuild development kit from https://www.gisinternals.com/ is used.
  With the help of this SDK, just `pip wheel gdal` is enough to create wheel including the `.pyd` extensions.
  The rest of GDAL related DLLs are added later.

  This process is much faster, than the full-build approach, but lacks GDAL version / runner OS
  flexibility.

- Linux / MacOS: The required external packages are installed via `apt-get` or `brew`, then continue
  just like "Setup using conda packages".

  The `extra-packages` input can be used, to install additional `apt-get` or `brew` packages, similar to
  the conda build.

### Wheel update (repack)

After the wheel is created, by `pip wheel gdal` or `setup.py bdist_wheel`, it still does not include the
dependent GDAL `.dll`-s, `.so`-s or `.dylib`-s. This is fixed by repacking the wheel, by adding these
dependencies. This step also adds (for convenience only):

- [GDAL programs](https://gdal.org/programs/) under `osgeo_utils/apps`
- The extra GDAL plugins under `osgeo_utils/plugins` (env-var `GDAL_DRIVER_PATH`)
- The [proj-data](https://proj.org/) package under `osgeo_utils/proj` (env-var `PROJ_DATA`)


### Example: GDAL inlined in workflow file

Code from [Reproject a Geometry](https://pcjericks.github.io/py-gdalogr-cookbook/projection.html#reproject-a-geometry)

```yaml
steps:
- uses: actions/setup-gdal@main
- shell: python
  run: |
    from osgeo import ogr
    from osgeo import osr

    source = osr.SpatialReference()
    source.ImportFromEPSG(2927)

    target = osr.SpatialReference()
    target.ImportFromEPSG(4326)

    transform = osr.CoordinateTransformation(source, target)

    point = ogr.CreateGeometryFromWkt("POINT (1120351.57 741921.42)")
    point.Transform(transform)

    print(point.ExportToWkt())
```
