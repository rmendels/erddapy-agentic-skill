---
name: erddapy
description: Use when writing Python code with erddapy to search for, retrieve metadata about, or download data from ERDDAP(TM) servers. Covers the ERDDAP class, tabledap and griddap workflows, constraints format, response conversion (to_pandas, to_xarray, to_ncCF, to_iris), multi-server search, and common pitfalls.
metadata:
  author: Roy Mendelssohn (@rmendels)
  version: "1.0"
license: CC0
---

erddapy is a Python client for ERDDAP(TM) servers — it constructs ERDDAP REST URLs and returns data in pandas, xarray, netCDF4, or iris formats. Works with any public ERDDAP server. (For the R equivalent, see the `rerddap`/`rerddapXtracto`/`plotdap`/`rerddapUtils` skills; for raw URL construction in any language, see the `erddap` skill.)

## Core Principle: info URL Before Data

Always check the info URL to get exact variable names, dimension names, and data ranges before building a download request — especially for griddap. Guessing names causes silent wrong results or errors.

```python
from erddapy import ERDDAP

e = ERDDAP(server="https://coastwatch.pfeg.noaa.gov/erddap", protocol="tabledap")

# Inspect dataset metadata before requesting data
info_url = e.get_info_url(dataset_id="erdMH1chla8day", response="csv")
# Fetch and read with pandas to see variables, dimensions, actual_range
import pandas as pd
info_df = pd.read_csv(info_url, skiprows=0)
```

## ERDDAP Class: Constructor and Attributes

```python
e = ERDDAP(
    server="https://coastwatch.pfeg.noaa.gov/erddap",
    protocol="tabledap",   # or "griddap"; can also be set later
    response="html",       # default; overridden per-request
)

# Set before fetching
e.dataset_id = "erdCinpKfmBT"
e.response   = "csv"          # "nc", "csvp", "json", "mat", "parquet", ...
e.variables  = ["longitude", "latitude", "temperature", "time"]
e.constraints = {
    "time>=": "2006-08-24T00:00:00Z",
    "time<=": "2007-08-24T00:00:00Z",
    "latitude>=": 30.0,
    "latitude<=": 40.0,
}
```

Instance attributes written after construction: `dataset_id`, `response`, `variables`, `constraints`, `dim_names` (griddap only), `requests_kwargs`, `auth`.

## Constraint Format

Constraints are dicts where **the key includes the operator** — not a separate argument:

```python
# ✅ Correct
constraints = {
    "time>=": "2016-07-10T00:00:00Z",
    "time<=": "2017-02-10T00:00:00Z",
    "latitude>=": 38.0,
    "latitude<=": 41.0,
    "longitude>=": -72.0,
    "longitude<=": -69.0,
}

# Relative constraints also work
constraints = {
    "time>": "now-7days",
    "depth>": "max(depth)-23",
}

# ❌ Wrong — operator must be part of the key
constraints = {"time": ">= 2016-07-10T00:00:00Z"}  # broken
```

## tabledap Workflow

```python
from erddapy import ERDDAP

e = ERDDAP(server="https://gliders.ioos.us/erddap", protocol="tabledap")

e.dataset_id = "whoi_406-20160902T1700"
e.response   = "csv"
e.variables  = ["depth", "latitude", "longitude", "salinity", "temperature", "time"]
e.constraints = {
    "time>=": "2016-07-10T00:00:00Z",
    "time<=": "2017-02-10T00:00:00Z",
    "latitude>=": 38.0,
    "latitude<=": 41.0,
    "longitude>=": -72.0,
    "longitude<=": -69.0,
}

df = e.to_pandas()       # pandas DataFrame
ds = e.to_xarray()       # xarray Dataset
nc = e.to_ncCF()         # netCDF4.Dataset (CF-compliant)
cl = e.to_iris()         # iris CubeList
```

## griddap Workflow

Setting `dataset_id` on a griddap instance **automatically calls `griddap_initialize()`**, which fetches metadata, populates `constraints`, `dim_names`, and `variables` from the server. Modify `constraints` and `variables` after that:

```python
e = ERDDAP(server="https://coastwatch.pfeg.noaa.gov/erddap", protocol="griddap")

# This triggers griddap_initialize() automatically — fetches metadata
e.dataset_id = "jplMURSST41"

# e.constraints is now pre-filled with full dimension ranges; narrow it down
e.constraints = {
    "time>=": "2020-06-01T00:00:00Z",
    "time<=": "2020-06-07T00:00:00Z",
    "time_step": "1",
    "latitude>=": 30.0,
    "latitude<=": 50.0,
    "latitude_step": "1",
    "longitude>=": -140.0,
    "longitude<=": -110.0,
    "longitude_step": "1",
}

# Limit variables to those you need
e.variables = ["analysed_sst"]

ds = e.to_xarray()

# Or call griddap_initialize() explicitly with a step (subsampling)
e2 = ERDDAP(server="https://coastwatch.pfeg.noaa.gov/erddap", protocol="griddap")
e2.griddap_initialize(dataset_id="jplMURSST41", step=4)
```

**griddap constraint keys use `_step` suffix for stride:** `"time_step"`, `"latitude_step"`, `"longitude_step"`.

## URL Builders

All URL-builder methods return a string — useful for inspecting the request before fetching, or for passing to `pandas.read_csv()` / `requests.get()` directly:

```python
url = e.get_download_url()      # full data download URL
url = e.get_info_url(dataset_id="erdMH1chla8day", response="csv")
url = e.get_search_url(search_for="sea surface temperature", response="csv")
url = e.get_categorize_url(categorize_by="ioos_category", value="Temperature", response="csv")

# Override any instance attribute for one-off URL construction
url = e.get_download_url(
    dataset_id="erdCinpKfmBT",
    protocol="tabledap",
    variables=["longitude", "latitude", "temperature"],
    response="csv",
    constraints={"time>=": "2006-08-24T00:00:00Z"},
    distinct=True,
)
```

## Variable Discovery

```python
# Variables with a specific attribute value
e.get_var_by_attr(dataset_id="whoi_406-20160902T1700", standard_name="sea_water_temperature")
# ['temperature']

# Variables with axis attribute matching X, Y, Z, or T
axis = lambda v: v in ["X", "Y", "Z", "T"]
e.get_var_by_attr(dataset_id="whoi_406-20160902T1700", axis=axis)
# ['latitude', 'longitude', 'time', 'depth']
```

## Dataset and Server Discovery

```python
from erddapy.servers.servers import servers
from erddapy.multiple_server_search import search_servers, advanced_search_servers

# List known public ERDDAP servers (dict of short_name -> Server namedtuple)
known = servers()
for name, srv in list(known.items())[:5]:
    print(name, srv.url)

# Search one server
url = e.get_search_url(search_for="chlorophyll", response="csv", protocol="griddap")
results = pd.read_csv(url)

# Search all known public servers
df = search_servers("sea surface temperature", protocol="tabledap")
df = search_servers("chlorophyll", protocol="griddap", parallel=True)

# Advanced multi-server search (bounding box, time, etc.)
df = advanced_search_servers(
    protocol="tabledap",
    minLat=30, maxLat=50,
    minLon=-140, maxLon=-110,
    minTime="2020-01-01T00:00:00Z",
    maxTime="2020-06-01T00:00:00Z",
)

# Search advanced on one server
url = e.get_search_url(
    search_for="temperature",
    protocol="tabledap",
    response="csv",
    minLat=30, maxLat=50,
    minLon=-140, maxLon=-110,
)
```

## Response Formats

| `response` value | Notes |
|---|---|
| `"csv"` | ISO-8859-1 CSV, row 1 = names, row 2 = units |
| `"csvp"` | Like csv; `to_pandas()` default |
| `"nc"` | NetCDF3; `to_xarray()` default for griddap |
| `"ncCF"` | CF-compliant NetCDF; `to_ncCF()` / tabledap |
| `"json"` | ERDDAP JSON table format |
| `"mat"` | MATLAB |
| `"parquet"` | Requires ERDDAP >= 2.25 |
| `"opendap"` | OPeNDAP endpoint (no download, griddap only) |

## Authentication and Requests Options

```python
e = ERDDAP(server="https://private.example.org/erddap", protocol="tabledap")
e.auth = ("username", "password")           # HTTP Basic Auth
e.requests_kwargs = {"timeout": 120}        # passed to requests/urlopen

df = e.to_pandas(requests_kwargs={"timeout": 60})   # override per-call
ds = e.to_xarray(mask_and_scale=True)               # xarray kwargs passed through
```

## xarray Backend

erddapy registers an xarray backend so you can open ERDDAP netCDF URLs directly:

```python
import xarray as xr

url = e.get_download_url()
ds = xr.open_dataset(url, engine="erddapy")
```

## Common Mistakes

| Mistake | Fix |
|---|---|
| Operator outside the key: `constraints = {"time": ">= 2020-01-01"}` | Key must include operator: `{"time>=": "2020-01-01T00:00:00Z"}` |
| Setting `dataset_id` before `protocol="griddap"` | Set `protocol` in the constructor or before `dataset_id`; setting `dataset_id` auto-triggers `griddap_initialize()` |
| Using griddap `constraints` format for tabledap (with `_step`) | tabledap constraints are pure `variable OP value`; `_step` keys are griddap-only |
| Calling `to_pandas()` on a griddap dataset with `response="nc"` | `to_pandas()` uses csvp; works for tabledap data; for griddap prefer `to_xarray()` |
| Large unconstrained griddap request | After `griddap_initialize()`, narrow `e.constraints` before fetching — full-range requests can be gigabytes |
| Forgetting that `to_xarray()` passes extra kwargs to `xr.open_dataset` | Use keyword arguments: `e.to_xarray(mask_and_scale=True)` |
| Assuming `parquet` format always works | Requires ERDDAP server >= 2.25; erddapy raises an error otherwise |

## Quick Reference

| Task | Method / Function |
|---|---|
| Build any URL without fetching | `get_download_url()`, `get_info_url()`, `get_search_url()`, `get_categorize_url()` |
| Inspect dataset metadata | `get_info_url(dataset_id=..., response="csv")` + `pd.read_csv()` |
| Find variables by attribute | `get_var_by_attr(dataset_id=..., standard_name=...)` |
| Initialize griddap constraints from server | `griddap_initialize(dataset_id=..., step=1)` |
| Fetch as pandas DataFrame | `to_pandas()` |
| Fetch as xarray Dataset | `to_xarray()` |
| Fetch as CF netCDF4 object | `to_ncCF()` |
| Fetch as iris CubeList | `to_iris()` |
| Download to file | `download_file(file_type="nc")` |
| List known public ERDDAP servers | `from erddapy.servers.servers import servers; servers()` |
| Search multiple servers | `from erddapy.multiple_server_search import search_servers` |
| Advanced multi-server search | `from erddapy.multiple_server_search import advanced_search_servers` |
