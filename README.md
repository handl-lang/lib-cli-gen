# lib-cli-gen — generated handl SDKs for installed CLIs

One repo, many tools: `git`, `docker`, `ffmpeg`, `kubectl`, `aws`.

For each tool, `hnl sdkgen cli` emits two modules:

- `generated/<tool>/<tool>.hnl` — the **port** (types + arrow signatures,
  `cli`/`cli-c` contract)
- `generated/<tool>-cli-host/<tool>.hnl` — the **adapter** over
  `lib-cli`'s `cli-c` (posix_spawnp, no shell), a sibling dir of the port

## Committed source vs. generated output

`generated/` is **gitignored** — a build artifact, regenerated on demand:

```bash
for s in specs/*.clispec.yaml; do
  hnl sdkgen cli "$s" --output "generated/$(basename "$s" .clispec.yaml)"
done
```

`scripts/hnl-gate.sh` runs that loop on every gate (spec mode is
deterministic — re-emitting covers spec edits, not stale artifacts).

The committed source of truth is:

- `specs/<tool>.clispec.yaml` — the command tree each SDK is built from.
  Specs are probe-emitted where `hnl sdkgen cli <tool> --config
  probe.sdkgen.yaml` produces usable output (docker, kubectl), and
  hand-authored corrected baselines where the probe is garbage-level
  (git — unrecognized section headers; ffmpeg — usage groups misparsed
  as a command tree; aws — groff man-page help the parser can't read).
  Hand-patches are marked `# hand-authored` / `# hand-tuned` inline.
- `src/<tool>.test.hnl` — a smoke test per tool. Local-safe calls only:
  `git version`, `docker version --format {{.Client.Version}}` (client
  only — no daemon), `ffmpeg -version`, `kubectl version --client`
  (no cluster), `aws --version`.

## Regenerating a spec from a live tool

```bash
hnl sdkgen cli <tool> --config probe.sdkgen.yaml --output /tmp/gen/<tool>
# probe.sdkgen.yaml (CliConfig fields):
#   emit_spec: specs/<tool>.clispec.yaml
#   max_depth: 2          # top-level; a `probe:` block belongs in
#                         # *.clispec.yaml, not this overlay
#   drop: [<dotted.command.paths>]   # when a leaf fails validation
```

Then hand-tune the emitted spec and regenerate as above.

## Gate

```bash
./ci.sh        # or: scripts/hnl-gate.sh
HNL=/path/to/hnl scripts/hnl-gate.sh
```

`HANDL_EXT_DIR` must point at the built `ext-*` cdylibs for `sdkgen`
(`$CARGO_TARGET_DIR/debug` of the `host-cli` build).

Runtime dep: `lib-cli` (`cli` port + `cli-c` backend) — mapped in
`.handl.toml`.
