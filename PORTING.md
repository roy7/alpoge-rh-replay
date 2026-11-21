# Porting track: newer Lean and Mathlib

This is an optional track, separate from the replay. The replay checks the archive **exactly as published**, on Lean 4.33.1 and the pinned Mathlib commit, and nothing here changes it. Port work never earns replay credit. It never touches `verify.sh`, `Audit.lean`, `parts.sha256`, the claims bot or the completed ledger.

The port asks a different question: does the same proof still go through on a current toolchain? A port also lets newer checkers read the result. Lean 4.34.0 fixed three kernel soundness issues, and some v4.36.0 checkers cannot read 4.33-era files.

## Target

`port/target.json` records the target:

- **Lean:** `leanprover/lean4:v4.36.0`, the latest stable release (2026-11-16).
- **Mathlib:** `fc4f59e577feb1a4956671bc04ac18aaaf8cd842`, Mathlib's `v4.36.0` release tag.

### Retargeting the port

1. Open a separate retarget PR. Link the last full Git commit on the old target so reviewers can recover its target file, module records, patches and triage results together. Link the associated build transcripts and list the landed patches that need rework or still need testing.
2. In the same PR, update the port pins in `port/target.json`, remove the old module JSON and patch files from `port/patches/`, and retire the old results and reports from `port/triage/`. Git preserves the originals at the linked commit. A record's pins describe the build that produced it. Carry a record forward only after a fresh successful build on the new target.
3. Run `python3 tracker/validate_ports.py` and inspect `python3 tracker/port.py next`. Removing an active JSON record returns its module to the queue in dependency order. Depth-zero packets are eligible first, subject to existing reservations. Coordinate outstanding port issues with their claimants so each reservation refers to the intended target.
4. Run triage on the new pins. Recover any promising patch from the linked commit, apply it to the pinned source and follow the packet build and submission steps below. Record fresh evidence even when the old patch applies verbatim.

## Triage results

A Google Cloud job compiles each depth-0 module unchanged on the target toolchain and records whether it builds. When it fails, it records the first errors. Results are published in `port/triage/`. Start with modules marked `fail`. Modules marked `ok` need only an `unchanged` record, unless the `IsLattice` audit below reports them: compiling unchanged is not enough there.

## Unit of work

Work in **packets**, the same 64-module groups the replay uses (`python3 tracker/tasks.py show P0123`), in import-depth order. A module can only be ported once everything it imports is ported, so start at depth 0 and move up. `python3 tracker/tasks.py next --cold` lists shallow packets.

Each ported module is stored as a **patch against its pinned source**, not as a copy of the archive:

```
port/patches/<ModuleId>.patch       # unified diff against the pinned source file
port/patches/<ModuleId>.json        # {"module", "source_sha256", "lean", "mathlib", "notes"}
```

`source_sha256` must equal the module's hash in `tracker/modules.jsonl.gz`, so a patch always applies to the published file. If a module needs no changes, add only the `.json` with `"notes": "unchanged"`.

## Claim and submit

1. **Reserve.** Run `python3 tracker/port.py next` for the shallowest open packets whose imports are already ported, then `python3 tracker/port.py claim P0123`. Before opening a `Port P0123` issue with the `port` label, the command checks the target pins and whether any requested module still needs a record on upstream `main`. The replay claims bot ignores these titles, and maintainers resolve overlaps: the earlier open issue wins. Close the issue if you stop.
2. **Port.** Use a fresh checkout of the pinned archive with the target toolchain and Mathlib. Fix each module until `lake build +<Module>:olean` succeeds. Then run the `IsLattice` audit (below) on the packet's modules. Fix the source declaration behind every hit, including in modules that built unchanged, or explain in the notes why the reference is intended (for example a lemma that concludes the new `IsLattice` from `IsFullLattice`).
3. **Submit a PR to `main`** that adds the packet's `port/patches/` files and links the issue. Include the build transcript for every module in the packet, ending in success, plus the exact toolchain and Mathlib commit.

Use a title such as `Port P0123 to Lean 4.36` for the PR. The queue and claim
commands recognize packet/module IDs in open PR titles containing `port`,
`ports`, or `porting` (or carrying the `port` label). This keeps pending port
submissions reserved if their claim issue closes before the PR merges. These
checks share the existing live issue scan; unrelated replay PRs do not reserve
port work. Titles without task IDs cannot be recognized, so keep the claim open
until the PR merges.

If no whole packet is ready, `python3 tracker/port.py next --include-partial` also
lists unreserved packets with some ready modules. Whole packets come first;
partial packets are ordered by depth and then the number of ready modules.
Use `python3 tracker/port.py show P0123 --ready-only` to list the unported modules
whose imports are satisfied by records on your checkout or by other ready members
of the same packet. Refresh `main` before using these suggestions. Reserve the
packet, build only its ready members, and retain their successful outputs while
the other prerequisites are being ported. Recheck the remaining members after
updating `main`; submit the packet only when all its modules build successfully.

To reserve several independent packets, use `python3 tracker/port.py claim P0123 P0124`.
The command checks every target against upstream `main`, rejects overlapping
requests, and checks all reservations with one complete live issue scan before
creating any issue. Issue creation is sequential: if GitHub rejects a later
creation, earlier issues remain open. Review the printed issue URLs before retrying.

`tracker/port_build.py` can do steps 2 and 3 on Linux with elan, Git, GNU `patch` and `diff`. `prepare WORK` extracts the verified archive into `WORK/lean` on the target pins. `check P0123 WORK` builds each module of the packet and prints the first errors of each failure, after applying the merged patches of the packet's project imports; `--ready-only` builds just the members `port.py show P0123 --ready-only` lists. Edit the failing modules in `WORK/lean/src`, then `diff P0123 WORK` writes their patches against the pinned sources. `reproduce P0123 WORK OUT --issue N` rebuilds the packet in a fresh tree from the pinned archive plus the import and packet patches with `--no-cache`, and writes the build transcript, per-module logs and a receipt with source, patch, object and log digests. `records P0123 WORK --evidence URL --notes notes.json` writes the packet's records; the notes file maps each patched module to its note.

A module can build unchanged and still change meaning when Mathlib reuses a name for a different notion, as with `Submodule.IsLattice` (#26196). `check` warns when an unchanged module mentions a name on the tool's `REUSED_NAMES` list, and `records` refuses to record such a module as `unchanged` until you confirm with `--reviewed M000001` that every use keeps its pinned meaning. Add a name to the list when a port finds another one.

## Rules a port PR must follow

- **No `sorry`, `admit`, new `axiom`, `@[implemented_by]`, `native_decide` or `unsafe`** that the original module didn't have.
- **Don't change statements.** Proofs may change freely. A changed definition, theorem statement or instance needs a `notes` entry explaining why it is equivalent. Any change to `Main`, `Statements` or the five public theorems is reviewed separately and will usually be refused.
- **Keep it mechanical.** Prefer the smallest change: renamed lemmas, changed tactic behavior, new implicit arguments. Don't refactor or golf proofs as part of a port.
- **One packet per PR.** That keeps review and rework manageable.
- **`Submodule.IsLattice` must become `Submodule.IsFullLattice`.** Mathlib #44365 renamed the pinned `IsLattice` (finitely generated and spanning) to `IsFullLattice` and reused the name `IsLattice` for a weaker notion without `span_eq_top`. A statement or definition that still says `IsLattice` usually compiles unchanged but silently means the weaker notion. When the scalars are the fraction field of the coefficient ring (`[IsFractionRing R K]`, as for `𝓞 E` in a number field `E`), replace it with `IsFullLattice` and add an equivalence note: there `IsFullLattice ↔ FG ∧ span = ⊤`, which is the pinned definition, because `Submodule.IsLattice.of_fg` supplies the new field. Without that hypothesis `IsFullLattice` is strictly stronger than the pinned notion, so state `M.FG ∧ Submodule.span K M = ⊤` explicitly instead. Proofs may still use the new `IsLattice` API, for example `IsLattice.of_fg` or `IsLattice.free`.

## `IsLattice` audit

Text search cannot find these cases reliably: dot notation (`L.IsLattice K`) hides the namespace, and the archive defines several unrelated declarations named `IsLattice` in its own namespaces. `port/islattice_audit.lean` asks the elaborated environment instead. It reports every declaration of the given modules whose type, or whose body if it is a definition, refers to the constant `Submodule.IsLattice`, and exits with status 1 if there are any. From the ported tree, after building the modules:

```
lake env lean --run /path/to/port/islattice_audit.lean Module.One Module.Two ...
```

One source declaration can produce several hits, because a structure field also appears in its constructor, projection and recursor. A wrapper defined in another module is reported only in its own module, so a clean packet audit does not cover wrappers from elsewhere; the whole-tree audit below does. Exit status 2 means the audit could not run (for example a module is not built), not that it found hits.

Run it on every module of a packet before submitting. Run it as well before landing `unchanged` records in bulk from triage: build the modules whose pinned source mentions `IsLattice`, then audit them.

## When the port is complete

When every module has a patch or an `unchanged` record, a maintainer assembles the ported tree from the pinned archive plus the patches. They then build `Main`, run the axiom audit, the `IsLattice` audit over all project modules (any remaining hits must be ones the records' notes explain) and the Mathlib-only statement comparison, and run the independent checkers on the target toolchain. That result is reported **separately** from the replay's own verdict.
