# FD-HD truth-labelling — interactive run on a non-GPVM machine

Run the same chain as [`fdhd_sparse_v10_20_interactive.md`](fdhd_sparse_v10_20_interactive.md)
on **any Linux machine with CVMFS + apptainer** (your institution's cluster, a workstation),
instead of a FNAL `dunegpvm`.

```
GEN (GENIE) -> G4 -> WireCell Sim+SP -> 2D Truth Labelling -> (optional) dense->sparse HDF5
```

**You do NOT have to copy the 150 MB bundle.** With internet on the machine you can build
`larwirecell` yourself, `git clone` the configs, and `pip install` the Python deps. The bundle
is just an offline/quick shortcut. Two routes are given below — pick one.

---

## Common requirements (both routes)

1. **CVMFS** with these repos (system-mounted at `/cvmfs`, or via `cvmfsexec` if no root):
   `dune.opensciencegrid.org`, `larsoft.opensciencegrid.org`,
   `singularity.opensciencegrid.org` (SL7 image), `dune.osgstorage.org` (GENIE flux + CVN
   model), and `oasis.opensciencegrid.org` (only if you lack a local apptainer). Install
   `cvmfs-config-osg` so the repo keys resolve. **No grid proxy/token needed** (no justIN; flux
   is read from CVMFS).
   Without root: `git clone https://github.com/cvmfs/cvmfsexec.git && cd cvmfsexec && ./makedist osg`,
   then prefix commands with `./cvmfsexec <repos…> -- …`.
2. **apptainer** (or singularity) — system install, or the one under `oasis` CVMFS.
3. **SL7 binaries** — dunesw and the larwirecell build are `slf7.x86_64`; everything runs
   inside the `fnal-dev-sl7` container (don't run native EL9).

---

## Getting the ingredients — choose a route

You need three things: (1) the **larwirecell** build, (2) the **cffm-if** configs, (3) Python
deps for the optional sparse step.

### Route A — from sources (needs internet; nothing to copy)

- **larwirecell** — build it yourself with mrb (source:
  `https://github.com/Ningclover/larwirecell` branch **`xn/trackid_pid_map`**). Inside the SL7
  container, after `setup dunesw v10_20_08d00 -q e26:prof`:
  ```bash
  export MRB_PROJECT=larsoft
  mkdir -p ~/fdhd_dev && cd ~/fdhd_dev
  mrb newDev -v v10_20_08 -q e26:prof
  source localProducts_larsoft_v10_20_08_e26_prof/setup
  cd srcs
  git clone https://github.com/Ningclover/larwirecell.git -b xn/trackid_pid_map
  # REQUIRED FIX: this branch pins `wirecell v0_36_1` in ups/product_deps, but
  # dunesw v10_20_08d00 provides v0_34_2 -> the build fails without this edit.
  # (larevt v10_00_19 and larsim v10_20_02 already match the release.)
  sed -i 's/v0_36_1/v0_34_2/' larwirecell/ups/product_deps
  mrb uc                 # register larwirecell in srcs/CMakeLists.txt
  cd ..
  mrbsetenv
  mrb i -j8              # build + install -> localProducts/larwirecell/v10_03_05
  ```
  (Validated: this produces `larwirecell v10_03_05` with `libWireCellAIML.so` carrying the
  `TrackIDPIDMap2h5` module — identical to the prebuilt bundle.)
- **configs** — clone the cffm-if repo (configs are committed unmodified on this branch; the
  21 MB `ts-model/CP49_mobilenetv3.ts` is a plain git blob, so a normal clone fetches it):
  ```bash
  git clone -b xn/rebin_2 https://github.com/HaiwangYu/cffm-if.git
  ```
- **Python deps** (for the sparse step) — `pip install numpy h5py scipy` inside the container.

→ Setup uses **`mrbslp`** (your build sits at your own path, so the localProducts `setup`
script is valid — no relocation needed).

### Route B — portable bundle (offline / quick; copy one file)

```bash
scp <you>@dunegpvm0X.fnal.gov:/exp/dune/app/users/jjo/fdhd_v10_20_localwc_bundle.tar .
mkdir -p ~/fdhd && tar -xf fdhd_v10_20_localwc_bundle.tar -C ~/fdhd
# ~/fdhd/{localProducts (prebuilt larwirecell), cffm-if, convert-to-sparse.py, wheels (cp39)}
```
→ Setup uses the **`PRODUCTS` override** (a prebuilt product moved to a new path; `mrbslp`
would fail here because its `setup` hard-codes the original `/exp/...` build path).

---

## Enter the SL7 container
Bind `/cvmfs` and your working/ingredient dirs (no FNAL filesystems needed):
```bash
APPTAINER=apptainer        # or /cvmfs/oasis.opensciencegrid.org/mis/apptainer/current/bin/apptainer
"$APPTAINER" shell --shell=/bin/bash -B /cvmfs,$HOME --ipc --pid \
  /cvmfs/singularity.opensciencegrid.org/fermilab/fnal-dev-sl7:latest
```

## Setup (inside the container)
```bash
source /cvmfs/dune.opensciencegrid.org/products/dune/setup_dune.sh
setup dunesw v10_20_08d00 -q e26:prof
```
**Route A (self-built larwirecell):**
```bash
source ~/fdhd_dev/localProducts_larsoft_v10_20_08_e26_prof/setup
mrbslp
```
**Route B (copied prebuilt larwirecell):**
```bash
export PRODUCTS="$HOME/fdhd/localProducts:${PRODUCTS}"
setup larwirecell v10_03_05 -q e26:prof
```
**Both** — verify and point the search paths at the cffm-if configs:
```bash
ups active | grep '^larwirecell'        # expect v10_03_05
CFG=<path to cffm-if>/dune10kt-1x2x6     # Route A: ~/cffm-if/...  Route B: ~/fdhd/cffm-if/...
export FW_SEARCH_PATH="$CFG:${FW_SEARCH_PATH}"
export FCL_SEARCH_PATH="$CFG:${FCL_SEARCH_PATH}"
export FHICL_FILE_PATH="$CFG:${FHICL_FILE_PATH}"
export WIRECELL_PATH="$CFG:${WIRECELL_PATH}"
```

## Run the chain
```bash
cd "$CFG"      # configs (fcls, *.jsonnet, ts-model/) live here
NEV=10
lar -n $NEV -c gen_genie.fcl           -o gen.root
lar         -c g4.fcl         -s gen.root -o g4.root
lar         -c wcls_sim_sp.fcl -s g4.root -o sp.root
lar         -c wcls-labelling2d_sep.fcl -s sp.root      # keep_truth: true  -> writes the HDF5
```

## Optional: dense -> sparse
Use the cffm-if repo's own `dune10kt-1x2x6/exa_10evts_for_tag/densify_pixeldata.py`, **or**
jjo's `convert-to-sparse.py` (the version in the bundle is updated for the new
energyfrac/numelectrons frames):
```bash
# Route A deps (internet):   pip install numpy h5py scipy
# Route B deps (offline):    python3 -m venv ~/venv && source ~/venv/bin/activate && \
#   pip install --no-index --find-links ~/fdhd/wheels numpy==1.23.5 h5py==3.7.0 scipy==1.9.3
python convert-to-sparse.py --input-dir <raw .h5 dir> --output-dir sparse \
  --pattern "*pixeldata-anode*.h5" --copy-metadata "*metadata.h5" \
  --compression gzip --compression-level 2
```

## Final dataset — files & format
Identical to the GPVM run — see
[`fdhd_sparse_v10_20_interactive.md`](fdhd_sparse_v10_20_interactive.md). In short: 12
`pixeldata-anode*.h5` (per-frame `frame_rebinned_reco` + truth planes `frame_trackid_1st/2nd`,
`frame_energyfrac_1st/2nd`, `frame_total_numelectrons`, each (2560,1500), + `channels_*`/
`tickinfo_*`), `metadata.h5`, and `trackid_pid_map.h5` (`mcpart/` + `simchnl/` MC truth). After
sparse conversion each `frame_*` becomes `coords`+`features`.

## Gotchas (off-site)
- **`mrbslp` vs `PRODUCTS`**: use `mrbslp` only for a build you made *in place* (Route A). A
  prebuilt localProducts moved to a new machine/path (Route B) must use the `PRODUCTS` override
  — its `setup` script bakes in the original build path. The UPS product itself is relocatable
  (table uses only `${UPS_PROD_DIR}`-relative paths).
- **Repos/branches differ**: configs = `HaiwangYu/cffm-if` `xn/rebin_2`; larwirecell source =
  `Ningclover/larwirecell` `xn/trackid_pid_map`.
- **SimChannel smearing**: the upstream cffm-if `wcls-sim-drift-simchannel-nf-sp.jsonnet` is
  *sharp*. For truth↔reco registration, apply the two `smear_long`/`smear_tran` keys (or use
  the tracked `configs/wcls-sim-drift-simchannel-nf-sp.jsonnet`). See §11 of the localbuild doc.
- **Flux streaming** from `dune.osgstorage.org` over CVMFS can be slow off-site. If so, set
  `gen_genie.fcl`'s `FluxCopyMethod: "IFDH"` (copy-then-read) or point `FluxSearchPaths` at a
  local copy of the flux files.
- **Python is cp39** in the SL7 dunesw env — for Route B install the cp39 wheels *inside* the
  container; for Route A a plain `pip install` inside the container picks compatible builds.
- Give the CVMFS cache a few GB (first run pulls a lot).
