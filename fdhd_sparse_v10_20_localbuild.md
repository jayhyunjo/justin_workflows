# FD-HD sparse workflow on justIN with a LOCAL larwirecell build

This documents how to run the FD-HD chain **GEN → G4 → WireCell Sim+SP → Truth Labelling →
dense→sparse HDF5** on justIN using a **locally-built (non-official) `larwirecell`**, shipped
to the grid worker nodes inside the input bundle. It generalizes to any future case where you
need a custom-built UPS product in a justIN job.

- Jobscripts: five variants (beam × truth on/off) — see **Jobscript variants** below
- Base release: `dunesw v10_20_08d00 -q e26:prof`
- Custom product: `larwirecell` UPS `v10_03_05`, branch `xn/trackid_pid_map`
  (`Ningclover/larwirecell`) — adds `TrackIDPIDMap2h5` (MC-truth → HDF5) and Labelling2D
  energy-fraction / total-num-electrons outputs.

---

## Jobscript variants

All variants share one bundle (`fdhd_v10_20_localwc_bundle.tar`) and the local larwirecell
build; they differ only in which generator + labelling fcl the jobscript runs. Upload the
bundle once, then pick a variant with `--jobscript`.

| Jobscript | Beam (generator fcl) | Truth labels (labelling fcl) |
|---|---|---|
| `fdhd_sparse_v10_20_numu_withtruth.jobscript`          | νμ full FHC beam (`gen_genie.fcl`)              | on  (`wcls-labelling2d_sep.fcl`) |
| `fdhd_sparse_v10_20_numu_notruth.jobscript`            | νμ full FHC beam (`gen_genie.fcl`)              | off (`wcls-labelling2d_sep_notruth.fcl`) |
| `fdhd_sparse_v10_20_nue_withtruth.jobscript`           | νe appearance / swap (`gen_genie_nueswap.fcl`)  | on  (`wcls-labelling2d_sep.fcl`) |
| `fdhd_sparse_v10_20_nue_notruth.jobscript`             | νe appearance / swap (`gen_genie_nueswap.fcl`)  | off (`wcls-labelling2d_sep_notruth.fcl`) |
| `fdhd_sparse_v10_20_intrinsic_nue_withtruth.jobscript` | intrinsic beam νe, `GenFlavors:[12]` (`gen_genie_nue.fcl`) | on  (`wcls-labelling2d_sep.fcl`) |

"Truth off" flips `keep_truth → false` so `pixeldata-anode*.h5` is reco-only (`rebinned_reco`);
`trackid_pid_map.h5` + `metadata.h5` are still produced. See §6 (converter) and §10 (variants).

---

## 1. The core problem and the idea

Grid workers can only see **CVMFS** and **dCache** — they cannot see `/exp` or `/nashome`.
The official `dunesw v10_20_08d00` (and its `larwirecell v10_03_04`) is on CVMFS, but our
custom `larwirecell` build lives under `/exp/dune/data/users/xning/...`, invisible to workers.

So we **package the locally-built product into the bundle**, upload the bundle to CVMFS with
`justin-cvmfs-upload`, and in the jobscript set the product up directly via UPS — overriding
the official one.

Key facts that make this work:
- The only locally-built product is `larwirecell` (the mrb dev area builds nothing else).
- mrb-built UPS products are **relocatable**: the table file
  (`.../slf7.x86_64.e26.prof/ups/larwirecell.table`) uses only `${UPS_PROD_DIR}`-relative
  paths, and the `.version` file has no absolute paths.
- Our build is `v10_03_05`, **higher** than the official `v10_03_04`, so a plain
  `setup larwirecell v10_03_05` from a prepended `PRODUCTS` wins cleanly.
- The build is `slf7.x86_64`, matching the SL7 container justIN jobs run in.
- Build dependencies (`larevt v10_00_19`, `larsim v10_20_02`, `wirecell v0_34_2`) are exactly
  what `dunesw v10_20_08d00` provides → ABI compatible.

> **Do NOT** use `mrbslp` / `localProducts.../setup` on the worker — that script is explicitly
> *not relocatable* (it bakes in the `/exp/...` build path, which doesn't exist on a worker).

---

## 2. Building the local product (reference)

Done in xning's mrb dev area, inside the SL7 container. Summary (see the area's own README):
```bash
source /cvmfs/dune.opensciencegrid.org/products/dune/setup_dune.sh
setup dunesw v10_20_08d00 -q e26:prof
# mrb newDev -v v10_20_08 -q e26:prof ; clone larwirecell -b xn/trackid_pid_map
source .../localProducts_larsoft_v10_20_08_e26_prof/setup
mrbsetenv
mrb i -j 8         # builds larwirecell into localProducts
```
Result: `localProducts_larsoft_v10_20_08_e26_prof/larwirecell/v10_03_05/`.

---

## 3. Assembling the bundle

The bundle (`fdhd_v10_20_localwc_bundle.tar`, ~150 MB) is a tar of these top-level entries:

```
localProducts/              # the relocatable larwirecell UPS product (runtime bits only)
  .upsfiles/                #   UPS product DB config (dbconfig)
  larwirecell/v10_03_05/    #   the built product (lib/, fcl/, include/, ups/table)
  larwirecell/v10_03_05.version/
cffm-if/dune10kt-1x2x6/     # FD-HD configs (NOT in CVMFS dunesw)
  gen_genie.fcl  g4.fcl  wcls_sim_sp.fcl  wcls-labelling2d_sep.fcl
  *.jsonnet                 #   wire-cell pipeline configs
  ts-model/CP49_mobilenetv3.ts
convert-to-sparse.py        # dense→sparse HDF5 converter (schema updated, see §6)
wheels/                     # offline python deps: numpy 1.23.5, h5py 3.7.0, scipy 1.9.3 (cp39)
```

Build steps (from a clean staging dir):
```bash
STAGE=~/fdhd_v10_20_bundle
SRC_LP=/exp/dune/data/users/xning/larsoft/v10_20_08d00/localProducts_larsoft_v10_20_08_e26_prof
SRC_CFG=/exp/dune/data/users/xning/larsoft/cffm-if/dune10kt-1x2x6

mkdir -p "$STAGE/localProducts" "$STAGE/cffm-if/dune10kt-1x2x6"
# runtime bits only — skip build_slf7.x86_64/ and srcs/
cp -a "$SRC_LP/.upsfiles" "$SRC_LP/larwirecell" "$STAGE/localProducts/"
# clean config only — no .root/.log/.png clutter
cp -a "$SRC_CFG"/*.fcl "$SRC_CFG"/*.jsonnet "$SRC_CFG/ts-model" "$STAGE/cffm-if/dune10kt-1x2x6/"
cp -a convert-to-sparse.py wheels "$STAGE/"     # converter + wheelhouse

tar -cf ~/fdhd_v10_20_localwc_bundle.tar -C "$STAGE" localProducts cffm-if convert-to-sparse.py wheels
```

Notes:
- The wheels are `cp39` because `dunesw v10_20_08d00` ships **Python 3.9.15**. If you bump the
  release and python changes, re-`pip download` matching wheels.
- `SPE_DAPHNE2_FBK_2022.dat` and the base `pgrapher/experiment/dune10kt-1x2x6/*` jsonnet are
  provided by the CVMFS `wirecell` product — not bundled.

---

## 4. Shipping it: `justin-cvmfs-upload`

```bash
INPUT_TAR_DIR_LOCAL=$(justin-cvmfs-upload fdhd_v10_20_localwc_bundle.tar)
echo "$INPUT_TAR_DIR_LOCAL"   # e.g. /cvmfs/fifeuser3.opensciencegrid.org/sw/dune/<hash>
```
`justin-cvmfs-upload` unpacks the tar and publishes it to CVMFS; `INPUT_TAR_DIR_LOCAL` is the
**directory** (so files are at `$INPUT_TAR_DIR_LOCAL/localProducts`, etc.). The path is
content-addressed — re-uploading identical content gives the same path; changed content gives
a new one. Allow a few minutes for CVMFS propagation before the path is readable.

---

## 5. How the jobscript reads the local build

The relevant part of [`fdhd_sparse_v10_20_numu_withtruth.jobscript`](fdhd_sparse_v10_20_numu_withtruth.jobscript):

```bash
source /cvmfs/dune.opensciencegrid.org/products/dune/setup_dune.sh
setup dunesw v10_20_08d00 -q e26:prof

# Override official larwirecell with the local build from the bundle:
export PRODUCTS="${INPUT_TAR_DIR_LOCAL}/localProducts:${PRODUCTS}"
setup larwirecell v10_03_05 -q e26:prof
ups active | grep '^larwirecell'    # verify: v10_03_05 from the bundle path

# Point fcl / wire-cell search paths at the bundled FD-HD configs:
export FW_SEARCH_PATH="${INPUT_TAR_DIR_LOCAL}/cffm-if/dune10kt-1x2x6:${FW_SEARCH_PATH}"
export FCL_SEARCH_PATH="${INPUT_TAR_DIR_LOCAL}/cffm-if/dune10kt-1x2x6:${FCL_SEARCH_PATH}"
export FHICL_FILE_PATH="${INPUT_TAR_DIR_LOCAL}/cffm-if/dune10kt-1x2x6:${FHICL_FILE_PATH}"
export WIRECELL_PATH="${INPUT_TAR_DIR_LOCAL}/cffm-if/dune10kt-1x2x6:${WIRECELL_PATH}"
```
The `setup larwirecell` prepends the custom `lib/` to `CET_PLUGIN_PATH` (so ART loads the
custom WCLS tool/modules) and `fcl/` to `FHICL_FILE_PATH`. The chain then runs `gen_genie.fcl`
→ `g4.fcl` → `wcls_sim_sp.fcl` → `wcls-labelling2d_sep.fcl`, collects `pixeldata-anode*.h5`,
`metadata.h5`, and `trackid_pid_map.h5`, runs the sparse conversion, and tars `sparse/` into
`out_<label_tag>.tgz`.

Truth labelling is **on** (`keep_truth: true` in `wcls-labelling2d_sep.fcl`).

---

## 6. Sparse converter schema (IMPORTANT when the labelling output changes)

`convert-to-sparse.py` has **hardcoded** dataset lists. The `xn/trackid_pid_map` Labelling2D
emits energy-fraction + total-num-electrons frames (replacing the older `pid_*` frames), so
the lists were updated to:
```python
FRAMES_2D = [frame_rebinned_reco, frame_trackid_1st, frame_trackid_2nd,
             frame_energyfrac_1st, frame_energyfrac_2nd, frame_total_numelectrons]
ARRAYS_1D = [channels_/tickinfo_ variants of the same six]
```
Each `frame_*` is stored sparsely as `coords` (nonzero indices) + `features`. The
`trackid_pid_map.h5` truth map is copied **as-is** (not pixel-image data).

> If you change the labelling fcl/module so it emits different datasets, **update these lists**
> or they will be silently dropped from the staged-out sparse files.

---

## 7. Submitting to justIN — step by step

**0. Enter the SL7 container + set up justin** (once per login; the web auth lasts 7 days):
```bash
~/dune_container.sh                                              # apptainer shell into fnal-dev-sl7
source /cvmfs/dune.opensciencegrid.org/products/dune/setup_dune.sh
setup justin
justin get-token          # prints a dunejustin.fnal.gov/authorize URL -> open it, confirm the session code
```

**1. Upload the bundle** → capture `INPUT_TAR_DIR_LOCAL` (re-run only when the bundle changes;
one upload serves all jobscripts):
```bash
cd /exp/dune/app/users/jjo
INPUT_TAR_DIR_LOCAL=$(justin-cvmfs-upload fdhd_v10_20_localwc_bundle.tar)
echo "$INPUT_TAR_DIR_LOCAL"
```
Wait ~2–5 min for CVMFS to propagate before submitting (an immediate "localProducts not found"
just means it propagated late — resubmit).

**2. Stage-out destination:**
```bash
export USERF="$USER"
export FNALURL="https://fndcadoor.fnal.gov:2880/dune/scratch/users"
cd /exp/dune/app/users/jjo/justin_workflows
```

**3. Submit** — pick a jobscript from the variant table above; scale with `--monte-carlo`
(number of jobs) and `NUM_EVENTS` (events/job). For a new/changed variant, start with 1 job,
then 10, then full:
```bash
justin simple-workflow \
  --monte-carlo 1 \
  --jobscript fdhd_sparse_v10_20_numu_withtruth.jobscript \
  --env NUM_EVENTS=10 \
  --env INPUT_TAR_DIR_LOCAL="$INPUT_TAR_DIR_LOCAL" \
  --rss-mib 4999 \
  --output-pattern "out_*.tgz:${FNALURL}/${USERF}"
```
Prints a **workflow ID**. (For a local, no-grid dry run: replace `simple-workflow` with
`justin-test-jobscript` — same `--jobscript`/`--env`; outputs land in
`/tmp/justin-test-jobscript.XXXXXX/home/workspace`.)

**4. Monitor:**
```bash
justin show-workflows --workflow-id <ID>     # overall state
justin show-stages    --workflow-id <ID>     # % progress
justin show-jobs      --workflow-id <ID>     # per-job states
justin fetch-logs --jobsub-id <JOBSUB_ID> --unpack    # on a failure, then read jobscript.log
```
justIN over-subscribes for MC targets (you'll see more jobs than `--monte-carlo`); an occasional
`jobscript_error` is usually transient flux-file I/O on a flaky site and is retried automatically.

**5. Collect outputs** (dCache scratch):
```bash
gfal-ls -l ${FNALURL}/${USERF}/ | grep out_        # list staged-out tarballs
gfal-copy ${FNALURL}/${USERF}/out_<...>.tgz ./      # pull one down
tar -tzf out_<...>.tgz                              # sparse/*.h5 + metadata.h5 + trackid_pid_map.h5
```

**Typical flow for a fresh variant:** upload (step 1) → `justin-test-jobscript` (local sanity) →
`simple-workflow --monte-carlo 1` (verify output) → `--monte-carlo 10` → full.

---

## 8. Environment / gotchas

- Run everything inside the SL7 container (`fnal-dev-sl7`):
  ```bash
  /cvmfs/oasis.opensciencegrid.org/mis/apptainer/current/bin/apptainer shell --shell=/bin/bash \
    -B /cvmfs,/exp,/nashome,/pnfs/dune,/opt,/run/user,/etc/hostname,/etc/hosts,/etc/krb5.conf \
    --ipc --pid /cvmfs/singularity.opensciencegrid.org/fermilab/fnal-dev-sl7:latest
  ```
  On AlmaLinux 9 hosts the SL7 container is required (the official `dunesw` is built for SL7).
- `justin` server commands need a one-time browser authorization (`justin get-token` →
  `https://dunejustin.fnal.gov/authorize/...`), valid 7 days. The WLCG bearer token alone
  (used by `justin-cvmfs-upload`) is not sufficient for `simple-workflow`/`test-jobscript`.

---

## 9. Verification checklist (per the validated Stage-A run)

- `ups active | grep larwirecell` → `v10_03_05` from the bundle's `localProducts` path.
- `trackid_pid_map.h5` present, with `1/mcpart/{track_ids,pids,mother_ids,mother_pids,
  processes,start_xyzts,end_xyzts,start_moms,end_moms,end_processes,statuses,masses,
  ndaughters,ntrajpts}` and `1/simchnl/{...,energies}`; masses match PDG values.
- dense `pixeldata-anode*.h5` contain `frame_energyfrac_1st/2nd` + `frame_total_numelectrons`.
- sparse `pixeldata-anode*.h5` in `out_*.tgz` contain those same frames as `coords/features`,
  plus `trackid_pid_map.h5` copied as-is.

---

## 10. Flavor variants (hardcoded), e.g. intrinsic nu_e only

To run a single beam flavor, add a thin GENIE override fcl plus a sibling jobscript that uses
it for the GEN stage — the full-beam files stay untouched:

- [`configs/gen_genie_nue.fcl`](configs/gen_genie_nue.fcl) — `#include "gen_genie.fcl"` then
  `physics.producers.generator.GenFlavors: [ 12 ]` (PDG 12 = nu_e). This selects only the
  intrinsic beam nu_e rays from the numu-dominated LBNF flux → a pure nu_e sample with the
  correct beam-nu_e spectrum. (GEN is somewhat slower, since nu_e is ~1% of the flux.)
- [`fdhd_sparse_v10_20_intrinsic_nue_withtruth.jobscript`](fdhd_sparse_v10_20_intrinsic_nue_withtruth.jobscript) — identical to
  `fdhd_sparse_v10_20_numu_withtruth.jobscript` except the GEN stage runs `gen_genie_nue.fcl`.

Other flavors: `[ -12 ]` = nu_e_bar, `[ 14 ]` = nu_mu, `[ -14 ]` = nu_mu_bar.

`configs/gen_genie_nue.fcl` is the tracked source; it must also be placed in the bundle's
`cffm-if/dune10kt-1x2x6/` (next to `gen_genie.fcl`, which it includes) before
`justin-cvmfs-upload`.

Validated (local `justin-test-jobscript`, 1 evt): GENIE MCTruth `nu_pdg = 12 (nu_e) CC`;
sparse output carries the energyfrac/numelectrons frames plus `trackid_pid_map.h5`.

### Intrinsic nu_e vs nu_e-appearance ("swap") — prefer swap for statistics

`gen_genie_nue.fcl` above selects the **intrinsic** beam nu_e (~1% of the flux) → physically
real beam nu_e, but low rate and slow to generate. For a *sizable* nu_e sample, use the
**swap** sample instead, which relabels the dominant numu flux as nu_e (the nu_e-appearance
signal) — nu_e at full numu-flux statistics, and fast to generate:

- [`configs/gen_genie_nueswap.fcl`](configs/gen_genie_nueswap.fcl) — `#include "gen_genie.fcl"`
  then `physics.producers.generator.MixerConfig: "map 12:16 -12:-16 14:12 -14:-12"`
  (`14->12` numu→nu_e; `12->16` intrinsic nu_e→nu_tau to avoid double counting). This is the
  one-line difference between DUNE's `prodgenie_nu_dune10kt_1x2x6.fcl` (nonswap) and
  `prodgenie_nue_dune10kt_1x2x6.fcl` (swap); we apply it on top of the customized
  `gen_genie.fcl` so the fiducial/beam-center customizations are preserved.
- [`fdhd_sparse_v10_20_nue_withtruth.jobscript`](fdhd_sparse_v10_20_nue_withtruth.jobscript) — GEN stage
  runs `gen_genie_nueswap.fcl`; rest identical.

Same flux files as `gen_genie.fcl` (no new data). Keeps the base `GenFlavors`, so the sample
is nu_e-dominant with a small nu_e_bar component (FHC); add `GenFlavors: [ 12 ]` for strictly
nu_e. Validated (local, 1 evt): `nu_pdg = 12 (nu_e) CC`, fast GEN, sparse frames +
`trackid_pid_map.h5` present.

---

## 11. SimChannel smearing (truth ↔ reco registration) — ENABLED in this bundle

By default the truth SimChannel is *sharp* relative to the SP/reco charge (it carries drift
diffusion but not the field-response + electronics + SP-deconvolution broadening), so a rim of
real reco charge at track/shower edges gets truth label 0 (Background). This bundle enables
**in-pipeline SimChannel smearing** to fix that at the source: the truth footprint is broadened
to SP/reco resolution so the 2D labels register onto the deconvolved reco charge.

**The change** — two keys on the `wclsDepoFluxWriter` node named `postdrift` in
[`configs/wcls-sim-drift-simchannel-nf-sp.jsonnet`](configs/wcls-sim-drift-simchannel-nf-sp.jsonnet)
(the Sim+SP graph that `wcls_sim_sp.fcl` loads via its `configs:` list):
```jsonnet
smear_long: 2.6526,                        // raw ticks; = sigma_t/tick, sigma_t = 1/(2*pi*0.12MHz) = 1.326 us
smear_tran: [0.37612, 0.37612, 0.09403],   // wire pitch, per plane [U,V,W]; induction k=0.75->0.376, collection k=3.0->0.094
```
`DepoFluxWriter` adds these Gaussian widths **in quadrature** on top of each post-drift depo's
physical diffusion (`0.0` = no *extra* smearing = the stock/sharp behavior). No C++, tagger,
or converter change. The numbers are the dune10kt-1x2x6 SP-filter recipe (time: `Gaus_wide`
σ_f=0.12 MHz; wire: σ_w = 1/(2·√π·k), induction k=0.75, collection k=3.0).

**Scope**: applies to **all** variants — every jobscript runs `wcls_sim_sp.fcl` → this one
jsonnet. Only the `*_withtruth` samples are affected in practice (`*_notruth` pixeldata is
reco-only). Reco (`frame_rebinned_reco`) is unchanged; only the truth footprint
(`frame_total_numelectrons` + label frames) grows to match reco.

**Off-site / from-source**: the upstream `HaiwangYu/cffm-if` jsonnet is *sharp* — apply the two
keys above, or use the tracked `configs/wcls-sim-drift-simchannel-nf-sp.jsonnet`. To revert to
sharp, delete the two keys (or set both `0.0`).

**Verify from a `*_withtruth` output**: the untagged reco-charge fraction on the collection
plane (W) should be ~0% (sharp: a few %); the truth footprint grows ~35–55%; reco is unchanged.
