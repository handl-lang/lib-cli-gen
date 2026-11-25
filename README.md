# lib-cli-gen — generated handl SDKs for installed CLIs

One repo, many tools: `git`, `docker`, `ffmpeg`, `kubectl`, `aws`.

For each tool, `hnl codegen cli` emits two modules:

- `generated/<tool>/<tool>.hnl` — the **port** (types + arrow signatures,
  `cli`/`cli-c` contract)
- `generated/<tool>/<tool>-cli-host/<tool>.hnl` — the **adapter**
  over `lib-cli`'s `cli-c` (posix_spawnp, no shell), nested inside the
  port dir (PL-004 `ArtifactRef` layout)

## Committed source vs. generated output

`generated/` is **gitignored** — a build artifact, regenerated on demand:

```bash
for s in specs/*.clispec.yaml; do
  hnl codegen cli "$s" --output "generated/$(basename "$s" .clispec.yaml)"
done
```

`scripts/hnl-gate.sh` runs that loop on every gate (spec mode is
deterministic — re-emitting covers spec edits, not stale artifacts).

The committed source of truth is:

- `specs/<tool>.clispec.yaml` — the command tree each SDK is built from.
  Specs are probe-emitted where `hnl codegen cli <tool> --config
  probe.sdkgen.yaml` produces usable output (docker, kubectl), and
  hand-authored corrected baselines where the probe is garbage-level
  (git — unrecognized section headers; ffmpeg — usage groups misparsed
  as a command tree; aws — groff man-page help the parser can't read).
  Hand-patches are marked `# hand-authored` / `# hand-tuned` inline.
- `src/<tool>.test.hnl` — a smoke test per tool. Local-safe calls only:
  `git version`, `docker version --format {{.Client.Version}}` (client
  only — no daemon), `ffmpeg -version`, `kubectl version --client`
  (no cluster), `aws --version`.

## Composing output

Adapters return `Result<Output, Spawn>`; `Output.stdout`/`stderr` are
plain `String`s. Commands stamped `meta:json-capable` get a
`--- json-capable` doc comment in the generated adapter — that's the
hint that stdout under `--json`/`--format`/`--output` is
machine-readable. Parse it by composing **free pure arrows**, not
methods on `Output` — `x.json()`-style adapters were rejected: not
composable, not treeshakeable, and they would hide the `~>` purity
line.

```handl
from "json" import { parse }          -- lib-json:  String ~> Value
from "str-handl" import { lines }     -- str-handl: String ~> [String]

r := gh_config_list()                  -- generated adapter → ok Output
v := parse(tag.untag(r).value.stdout)  -- `--json` stdout → Value
ls := lines(tag.untag(r).value.stdout) -- line-oriented stdout → lines
```

`json` resolves through the host module chain (kit-stdlib registers
it); `str-handl` is mapped in `.handl.toml` next to `cli`/`cli-c`.

`parse` is lenient: `""`/garbage/unclosed input returns `null`, as
does junk trailing a complete value. A stray unmatched closer
(`}`/`]`) with an empty frame stack *raises* an eval error
(`match: no arm matched`) — not an `err`-tagged `Result` — so guard
the call when the text is untrusted; under a `--json` flag you
requested, compose directly.

## Regenerating a spec from a live tool

```bash
hnl codegen cli <tool> --config probe.sdkgen.yaml --output /tmp/gen/<tool>
# probe.sdkgen.yaml (CliConfig fields):
#   emit_spec: specs/<tool>.clispec.yaml
#   max_depth: 2          # top-level; a `probe:` block belongs in
#                         # *.clispec.yaml, not this overlay
#   drop: [<dotted.command.paths>]   # when a leaf fails validation
```

Then hand-tune the emitted spec and regenerate as above.

## Gate

```bash
HNL=../host-cli/target/debug/hnl scripts/hnl-gate.sh  # or: scripts/hnl-gate.sh
HNL=/path/to/hnl scripts/hnl-gate.sh
```

`HANDL_EXT_DIR` must point at the built `ext-*` cdylibs for `codegen`
(`$CARGO_TARGET_DIR/debug` of the `host-cli` build).

Runtime dep: `lib-cli` (`cli` port + `cli-c` backend) — mapped in
`.handl.toml`.

## License

MIT.
