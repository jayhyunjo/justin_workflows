# FD-HD truth-labelling — interactive run on a non-GPVM machine (portable)

Run the same chain as [`fdhd_sparse_v10_20_interactive.md`](fdhd_sparse_v10_20_interactive.md),
but on **any Linux machine with CVMFS + apptainer** (your institution's cluster, a workstation,
etc.) instead of a FNAL `dunegpvm`.

**Key difference from the GPVM version:** off-site you have no `/exp`, so you cannot use
`mrbslp` (the localProducts `setup` script hard-codes the `/exp/...` build path). Instead ship
the **relocatable bundle** (the same `fdhd_v10_20_localwc_bundle.tar` used for justIN) and set
`larwirecell` up directly via `PRODUCTS` — the product is relocatable, so this works anywhere.

```
GEN (GENIE) -> G4 -> WireCell Sim+SP -> 2D Truth Labelling -> (optional) dense->sparse HDF5
```

## Requirements

1. **CVMFS** with these repos available (system-mounted at `/cvmfs`, or via `cvmfsexec`):
   - `dune.opensciencegrid.org`, `larsoft.opensciencegrid.org` — dunesw + larsoft
   - `singularity.opensciencegrid.org` — the SL7 container image
   - `dune.osgstorage.org` — GENIE beam **flux files** + the **CVN** protobuf
   - `oasis.opensciencegrid.org` — only if you don't have a local apptainer
   Make sure `cvmfs-config-osg` (or equivalent) is installed so the `*.osgstorage.org` /
   `*.opensciencegrid.org` repo keys resolve. **No grid proxy/token is needed** — the flux is
   read straight from CVMFS, and there is no justIN involved.
2. **apptainer** (or singularity) — system install, or the one under `oasis` CVMFS.
3. **The bundle** `fdhd_v10_20_localwc_bundle.tar` (~150 MB) copied to the machine, e.g.
   `scp <you>@dunegpvm0X.fnal.gov:/exp/dune/app/users/jjo/fdhd_v10_20_localwc_bundle.tar .`
   It contains everything not on CVMFS: the relocatable `larwirecell v10_03_05` build
   (`localProducts/`), the FD-HD configs (`cffm-if/`), `convert-to-sparse.py`, and cp39 python
   `wheels/`.
4. **Disk/network**: GENIE streams the flux from `dune.osgstorage.org` over CVMFS — fine, but
   can be slow off-site; expect a few GB of CVMFS cache. (See gotchas for pre-staging flux.)

## Step 0 — CVMFS without root (only if `/cvmfs` isn't already mounted)
If the machine has `/cvmfs` mounted system-wide, skip this. Otherwise use **cvmfsexec**:
```bash
git clone https://github.com/cvmfs/cvmfsexec.git && cd cvmfsexec
./makedist osg                        # fetch the OSG cvmfs config + keys
# Then prefix the container launch in Step 2 with:  ./cvmfsexec dune.opensciencegrid.org \
#   larsoft.opensciencegrid.org singularity.opensciencegrid.org dune.osgstorage.org \
#   oasis.opensciencegrid.org -- <command>
```

## Step 1 — unpack the bundle
```bash
mkdir -p ~/fdhd && tar -xf fdhd_v10_20_localwc_bundle.tar -C ~/fdhd
# ~/fdhd/{localProducts, cffm-if, convert-to-sparse.py, wheels}
mkdir -p ~/fdhd/work        # writable run dir
```

## Step 2 — enter the SL7 container
Bind **only** `/cvmfs` and your local bundle/work dirs (no FNAL filesystems):
```bash
APPTAINER=apptainer        # or /cvmfs/oasis.opensciencegrid.org/mis/apptainer/current/bin/apptainer
"$APPTAINER" shell --shell=/bin/bash \
  -B /cvmfs,$HOME/fdhd \
  --ipc --pid \
  /cvmfs/singularity.opensciencegrid.org/fermilab/fnal-dev-sl7:latest
```
(`$HOME` is auto-mounted by apptainer, so `~/fdhd` is visible; the explicit `-B` is just to be safe.)

## Step 3 — set up dunesw + the local larwirecell (via PRODUCTS, NOT mrbslp)
```bash
source /cvmfs/dune.opensciencegrid.org/products/dune/setup_dune.sh
setup dunesw v10_20_08d00 -q e26:prof

export PRODUCTS="$HOME/fdhd/localProducts:${PRODUCTS}"
setup larwirecell v10_03_05 -q e26:prof
ups active | grep '^larwirecell'      # expect v10_03_05 from $HOME/fdhd/localProducts

# point fcl / wire-cell search paths at the bundled FD-HD configs
CFG="$HOME/fdhd/cffm-if/dune10kt-1x2x6"
export FW_SEARCH_PATH="$CFG:${FW_SEARCH_PATH}"
export FCL_SEARCH_PATH="$CFG:${FCL_SEARCH_PATH}"
export FHICL_FILE_PATH="$CFG:${FHICL_FILE_PATH}"
export WIRECELL_PATH="$CFG:${WIRECELL_PATH}"
```

## Step 4 — run the chain
```bash
cd "$CFG"          # configs (fcls, *.jsonnet, ts-model/) are here; or cd ~/fdhd/work with $CFG on the paths above
NEV=10
lar -n $NEV -c gen_genie.fcl           -o gen.root      # GENIE beam (numu-dominated FHC)
lar         -c g4.fcl         -s gen.root -o g4.root    # Geant4
lar         -c wcls_sim_sp.fcl -s g4.root -o sp.root    # WireCell detsim + signal processing
lar         -c wcls-labelling2d_sep.fcl -s sp.root      # 2D truth labelling -> writes the HDF5
```
`wcls-labelling2d_sep.fcl` has `keep_truth: true` (the "with truth" switch).

## Step 5 — optional dense -> sparse (offline, from the bundled wheels)
```bash
python3 -m venv ~/fdhd/venv && source ~/fdhd/venv/bin/activate
pip install --no-index --find-links ~/fdhd/wheels numpy==1.23.5 h5py==3.7.0 scipy==1.9.3
python ~/fdhd/convert-to-sparse.py \
  --input-dir <dir with the labelling .h5> --output-dir sparse \
  --pattern "*pixeldata-anode*.h5" --copy-metadata "*metadata.h5" \
  --compression gzip --compression-level 2
deactivate
```

## Final dataset — files & format
Identical to the GPVM run — see
[`fdhd_sparse_v10_20_interactive.md`](fdhd_sparse_v10_20_interactive.md) for the full
breakdown. In short:
- `pixeldata-anode0.h5 … anode11.h5` (12 APAs): per frame, `frame_rebinned_reco` +
  truth planes `frame_trackid_1st/2nd`, `frame_energyfrac_1st/2nd`, `frame_total_numelectrons`
  (each (2560, 1500)) + `channels_*`/`tickinfo_*`.
- `metadata.h5` — event metadata.
- `trackid_pid_map.h5` — `mcpart/` (all G4 tracks) + `simchnl/` (TPC-ionizing tracks) MC truth.
- After sparse conversion, `frame_*` become `coords`+`features` subgroups.

## Gotchas (off-site specifics)
- **Use `PRODUCTS` + `setup larwirecell`, not `mrbslp`.** `mrbslp`/`localProducts/setup` bake in
  the original `/exp/...` build path, which doesn't exist on your machine. The UPS product
  itself is relocatable (table file uses only `${UPS_PROD_DIR}`-relative paths), so the
  `PRODUCTS` method works from any directory.
- **SL7 binaries.** dunesw + the local build are `slf7.x86_64`; you must run inside the
  `fnal-dev-sl7` container (don't try native EL9).
- **Flux streaming can be slow.** If `dune.osgstorage.org` over CVMFS is too slow, pre-copy the
  flux files locally and point `gen_genie.fcl`'s `FluxSearchPaths` at the local copy, or set
  `FluxCopyMethod: "IFDH"` to copy-then-read. (Flux dir:
  `/cvmfs/dune.osgstorage.org/pnfs/fnal.gov/usr/dune/persistent/stash/Flux/g4lbne/v3r5p4/.../neutrino/dune10kt_horizdrift_1x2x6/`.)
- **Python wheels are cp39** to match the SL7 dunesw Python 3.9 — install them *inside* the
  container (Step 5), not on the host.
- **CVMFS cache**: first run pulls a lot; give the cvmfs cache a few GB.
