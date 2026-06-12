# FD-HD truth-labelling production — running interactively (no justIN)

How to run the FD-HD (`dune10kt 1x2x6`) truth-labelling production chain **by hand** on an
interactive node, with the locally-built `larwirecell`. This is the same physics as the justIN
workflow (see [`fdhd_sparse_v10_20_localbuild.md`](fdhd_sparse_v10_20_localbuild.md)) but
without any of the grid/bundle machinery.

## What it does
```
GEN (GENIE)  →  G4  →  WireCell Sim + Signal Processing  →  2D Truth Labelling  →  (optional) dense→sparse HDF5
```

## Requirements

1. **CVMFS access**: `/cvmfs/dune.opensciencegrid.org`, `/cvmfs/larsoft.opensciencegrid.org`,
   and `/cvmfs/dune.osgstorage.org` (the last hosts the beam **flux files** and the **CVN
   protobuf** the labelling uses).
2. **SL7 environment.** `dunesw v10_20_08d00` is built for SL7. On an AlmaLinux/EL9 host, run
   inside the SL7 apptainer container first:
   ```bash
   /cvmfs/oasis.opensciencegrid.org/mis/apptainer/current/bin/apptainer shell --shell=/bin/bash \
     -B /cvmfs,/exp,/nashome,/pnfs/dune,/opt,/run/user,/etc/hostname,/etc/hosts,/etc/krb5.conf \
     --ipc --pid /cvmfs/singularity.opensciencegrid.org/fermilab/fnal-dev-sl7:latest
   ```
3. **Base release**: `dunesw v10_20_08d00 -q e26:prof` (from CVMFS).
4. **Custom `larwirecell` build** — UPS `v10_03_05`, branch **`xn/trackid_pid_map`** of
   `Ningclover/larwirecell` (adds the `TrackIDPIDMap2h5` module + the Labelling2D
   energy-fraction / total-num-electrons outputs). Either reuse the existing MRB build at
   `/exp/dune/data/users/xning/larsoft/v10_20_08d00/`, or build your own (see that area's
   `README.md`: `mrb newDev` → clone the branch → `mrbsetenv` → `mrb i -j8`).
5. **FD-HD configs** (fcls + WireCell jsonnet + torch model) — in
   `/exp/dune/data/users/xning/larsoft/cffm-if/dune10kt-1x2x6/`: `gen_genie.fcl`, `g4.fcl`,
   `wcls_sim_sp.fcl`, `wcls-labelling2d_sep.fcl`, the `*.jsonnet`, and
   `ts-model/CP49_mobilenetv3.ts`.
6. **For the sparse step only**: a Python 3.9 env with `numpy`, `h5py`, `scipy`, plus the
   `convert-to-sparse.py` script. (dunesw's python does **not** ship h5py — `pip install` into
   a venv, or `pip install --user`.)

## Setup (each session)
```bash
source /cvmfs/dune.opensciencegrid.org/products/dune/setup_dune.sh
setup dunesw v10_20_08d00 -q e26:prof
source /exp/dune/data/users/xning/larsoft/v10_20_08d00/localProducts_larsoft_v10_20_08_e26_prof/setup
mrbslp                              # activates local larwirecell v10_03_05 (overrides official v10_03_04)
ups active | grep larwirecell       # sanity: v10_03_05 from the localProducts path
```
> Interactively, `mrbslp` is the right way to pick up the local build. (The grid workflow
> avoided it only because the localProducts `setup` script isn't path-relocatable — irrelevant
> when running in place.)

## Run the chain
Work from a directory that contains the configs (simplest: copy `cffm-if/dune10kt-1x2x6/`
somewhere writable and `cd` into it, so the fcls, the `*.jsonnet`, and `ts-model/` are all
found):
```bash
NEV=10
lar -n $NEV -c gen_genie.fcl           -o gen.root     # GENIE beam (numu-dominated FHC)
lar         -c g4.fcl         -s gen.root -o g4.root    # Geant4
lar         -c wcls_sim_sp.fcl -s g4.root -o sp.root    # WireCell detsim + signal processing
lar         -c wcls-labelling2d_sep.fcl -s sp.root      # 2D truth labelling -> writes the HDF5
```
The labelling fcl has **`keep_truth: true`** — that is the "with truth" switch. (Setting it
`false` would make the pixeldata reco-only.)

## Optional: dense → sparse packaging
```bash
python convert-to-sparse.py \
  --input-dir  <dir with the labelling .h5> \
  --output-dir sparse \
  --pattern "*pixeldata-anode*.h5" \
  --copy-metadata "*metadata.h5" \
  --compression gzip --compression-level 2
```
`trackid_pid_map.h5` is a truth map (not pixel-image data) — copy it as-is, don't sparsify.

## Final dataset — files

| File | Per | Contents |
|---|---|---|
| `pixeldata-anode0.h5` … `pixeldata-anode11.h5` | 12 anodes (1×2×6 = 12 APAs) | 2D image stacks (reco + truth) |
| `metadata.h5` | per job/event set | event/bookkeeping metadata (`Truth2h5`) |
| `trackid_pid_map.h5` | per job/event set | full MC-truth particle maps (`TrackIDPIDMap2h5`) |

## File structure / format (HDF5)

**`pixeldata-anode*.h5`** — one top-level group per frame (e.g. `"1"`). With truth on, six
image planes, each stored as three datasets:
- `frame_rebinned_reco` — reco charge image, shape **(2560, 1500)** float32
  (2560 channels × 1500 time ticks; ticks rebinned ×4 from 6000)
- `frame_trackid_1st`, `frame_trackid_2nd` — leading / sub-leading truth track-ID per pixel
- `frame_energyfrac_1st`, `frame_energyfrac_2nd` — energy fraction of those contributors
- `frame_total_numelectrons` — total ionization electrons per pixel
- plus `channels_<name>` (shape (2560,)) and `tickinfo_<name>` (shape (3,)) for each plane

**`trackid_pid_map.h5`** — group per frame with two subgroups:
- `mcpart/` — all G4 tracks: `track_ids`, `pids` (PDG), `mother_ids`, `mother_pids`,
  `processes`, `start_xyzts`/`end_xyzts` (N×4, cm & ns), `start_moms`/`end_moms` (N×4, GeV),
  and extended fields `end_processes`, `statuses`, `masses`, `ndaughters`, `ntrajpts`
- `simchnl/` — only TPC-ionizing tracks (a subset): `track_ids`, `pids`, `mother_ids`,
  `mother_pids`, `processes`, `energies` (deposited, MeV)

**After sparse conversion**, each `frame_*` becomes a subgroup with `coords` (N×2 int32 nonzero
pixel indices) + `features` (N,), while `channels_*`/`tickinfo_*` are copied unchanged and
`metadata.h5` / `trackid_pid_map.h5` are copied as-is.

## Quick inspection
```bash
# in a python env with h5py:
python - <<'PY'
import h5py
for f in ["pixeldata-anode0.h5","trackid_pid_map.h5"]:
    print("==",f); 
    with h5py.File(f) as h: h.visititems(lambda n,o: print("  ",n, getattr(o,"shape","")))
PY
```
