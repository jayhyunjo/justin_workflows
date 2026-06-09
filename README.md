# FD-HD sparse workflow (dunesw v10_20_08d00, local larwirecell build)

GEN → G4 → WireCell Sim+SP → Truth Labelling → dense→sparse HDF5, staged out as `out_*.tgz`.
Runs against `dunesw v10_20_08d00 -q e26:prof` with a **locally-built** `larwirecell`
(`v10_03_05`, branch `xn/trackid_pid_map`: adds `TrackIDPIDMap2h5` + Labelling2D
energy-fraction / total-num-electrons), shipped in the input bundle
`fdhd_v10_20_localwc_bundle.tar`.

Full details — bundle assembly, how the local build is set up on the worker, converter
schema, gotchas — are in
[`fdhd_sparse_v10_20_localbuild.md`](fdhd_sparse_v10_20_localbuild.md).

## Jobscript variants

All variants share one bundle and the same local larwirecell build; they differ only in the
generator fcl (beam) and the labelling fcl (truth on/off). Upload the bundle once, then choose
a variant with `--jobscript`.

| Jobscript | Beam (generator fcl) | Truth labels (labelling fcl) |
|---|---|---|
| `numu_withtruth.jobscript`          | νμ full FHC beam (`gen_genie.fcl`)                          | on  (`wcls-labelling2d_sep.fcl`) |
| `numu_notruth.jobscript`            | νμ full FHC beam (`gen_genie.fcl`)                          | off (`wcls-labelling2d_sep_notruth.fcl`) |
| `nue_withtruth.jobscript`           | νe appearance / swap (`gen_genie_nueswap.fcl`)             | on  (`wcls-labelling2d_sep.fcl`) |
| `nue_notruth.jobscript`             | νe appearance / swap (`gen_genie_nueswap.fcl`)             | off (`wcls-labelling2d_sep_notruth.fcl`) |
| `intrinsic_nue_withtruth.jobscript` | intrinsic beam νe, `GenFlavors:[12]` (`gen_genie_nue.fcl`) | on  (`wcls-labelling2d_sep.fcl`) |

**Beam**
- `numu_*` — full FHC beam (νμ-dominated, all flavors); the standard nominal beam.
- `nue_*` — νe **appearance / swap**: the νμ flux is relabeled as νe (`MixerConfig`), giving
  νe at full νμ-flux statistics. Best for a sizable νe sample.
- `intrinsic_nue_*` — **real** beam νe only (`GenFlavors:[12]`, ~1% of the flux): correct
  intrinsic-νe spectrum but low stats and slower GEN.

**Truth labels**
- `*_withtruth` — `pixeldata-anode*.h5` carry per-pixel truth frames (`trackid_1st/2nd`,
  `energyfrac_1st/2nd`, `total_numelectrons`) alongside `rebinned_reco`.
- `*_notruth` — `keep_truth=false`: pixeldata is **reco-only** (`rebinned_reco`).
- Either way, `trackid_pid_map.h5` (full MC-truth map) and `metadata.h5` are still produced.

### Environment variables
- `--env NUM_EVENTS=<n>` — events generated per job (default 10).
- `--env INPUT_TAR_DIR_LOCAL=<path>` — CVMFS path returned by `justin-cvmfs-upload` (required;
  supplies the local larwirecell build, FD-HD configs, converter, and python wheels).

### Submit
```bash
export USERF="$USER"
export FNALURL="https://fndcadoor.fnal.gov:2880/dune/scratch/users"
INPUT_TAR_DIR_LOCAL=$(justin-cvmfs-upload fdhd_v10_20_localwc_bundle.tar)   # re-run only if bundle changes

justin simple-workflow \
  --monte-carlo <NJOBS> --jobscript <VARIANT>.jobscript \
  --env NUM_EVENTS=<n> --env INPUT_TAR_DIR_LOCAL="$INPUT_TAR_DIR_LOCAL" \
  --rss-mib 4999 --output-pattern "out_*.tgz:${FNALURL}/${USERF}"
```
For a local (no-grid) test, replace `simple-workflow` with `justin-test-jobscript` (same
`--jobscript` / `--env`). Supporting fcl overrides live in [`configs/`](configs/).

---

# Generic Script

(Below are originally copied from Jake Calcutt (calcuttj))

`generic.jobscript` can run `lar` using various fcl files and on various samples (data or MC from PDHD or PDSP).
### Test command
<pre>justin-test-jobscript --mql &lt;MQL_REQUEST&gt; --jobscript generic.jobscript
</pre>

### Optional environment variables
`--env NEVENTS` -- controls how many events to parse (default -1/all)

`--env NFILES` -- controls how many files to go through (default 1)

`--env FCLFILE` -- controls which fcl file to use (default `pduneana_Prod4a_MC_sce.fcl`)

`--env OUTPREFIX` -- controls prefix to output file (default `duneana_ntuple`) 

`--env DUNE_VERSION` -- controls which version of dunesw to use (default `v09_79_00d00`)

`--env DUNE_QUALIFIER` -- controls which qualifiers of dunesw to use (default `e26:prof`)




## PDSP MC NTuples
### MQL
<pre>
workflow$ metacat query -s "files from dune:all where core.file_type=mc and core.data_tier=full-reconstructed and art.run_type=protodune-sp and dune.requestid=RITM1115963 and mc.space_charge=yes"
Files:        83648
Total size:   161124291092849 (161.124 TB)
</pre>

### FCL
`pduneana_Prod4a_MC_sce.fcl`

### Required options
`--env NTUPLE=1` -- controls which output flag (NTUPLE: `-T` or default: `-o`) is used.

## PDSP Data Keepup
### MQL
<pre>
workflow$ metacat query -s "files from dune:all where core.file_type=detector and core.data_tier=raw and core.run_type=protodune-sp"
Files:        922532
Total size:   4717153167566366 (4717.153 TB)
</pre>

### FCL
`protoDUNE_SP_keepup_decoder_reco.fcl`

## PDHP Data Keepup
### MQL
<pre>
workflow$ metacat query -s "files from dune:all where core.file_type=detector and core.data_tier=raw and 22949 in core.runs"
Files:        16
Total size:   66927584048 (66.928 GB)
</pre>

### FCL
`runpdhdwibethtpcdecoder.fcl`

### Required options
`--env HDF5JOB=1` -- Tells the job to run in a subshell with LD_PRELOAD set so hdf5 can be streamed via xrootd