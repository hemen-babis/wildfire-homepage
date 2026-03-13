---
title: "WRF-SFIRE End-to-End Workflow"
description: "Complete end-to-end workflow for running WRF-SFIRE simulations manually via WPS, real.exe, and wrf.exe on ORCA."
url: "/wrf-sfire-workflow/"
type: "wrf-sfire-workflow"
---

Complete workflow for generating a WRF-SFIRE wildfire simulation run over a 10 km × 10 km domain: dx=dy=1000 m, 3-day forecast window, WRF history output every 30 minutes (145 timestamps), single SFIRE ignition centered in the domain.

> **Note:** Replace placeholder values like `<TAG>`, `<PATH>`, `<DATE>`, `<group>`, `<user>` with your actual run values. All commands assume you are inside the correct run directory for that stage.

## 0. Quick Glossary

**WPS outputs**

| Executable | Produces | Description |
|---|---|---|
| `geogrid.exe` | `geo_em.d0*.nc` | Static domain: terrain, landuse, etc. |
| `ungrib.exe` | `FILE:*` | Decoded meteorology from GRIB → WPS intermediate |
| `metgrid.exe` | `met_em.d0*.<YYYY-MM-DD_HH:MM:SS>.nc` | Forcing interpolated to your domain |

**WRF/WRF-SFIRE outputs**

| Executable | Produces | Description |
|---|---|---|
| `real.exe` | `wrfinput_d0*`, `wrfbdy_d01` | Initial and boundary conditions |
| `wrf.exe` | `wrfout_d0*_*` | Model outputs |

---

## 1. Environment Setup (ORCA runtime)

```bash
module purge
module load gcc
module load openmpi
```

**1.2 — NetCDF and compression libraries (Jasper / PNG / Zlib)**

```bash
export LD_LIBRARY_PATH=$HOME/libs/netcdf/lib:$HOME/libs/jasper/lib64:$LD_LIBRARY_PATH
export PATH=$HOME/libs/netcdf/bin:$PATH   # makes ncdump available
```

**1.3 — Define code locations (WPS and WRF)**

```bash
export WPS_DIR=$HOME/wrf_sfire/WPS_clean/WPS-4.6.0
export WRF_DIR=$HOME/wrf_sfire/WRF
```

---

## 2. Example Directory Layout

```
$HOME/wrf_sfire/
  WPS_clean/WPS-4.6.0/     # built WPS
  WRF/                     # built WRF-SFIRE
  libs/netcdf/             # NetCDF libraries
  libs/jasper/             # Jasper (+ png/zlib) libraries

/scratch/<group>/<user>/
  WPS_RUN_<TAG>/           # WPS run dir  (geo_em, FILE:, met_em)
  WRF_RUN_<TAG>/           # WRF run dir  (wrfinput, wrfbdy, wrfout)
```

---

## 3. WPS Preprocessing

### 3.0 Create and enter your WPS run directory

```bash
mkdir -p /scratch/<group>/<user>/WPS_RUN_<TAG>
cd /scratch/<group>/<user>/WPS_RUN_<TAG>

# Copy your namelist
cp -p <PATH_TO_YOUR_NAMELIST>/namelist.wps .

# Symlink executables
ln -sf $WPS_DIR/geogrid/src/geogrid.exe .
ln -sf $WPS_DIR/ungrib/src/ungrib.exe .
ln -sf $WPS_DIR/metgrid/src/metgrid.exe .
ln -sf $WPS_DIR/link_grib.csh .

# Verify
ls -l geogrid.exe ungrib.exe metgrid.exe link_grib.csh namelist.wps
```

### 3.1 geogrid.exe

**What it does:** Reads `namelist.wps` and the GEOG dataset and creates `geo_em.d01.nc` (terrain height, landuse, soil type, etc.).

**Verify key namelist settings:**

```bash
grep -nE "max_dom|dx|dy|e_we|e_sn|map_proj|ref_lat|ref_lon|truelat1|truelat2|stand_lon|geog_data_path|start_date|end_date" namelist.wps
```

**Run:**

```bash
rm -f geo_em.d0*.nc geogrid.log
./geogrid.exe |& tee geogrid.log
```

**Validate:**

```bash
ls -lh geo_em.d0*.nc
tail -n 60 geogrid.log
```

Expected: log ends with a success message and `geo_em.d0*.nc` files are present.

**Common failures:**

| Symptom | Fix |
|---|---|
| Wrong `geog_data_path` | Fix the path in `namelist.wps` |
| GEOG dataset missing/corrupt | Confirm directory exists with expected subfolders |
| Projection mismatch | Ensure `map_proj` and lat/lon match your intended domain |

### 3.2 ungrib.exe

**What it does:** Converts GRIB meteorology (e.g., GFS) into WPS intermediate files (`FILE:YYYY-MM-DD_HH`), later consumed by metgrid.

**Choose and link the correct Vtable:**

```bash
# For GFS:
ln -sf $WPS_DIR/ungrib/Variable_Tables/Vtable.GFS Vtable
```

**Link GRIB files:**

```bash
# Adjust pattern to your GRIB naming
./link_grib.csh /path/to/grib/files/gfs*.f*

# Check GRIBFILE links
ls -1 GRIBFILE.* | head -n 20
```

**Run:**

```bash
./ungrib.exe |& tee ungrib.log
```

**Validate:**

```bash
ls -1 FILE:* | head -n 20
tail -n 80 ungrib.log
```

**Common failures:**

**A) Missing runtime libraries (Jasper/PNG)**
```
error while loading shared libraries: libjasper.so...
```
Fix: `export LD_LIBRARY_PATH=$HOME/libs/jasper/lib64:$LD_LIBRARY_PATH`

**B) w3 / g2 link errors during WPS build**

If ungrib fails to link w3/g2, create symlinks so ungrib finds the libraries:
```bash
cd $WPS_DIR/ungrib/src/ngl
ln -sf w3/libw3.a   libw3.a
ln -sf g2/libg2_4.a libg2_4.a
```
Then rebuild WPS.

**C) Wrong Vtable for your GRIB source** — ungrib runs but produces incomplete or empty outputs. Pick the correct `Vtable.*` for your dataset (GFS vs ERA5 vs NAM, etc.).

### 3.3 metgrid.exe

**What it does:** Combines `geo_em.d0*.nc` (from geogrid) + `FILE:*` (from ungrib) → `met_em.d0*.<timestamp>.nc`, which `real.exe` consumes.

**Ensure METGRID.TBL is present:**

```bash
ln -sf $WPS_DIR/metgrid/METGRID.TBL METGRID.TBL
ls -l METGRID.TBL
```

**Run:**

```bash
rm -f met_em.d0*.nc metgrid.log
./metgrid.exe |& tee metgrid.log
```

**Validate:**

```bash
ls -lh met_em.d0*.nc | head
tail -n 80 metgrid.log
```

You should see `met_em.d01.<YYYY-MM-DD_HH:MM:SS>.nc` files spanning your forecast window.

### 3.4 WPS Completion Checklist

Before moving on to WRF, confirm all of these exist:

```bash
ls -1 geo_em.d0*.nc          # geogrid outputs
ls -1 FILE:* | head           # ungrib outputs
ls -1 met_em.d0*.nc | head   # metgrid outputs
ls -1 geogrid.log ungrib.log metgrid.log  # logs
```

Do not proceed to WRF until all files are present.

---

## 4. WRF / WRF-SFIRE Setup

### 4.1 Create and enter the WRF run directory

```bash
mkdir -p /scratch/<group>/<user>/WRF_RUN_<TAG>
cd /scratch/<group>/<user>/WRF_RUN_<TAG>
```

Expected structure inside `WRF_RUN_<TAG>/`:

```
namelist.input
met_em.d01.*.nc        (symlinked from WPS run)
real.exe               (symlink)
wrf.exe                (symlink)
logs/
rsl.out.0000, rsl.error.0000, ...
wrfinput_d01
wrfbdy_d01
wrfout_d01_YYYY-MM-DD_HH:MM:SS
```

### 4.2 Load modules and set exports

```bash
module purge
module load gcc
module load openmpi

export LD_LIBRARY_PATH=$HOME/libs/netcdf/lib:$HOME/libs/jasper/lib64:$LD_LIBRARY_PATH
export PATH=$HOME/libs/netcdf/bin:$PATH
export WRF_DIR=$HOME/wrf_sfire/WRF
```

### 4.3 Link WPS met_em files

```bash
ln -sf /scratch/<group>/<user>/WPS_RUN_<TAG>/met_em.d0*.nc .

# Confirm
ls -1 met_em.d01.*.nc | head
ls -1 met_em.d01.*.nc | tail
```

### 4.4 Link WRF executables

```bash
ln -sf $WRF_DIR/main/real.exe .
ln -sf $WRF_DIR/main/wrf.exe .
```

### 4.5 Copy namelist and create logs directory

```bash
cp -p <PATH_TO_YOUR_NAMELIST>/namelist.input .
mkdir -p logs
```

---

## 5. real.exe

**What it does:** Converts WPS forcing files (`met_em*`) into:
- `wrfinput_d01` — initial conditions
- `wrfbdy_d01` — boundary conditions

If `real.exe` fails, the answer is almost always in `rsl.error.0000`.

**Clean old outputs:**

```bash
rm -f wrfinput_d0* wrfbdy_d0* rsl.out.* rsl.error.*
rm -f logs/real.log
```

**Run:**

```bash
./real.exe |& tee logs/real.log
```

**Validate:**

```bash
ls -lh wrfinput_d01 wrfbdy_d01
tail -n 80 logs/real.log
```

**If it fails:**

```bash
ls -1 rsl.error.* | head
tail -n 120 rsl.error.0000
```

**Confirm your run matches requirements:**

```bash
grep -nE "run_(days|hours|minutes|seconds)|start_(year|month|day|hour)|end_(year|month|day|hour)|history_interval|frames_per_outfile|time_step" namelist.input

grep -nE "max_dom|e_we|e_sn|dx|dy|map_proj|ref_lat|ref_lon|truelat1|truelat2|stand_lon" namelist.input
```

---

## 6. wrf.exe (WRF-SFIRE simulation)

**What it does:** Runs the full atmospheric model with SFIRE enabled. Advances the fire-spread model alongside the atmospheric simulation.

Produces: `wrfout_d01_YYYY-MM-DD_HH:MM:SS` — one file per `history_interval` minutes. For `history_interval = 30` over a 3-day run, expect 145 output files.

**Clean old outputs:**

```bash
rm -f rsl.out.* rsl.error.*
rm -f logs/wrf.log
rm -f wrfout_d0*   # only if OK with deleting prior outputs
```

**Run:**

```bash
./wrf.exe |& tee logs/wrf.log
```

**Monitor progress while running:**

```bash
tail -f logs/wrf.log
ls -lt wrfout_d01_* | head

# Check timestep progress without tailing full log
grep -n "Timing for main" logs/wrf.log | tail
```

**Validate output after completion:**

```bash
# Count output files
ls -1 wrfout_d01_* | wc -l

# Check first and last timestamps
ls -1 wrfout_d01_* | head -n 3
ls -1 wrfout_d01_* | tail -n 3

# Inspect times inside a file (NetCDF)
f=$(ls -1 wrfout_d01_* | head -n 1)
ncdump -v Times "$f" | sed -n '1,35p'
```

**If wrf.exe fails:**

1. Check the first error file:
```bash
tail -n 200 rsl.error.0000
```

2. Check for missing input/boundary:
   - `wrfinput_d01` missing → `real.exe` did not complete
   - `wrfbdy_d01` missing → `real.exe` did not complete

3. Check your forcing set:
```bash
ls -1 met_em.d01.*.nc | head
```

4. Confirm library paths:
```bash
ldd ./wrf.exe | head
```

---

## 7. Slurm Monitoring

**Submit and capture job ID:**

```bash
jid=$(sbatch --parsable <your_job>.slurm)
echo "JOBID=$jid"
```

**Monitor:**

```bash
squeue -j "$jid"
sacct -j "$jid" --format=JobID,JobName,State,ExitCode,Elapsed,Timelimit -n
```

**Tail logs:**

```bash
tail -F logs/wrf.log
```
