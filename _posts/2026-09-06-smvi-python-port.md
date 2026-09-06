---
title: "Porting SMVI to Python: xarray, dask, and apply_ufunc for flash drought detection"
date: 2026-09-06
permalink: /posts/smvi-python-port/
tags:
  - flash-drought
  - python
  - xarray
  - dask
  - gldas
  - climate-indices
  - soil-moisture
categories:
  - research-science
excerpt: "How I ported the Soil Moisture Volatility Index from R to Python using xarray and dask, keeping the methodology intact across both equations and code."
---

Currently my research focuses on applied AI tools to understand Flash droughts, the Soil Moisture Volatility Index (SMVI) is one of the multiple definitions for Flash droughts, though the original SMVI implementation from [Osman et al. (2021)](https://doi.org/10.1175/JHM-D-20-0140.1) is in R. My entire pipeline runs on Python/xarray over a GLDAS Zarr store. Translating the algorithm was not the hard part. The hard part was making it run on a spatial grid of roughly 600 × 1440 pixels per year without either blowing up memory or waiting an unreasonable amount of time. This post walks through the methodology and its implementation together, equation first, then the code that maps directly to it. This implementation can be found at my [GitHub Flash drought toolkit > SMVI](https://github.com/roberto-chang-silva/flash-drought-toolkit/tree/main/definitions/smvi-gldas) repo.

# What the index is detecting

A day $t$ is a flash drought candidate if soil moisture is simultaneously declining faster than its recent baseline and drier than it historically is for that calendar day:

$$
\text{SMA5}(t) < \text{SMA20}(t) \quad \text{AND} \quad \text{SMA5}(t) < P_{20}(t)
$$

Where $\text{SMA5}$ and $\text{SMA20}$ are 5-day and 20-day trailing means of the standardized root-zone soil moisture anomaly, and $P_{20}(t)$ is the 20th percentile of that anomaly for the same calendar day across the full historical record. Both conditions must hold simultaneously. A slow drift into drought does not qualify. A rapid decline that stays above the historical 20th percentile does not qualify either.

# Standardization

Raw root-zone soil moisture follows a strong seasonal cycle. Comparing an August value to a February value directly would conflate seasonality with drought signal. Before anything else, each daily value is standardized by day-of-year:

$$
Z(t, d) = \frac{\text{RZSM}(t, d) - \mu(d)}{\sigma(d)}
$$

Where $\mu(d)$ and $\sigma(d)$ are the climatological mean and standard deviation of RZSM for day-of-year $d$, computed from the full record. A value of $Z = -1$ means one standard deviation drier than normal for that calendar day, regardless of season. All subsequent steps operate on $Z(t)$ rather than raw RZSM.

In code, this is a grouped subtraction and division over the `dayofyear` coordinate:

```python
clim_mean = rzsm_all.groupby("time.dayofyear").mean(skipna=True)
clim_std  = rzsm_all.groupby("time.dayofyear").std(skipna=True)

rzsm_yr = (rzsm_yr - clim_mean.sel(dayofyear=doy)) / clim_std.sel(dayofyear=doy)
```

The climatology is computed once from the full multi-year record before the year loop and reused for each year. `doy` is the array of day-of-year values for the year being processed, used to reindex the climatology back onto the year's time axis.

# Percentile climatology

For each day-of-year $d$, the 20th percentile threshold is:

$$
P_{20}(d) = Q_{0.20}\big(\{Z(y, d) \mid y \in \text{all years}\}\big)
$$

Since standardization is a per-day-of-year linear transformation, this is equivalent to computing the percentile in raw space and then standardizing:

$$
P_{20}(d) = \frac{Q_{0.20}\big(\{\text{RZSM}(y, d) \mid y \in \text{all years}\}\big) - \mu(d)}{\sigma(d)}
$$

The second form is what the code uses, because it requires only one `groupby.quantile` call over the raw data rather than standardizing the entire record first:

```python
pctl20_raw = rzsm_all.groupby("time.dayofyear").quantile(0.20).drop_vars("quantile")
pctl20 = (pctl20_raw - clim_mean) / clim_std
```

The three climatology arrays (`clim_mean`, `clim_std`, `pctl20`) are materialized together in a single `dask.compute()` call before the year loop. Inside the loop, `pctl20.sel(dayofyear=doy)` reindexes the threshold onto each year's time axis before it enters the detection function.

# Candidate mask and running means

The running means follow the standard trailing formulation:

$$
\text{SMA}_n(t) = \frac{1}{n} \sum_{i=t-n+1}^{t} Z(i)
$$

Right-aligned, so the first $n-1$ values are NaN by construction. The candidate mask is then:

$$
M(t) = \big(\text{SMA5}(t) < \text{SMA20}(t)\big) \land \big(\text{SMA5}(t) < P_{20}(t)\big)
$$

Inside the batch function, both rolling means are computed in one vectorized call across all pixels in a chunk by reshaping to a pandas DataFrame:

```python
pdf  = pd.DataFrame(s2)   # s2 is (time, n_pixels)
cvar = pdf.rolling(window=cfg.fd_v,  min_periods=cfg.fd_v).mean().values   # SMA5
thd  = pdf.rolling(window=cfg.fd_th, min_periods=cfg.fd_th).mean().values  # SMA20
mask = (cvar < thd) & (cvar < p2)
```

`p2` here is the percentile array broadcast to the same `(time, n_pixels)` shape. The mask is fully vectorized across pixels. Everything after this operates per-pixel on the run structure of `mask`.

# Event detection and physical filters

## Finding and filtering runs

Contiguous runs of `True` in $M(t)$ are candidate events. A run from index $s$ to $e$ survives the minimum duration filter only if:

$$
e - s \geq L_{\min} = 4 \text{ pentads} = 20 \text{ days}
$$

`_find_runs` uses a padded boolean diff to locate run boundaries without looping over time steps:

```python
def _find_runs(mask):
    padded = np.concatenate([[False], mask, [False]])
    starts = np.where(~padded[:-1] &  padded[1:])[0]
    ends   = np.where( padded[:-1] & ~padded[1:])[0]
    return starts, ends
```

## Bowen ratio filter

For each surviving run, the 10%-trimmed mean of the Bowen ratio over the event window must fall within:

$$
0.2 \leq \overline{\text{BR}} \leq 7.0 \quad \text{where} \quad \overline{\text{BR}} = \text{mean}_{0.1}\!\left(\frac{Q_h}{Q_{le}}\right)_{[f_i:l_i]}
$$

The lower bound excludes very wet conditions where no meaningful moisture decline is occurring. The upper bound excludes hyperarid surfaces where the soil was already effectively dry before the event. Trimming the outer 10% on each side makes the filter robust to flux outliers in the GLDAS output.

```python
def _trimmed_mean(arr, trim=0.1):
    s = np.sort(arr[~np.isnan(arr)])
    k = int(len(s) * trim)
    return np.nanmean(s[k : len(s) - k])

br_mean = _trimmed_mean(b2[fst : lst + 1, pix])
if np.isnan(br_mean) or br_mean > cfg.br_max or br_mean < cfg.br_min:
    continue
```

## Surface temperature filter

The trimmed mean surface temperature over the event window must exceed 273 K:

$$
\overline{T_s} = \text{mean}_{0.1}\big(\text{AvgSurfT}\big)_{[f_i:l_i]} \geq 273\;\text{K}
$$

Flash drought on frozen or snow-covered ground is a detection artifact. The same `_trimmed_mean` function handles this:

```python
ts_mean = _trimmed_mean(t2[fst : lst + 1, pix])
if np.isnan(ts_mean) or ts_mean < cfg.ts_thd:
    continue
```

# Severity

For events that pass all filters, severity is the cumulative deficit below $P_{20}$ over the event window:

$$
S_i = \sum_{t=f_i}^{l_i} \max\big(0,\; P_{20}(t) - \text{SMA5}(t)\big)
$$

Larger values mean a deeper and longer deficit. The index is unitless and in standardized anomaly space, so it is comparable across regions and seasons.

```python
deficit = p2[fst : lst + 1, pix] - cvar[fst : lst + 1, pix]
sev = float(np.sum(np.maximum(0, deficit)))
```

# Gap filling and ranking

Events separated by 15 days or fewer are merged to prevent a single drought from being split by a brief wet spell:

$$
f_{i+1} - l_i \leq 15 \implies \text{merge into } (f_i,\, l_{i+1})
$$

`_gap_fill` loops until no merges occur in a pass, which handles chains where merging two events brings a third within gap distance:

```python
def _gap_fill(events, max_gap):
    merged = list(events)
    while True:
        new, changed = [], False
        i = 0
        while i < len(merged):
            if i + 1 < len(merged) and (merged[i+1][0] - merged[i][1]) <= max_gap:
                new.append((merged[i][0], merged[i+1][1]))
                i += 2; changed = True
            else:
                new.append(merged[i]); i += 1
        merged = new
        if not changed:
            break
    return merged
```

After merging, events are sorted by duration descending and only the top `num_events` (default 6) are kept per pixel per year. Ranking happens after the merge, not before, which matters near the cap: two short events that merge into a longer one should compete for slots as a single event, not as two.

# Running it

```bash
python smvi.py \
  --year-begin 2003 --year-end 2026 \
  --zarr-path /data/GLDAS_CLSM025_DA1_D_RAW.zarr \
  --output-dir /data/SMVI_out \
  --chunk-lat 150 --chunk-lon 360 --chunk-time 50
```

> ```
> SMVI • 2003–2026
> FDL = 20, FDgap = 15
> Anomalies = True
> Season: 01-01 to 12-31
> Chunks: time=50, lat=150, lon=360
>
> Computing climatology (mean, std) by dayofyear …
> [########################################] | 100% Completed | 43.2s
>   clim_mean  : 366 days
>   clim_std   : 366 days
>   pctl20     : 366 days (anomaly space)
>
> === 2003 ===
>   365 timesteps
>   Detecting flash droughts …
> [########################################] | 100% Completed | 1m 12s
>   Written → /data/SMVI_out/SMVI_all.zarr (year 2003)
>   Written → /data/SMVI_out/SMVI_ts.zarr (year 2003)
> ```

# Notes

- The `apply_ufunc` call uses `dask="parallelized"` with `time` as the core dimension, so dask distributes work over `lat`/`lon` chunks. `allow_rechunk=True` is needed because `pctl20` is indexed by `dayofyear` and gets rechunked internally when broadcast onto the year's time axis.
- The script writes two Zarr stores: `SMVI_all.zarr` with summary variables indexed by `year`, and `SMVI_ts.zarr` with the full daily event mask and per-day severity appended along `time`. Writing per year and appending keeps peak memory flat regardless of record length.
- A `dayofyear` coordinate inherited from `groupby` operations shows up as a spurious variable in the output without `ds_out.drop_vars(["dayofyear"], errors="ignore")`. The `errors="ignore"` handles years where it does not appear so the loop does not abort.
- The gap-filling threshold of 15 days comes from the original paper and has not been sensitivity-tested on the target domain. Regions with strong monsoon interruptions are the most likely place where it matters.

## References

Osman, M., Zaitchik, B., Badr, H., Hameed, S., & Ghatak, M. (2021). Flash Drought Onset and Development Mechanisms in the United States. *Journal of Hydrometeorology*, 22(5), 1187-1202.
