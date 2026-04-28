# Earth Data Hub (EDH) — environment setup & first ERA5 download

This guide gets you from "I have a destine.eu account" to "I can stream ERA5 zarr from EDH and save subsets locally" in one sitting. The companion notebook `download_ERA5_EDH_t2m_Jan2026_test.ipynb` is the worked example.

## 0. Prerequisites

- A **destine.eu account** — sign up at <https://platform.destine.eu/> if you don't have one.
- Conda available on the machine (we use the user-local conda at `~/.conda/envs/`).
- ~3 GB of free space for the new env. If you're tight on home-quota inodes, install the env onto scratch (see step 2).

## 1. Create a dedicated conda environment

The yml below isolates EDH access from your existing CDS / cdsapi env so the two stay clean.

`destineE.yml`:
```yaml
name: destineE
channels:
  - conda-forge
  - defaults
dependencies:
  - python=3.11
  - ipykernel       # so Jupyter can use this env as a kernel
  - xarray
  - zarr            # zarr v3 is required for many EDH datasets
  - dask
  - fsspec
  - aiohttp
  - requests
  - netcdf4         # for writing local .nc subsets
  - h5netcdf
  - pip
  - pip:
      - destinelab  # destine.eu auth helper (pip-only)
```

Create it:
```bash
conda env create --file destineE.yml
```

If your home filesystem is short on inodes, install onto scratch instead:
```bash
mkdir -p /fs/scratch/$YOUR_PROJECT/$USER/conda_envs
conda env create --file destineE.yml \
    --prefix /fs/scratch/$YOUR_PROJECT/$USER/conda_envs/destineE
```
(Replace `$YOUR_PROJECT/$USER` with your project / username.)

## 2. Register the env as a Jupyter kernel

Activate the env (use whichever form created it) and register:

```bash
conda activate destineE
# OR if you used --prefix:
# conda activate /fs/scratch/$YOUR_PROJECT/$USER/conda_envs/destineE

python -m ipykernel install --user --name=destineE
```

The kernel will show up in JupyterLab as **destineE**.

## 3. Generate a Personal Access Token (PAT)

EDH uses its own PAT — separate from your destine.eu password and from any DESP / destinelab JWT.

1. Visit <https://earthdatahub.destine.eu/account-settings#my-personal-access-tokens>
2. Log in.
3. Click "Create new token", give it any name, copy the token **immediately** — it is shown only once.

The token looks like:
```
edh_pat_<long hex string>
```

## 4. Save the PAT in `~/.netrc`

xarray + zarr can pick up the PAT automatically from `~/.netrc` when `trust_env=True` is set on the HTTP client.

```bash
touch ~/.netrc
chmod 600 ~/.netrc
```

Open it with `nano ~/.netrc` (or any text editor) and **append** these two lines on their own lines:
```
machine data.earthdatahub.destine.eu
    password edh_pat_<paste your token here>
```

> **Important:** never paste your PAT into chat or email transcripts. If you ever expose it, revoke and regenerate from the EDH account-settings page immediately.

Sanity-check the file (without echoing the secret):
```bash
ls -la ~/.netrc                                    # must be -rw-------
grep -c '^machine data.earthdatahub.destine.eu$' ~/.netrc   # should print 1
```

## 5. Verify the wiring with the public test dataset

Before fetching real data, prove the env works against EDH's public test zarr (no auth needed):

```python
import xarray as xr

test = xr.open_dataset(
    "https://data.earthdatahub.destine.eu/public/test-dataset-v0.zarr",
    chunks={},
    engine="zarr",
)
print(test)
```

If this prints a Dataset summary, the env is wired up correctly. If it errors, the problem is the env / network / packages — not auth or PAT.

## 6. Open an authenticated ERA5 dataset

Now use a real EDH dataset. Example: ERA5 single-levels daily averaged.

```python
import xarray as xr

URL = "https://data.earthdatahub.destine.eu/era5/era5-single-levels-atmosphere-daily-utc-v0.zarr"

ds = xr.open_dataset(
    URL,
    storage_options={"client_kwargs": {"trust_env": True}},   # tells aiohttp to read ~/.netrc
    chunks={},
    engine="zarr",
    zarr_format=3,                                            # this dataset is published as zarr v3
)
print(ds.data_vars)
print("time coverage:", str(ds.valid_time.values[0])[:10], "->", str(ds.valid_time.values[-1])[:10])
```

If you get a `401 Unauthorized`: the PAT is missing, mistyped, or the `~/.netrc` block isn't being read. Re-check step 4.

## 7. Download a subset

Subset → load to memory → write to a local netCDF. Same pattern your CDS scripts use.

```python
t2m_jan = ds.t2m.sel(valid_time=slice("2026-01-01", "2026-01-31")).load()
t2m_jan.to_netcdf("/path/to/output/era5_t2m_dailymean_2026_p025_Jan_EDH.nc")
```

EDH delivers the dataset's **native resolution only** (here 0.25°). To produce coarser versions for downstream pipelines, subsample after download:

```python
t2m_p05 = t2m_jan.isel(latitude=slice(None, None, 2), longitude=slice(None, None, 2))   # 0.5°
t2m_p15 = t2m_jan.isel(latitude=slice(None, None, 6), longitude=slice(None, None, 6))   # 1.5°
```

These match the CDS `grid:[0.5, 0.5]` / `grid:[1.5, 1.5]` outputs gridpoint-for-gridpoint (CDS uses point-sampling, not block averaging).

## 8. Worked example

See `download_ERA5_EDH_t2m_Jan2026_test.ipynb` in this directory — it walks through steps 5 → 7 end-to-end and writes T2M for Jan 2026 at 0.25° / 0.5° / 1.5° in one pass.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `No module named ipykernel` after `python -m ipykernel install` | `ipykernel` not installed in the env | `conda install -n destineE ipykernel` then re-run |
| `401 Unauthorized` from EDH | PAT missing / wrong / `~/.netrc` not readable | Step 4. Confirm `grep -c` returns 1 and perms are `0600` |
| `KeyError: 'init_time'` or similar coord names | dataset uses different dim names | Print `ds.dims` and `ds.data_vars` and adapt the slice |
| `ZarrFormat error` | dataset is zarr v3, your zarr is v2 | `pip install -U zarr` (need ≥ 3.x) or set `zarr_format=3` |
| Disk-quota error during env create | home filesystem at inode limit | Install to scratch via `--prefix` (see step 1), or `conda clean -a -y` first |
| `Disk quota exceeded` mid-install | file count quota, not byte quota | Same — `quota -s` will show high `files` |

## References

- EDH catalog: <https://earthdatahub.destine.eu/collections/era5/>
- EDH Getting Started: <https://earthdatahub.destine.eu/getting-started>
- destinelab (PyPI): <https://pypi.org/project/destinelab/>
- destine.eu portal: <https://platform.destine.eu/>
