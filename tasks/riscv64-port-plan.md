# Task: Port `pi` to riscv64

Fork: https://github.com/gounthar/pi (from earendil-works/pi, MIT)
Upstream HEAD at fork time: `7968053`, version 0.84.4.

## The headline finding

There is no riscv64 *code* port to do. `pi` is TypeScript/JS end to end; the only C
in the tree is `packages/tui/native/darwin` and `packages/tui/native/win32`, neither
of which is built on Linux. [read the tree]

Better: the **published npm package already carries riscv64 in its shrinkwrap.**
`packages/coding-agent/npm-shrinkwrap.json` pins both
`@esbuild/linux-riscv64@0.28.1` and `@mariozechner/clipboard-linux-riscv64-gnu@0.3.9`,
and both are really published on npm with `cpu: riscv64`. [ran `npm view`; grepped shrinkwrap]

So the work splits into: prove the runtime path (probably already works), unblock the
*source build* (one real blocker), and extend distribution. Ordered by value.

## Evidence base

| Component | riscv64? | How I know |
|---|---|---|
| Node >= 22.19 (engines floor) | Yes | unofficial-builds index.json: 118 riscv64 releases incl. v26.0.0 [fetched] |
| esbuild 0.28.1 (monorepo only) | Yes | `@esbuild/linux-riscv64@0.28.1`, cpu=riscv64 [ran `npm view`]. **Correction:** not shipped in the published package at all — neither `esbuild` nor `@earendil-works/chord` is present in the installed tree, so this matters only for a source build [verified on hardware] |
| `@mariozechner/clipboard` 0.3.9 (optional) | Yes | `clipboard-linux-riscv64-gnu@0.3.9`, cpu=riscv64, 1.2MB [ran `npm view`] |
| `@silvia-odwyer/photon-node` | N/A | ships `photon_rs_bg.wasm`, arch-independent [read build script] |
| sqlite session backend | N/A | no native dep; deps are pure workspace packages [read package.json] |
| **`tsgo` (`@typescript/native-preview`)** | **No** | optionalDeps list linux-x64/arm/arm64 only [ran `npm view`] |
| **`bun`** | **No** | oven-sh/bun#6266 "Support riscv64" still open [fetched] |
| `@biomejs/biome` | No | cli-linux-{x64,arm64}{,-musl} only [ran `npm view`] |
| `canvas` 3.2.3 | source build | devDependency of `packages/ai` only; needs cairo/pango headers |

Clipboard degrades safely regardless: `loadClipboardNative()` returns `null` on
resolution failure, and it is already skipped on headless Linux via the
`DISPLAY`/`WAYLAND_DISPLAY` check. [read `packages/coding-agent/src/utils/clipboard-native.ts:31`]

## Phase 0 — Prove the runtime path on real hardware

Highest value, lowest cost. The evidence says `npm i -g @earendil-works/pi-coding-agent`
already works on riscv64. Nobody has ever tested it — upstream has zero issues or PRs
mentioning riscv [searched, 0 hits], so this is unproven, not proven.

- [x] BananaPi F3: install Node riscv64 >= 22.19 (unofficial-builds, debian trixie)
- [x] `time npm i -g @earendil-works/pi-coding-agent --ignore-scripts` — log wall + CPU time
- [x] `pi --version`, `pi --help`
- [x] Confirm which of the two happened: clipboard native loaded (riscv64 prebuilt)
- [x] Confirm `@esbuild/linux-riscv64` resolved (note: `--ignore-scripts` skips esbuild's
      postinstall verifier, so check the binary path resolves by hand)
- [x] One real agent turn (chat only; tool loop remains open, see above). Point it at the local llama-server from the existing
      article series rather than a paid API — that also feeds part 2/3 of the series.

Outcome is binary: "already works, here is the evidence" (then Phase 1-3 are the real
port) or a concrete failure list that rewrites everything below.

## Phase 0 RESULT — the runtime path works on riscv64

Run on `bananapif3-2` (BananaPi F3, SpaceMiT K1, 8 cores, 15G, Armbian 26.8.3 trixie,
Node v22.22.0) on 2026-09-04. All figures measured, not estimated.

**It works.** `npm install -g @earendil-works/pi-coding-agent --ignore-scripts` succeeded
with no riscv64-specific intervention of any kind.

| Measurement | Value |
|---|---|
| Install wall clock | **41.03 s** (127 packages, exit 0) |
| Install CPU | 46.51 s user + 8.44 s sys |
| Install peak RSS | 221 MB |
| `pi --version` | `0.84.4`, rc=0 |

That install time deserves a note: the plan was written expecting something like the 23
minutes OpenClaw took on this hardware. It was 41 seconds, because nothing is compiled
from source — every dependency resolves to a prebuilt or to pure JS.

### What was verified, scoped to pi's own tree

- **Native binding**: exactly one is needed and it is riscv64-correct —
  `node_modules/@mariozechner/clipboard-linux-riscv64-gnu/clipboard.linux-riscv64-gnu.node`.
  It loads with its full napi surface (18 exports incl. `getImageBinary`), resolved from
  pi's own tree, not from a neighbour. The darwin/win32 `.node` prebuilds ship in
  `pi-tui` but are inert on Linux.
- **No esbuild, no chord** in the published package. The plan's original claim that
  esbuild is a shipped runtime dependency was wrong; corrected in the table above.
- **Config/persistence**: pi created `~/.pi/agent/{auth.json,models-store.json}` on first
  run.
- **Error paths**: with no credentials, `--list-models` and `-p` both fail cleanly with an
  actionable message and rc=0, rather than crashing.

Method note: the first verification pass globbed the *global* npm root and reported
riscv64 packages that actually belonged to a pre-existing `openclaw` install on the same
board. Those were not evidence about pi. Every claim above was re-taken scoped to pi's
install root.

### Real inference on the board

No API keys were used. llama.cpp is a first-class provider in pi, and the board already
had a native riscv64 `llama-server` (`built with GNU 14.2.0 for Linux riscv64`,
`RISCV_V = 1 | RVV_VLEN = 32` — the vector extension is live).

- Model: `Llama-3.2-1B-Instruct-Q4_K_M`, 770 MB, downloaded in 72 s at ~10.7 MB/s
- Router: `llama-server --models-dir ~/models --jinja --port 8080`
- Baseline raw endpoint: `"Hello from RISC-V here."`, 8 tokens, 4.5 s
- **Through pi**: `"Hello from RISC-V I'm here."` — WALL 43.40 s, USER 6.31 s,
  MAXRSS 105 MB

The 43 s is model inference on a CPU, not pi overhead; pi's own share is the 6.31 s of
user CPU.

### The agent tool loop is NOT proven — open item

Two models were tried and **neither emitted a valid tool call**, so the agent loop has not
been demonstrated end to end on this board.

| Model | Result | Cost |
|---|---|---|
| Llama-3.2-1B-Instruct-Q4_K_M | Echoed the `read` tool JSON schema back as prose | 213.85 s wall |
| Qwen2.5-Coder-1.5B-Instruct-Q4_K_M | Replied `"Hello, world! How can I assist you today?"`, ignoring file and tool | 254.39 s wall |

Both runs sent ~1,400 tokens of tool schemas and came back with exactly **one** router
prompt — no second request, therefore no tool executed and no result fed back.

**This is a model-capability result, not a riscv64 result.** pi's tool-call parsing and
execution are plain JS with nothing architecture-specific in them. Proving the loop needs
a model competent at function calling, which at usable speed is beyond what an F3 does on
CPU: a 1.5B Q4 here generates at roughly 2 tok/s, so a 7B would be impractical. The
sensible way to close this is to point pi at a remote API once, purely to confirm the loop,
and keep local llama.cpp for the offline story.

A caution for whoever picks this up: **do not use output-token growth in the router log as
proof of a round trip.** An earlier attempt appeared to show one (prompt sizes 1432 -> 1491)
and it was an artefact. Killing a previous run with `pkill -f "pi -p"` had reaped the
`/usr/bin/time` wrapper but reparented the node child to init, where it kept generating
into the same log for another 11 minutes. Two runs, one log. The clean re-run — server
restarted, log truncated, single pi process verified — showed one prompt and no round trip.
Verify process isolation before trusting a shared log, and note that `pkill -f "pi -p"`
also matches the SSH command string carrying it, which killed the controlling session twice.

### Config gotcha worth writing down

`LLAMA_BASE_URL` alone did **not** make the provider visible — `--list-models` stayed
empty and `pi auth check --provider llama` returned `not_ready`. Hand-writing the
credential into `auth.json` (shape reverse-engineered from the bundle as
`{"type":"api_key","env":{"LLAMA_BASE_URL":...}}`) also returned `not_ready`. The
documented `/login llama.cpp` flow is interactive and needs a TTY.

What worked is the documented custom-provider path, `~/.pi/agent/models.json`:

```json
{ "providers": { "llamacpp-local": {
    "baseUrl": "http://127.0.0.1:8080/v1",
    "api": "openai-completions", "apiKey": "none",
    "models": [ { "id": "Llama-3.2-1B-Instruct-Q4_K_M" } ] } } }
```

Also note the router's load endpoints are `/models/load` and `/models/unload` — **not**
under `/api/`, which is Ollama-compat only.

### What this does to the rest of the plan

Phase 0 is answered: **no runtime port work is required.** Phases 1-3 stand unchanged and
are now the whole job — source build (`tsgo`), Docker, and the binary question. Phase 4's
issue can now lead with numbers instead of a proposal.

## Phase 1 — Build from source on riscv64

One blocker: **every package's `build` script calls `tsgo`**, which has no riscv64
binary. 13 call sites across `packages/*/package.json` and the root `check`. [grepped]

**Measured on the board 2026-09-04, and the failure mode is worse than the registry
metadata suggested.** `npm install @typescript/native-preview@7.0.0-dev.20260120.1` on
riscv64 **succeeds** — exit 0, `node_modules/.bin/tsgo` is created — and then throws when
executed:

```
Unable to resolve @typescript/native-preview-linux-riscv64. Either your platform is
unsupported, or you are missing the package on disk.
```

So `npm ci` looks green and the build is the thing that breaks. A CI job that gates on
install success would pass and mislead. (Install itself took 3m for one package, versus
41s for pi's whole 127-package tree — npm spent that time resolving, not compiling.)

#### Phase 1 finding: `tsc` is NOT a drop-in for `tsgo`

Measured on the board 2026-09-04. The plan assumed the swap was near-free and named that as
its main risk. The risk was real, and the reason has nothing to do with riscv64.

Shimming `node_modules/.bin/tsgo` to `tsc` 5.9.3 and running `npm run build:offline` builds
`chord`, then fails in `packages/tui`:

```
src/utils.ts(40,100): error TS1501: This regular expression flag is only available
                      when targeting 'es2024' or later.
```

Six occurrences. `packages/tui/src/utils.ts` uses ES2024 `v`-flag (unicodeSets) regexes —
`/\p{RGI_Emoji}/v`, `[\p{Spacing_Mark}--[...]]` set subtraction — while
`tsconfig.base.json` declares `"target": "ES2022"`, `"lib": ["ES2022"]`.

**`tsc` enforces TS1501; `tsgo` does not.** The build currently succeeds only because tsgo
skips that grammar check, so the repo has an undeclared dependency on tsgo-specific
permissiveness. Failed run: 74.05 s wall, 124.39 s CPU, `BUILD_EXIT=2`.

**x86_64 control, run to attribute this correctly.** Same machine, same tsconfig
(`target: ES2022`), same file, both compilers:

| Compiler | Result on `packages/tui` |
|---|---|
| `tsgo` 7.0.0-dev.20260120.1 | **exit 0**, emits `dist/utils.js` |
| `tsc` 5.9.3 | **exit 2**, the same six TS1501 errors, same lines |

So the divergence belongs to the compiler pair, not the architecture. Worth stating plainly
because "the build fails on riscv64" is the conclusion a single-platform run invites, and it
would have been wrong.

Two incidental notes from that control, both about the harness rather than the subject.
`tsc` still **emits** despite the TS1501 errors (`noEmitOnError` is not set) — it is the
non-zero exit that breaks the `&&` chain, so "dist exists" is not proof a package compiled
cleanly. And shimming by writing over `node_modules/.bin/tsgo` is unsafe: that path is a
symlink, so the write lands on `@typescript/native-preview/bin/tsgo.js` and corrupts the real
package for every consumer — including the `.orig` backup taken with `cp -a`, which copies
the link rather than the file. Reinstall the package to recover.

Candidate fix under test: `target`/`lib` -> `ES2024` in `tsconfig.base.json`. It clears the
TS1501 errors and `packages/tui` compiles. The regexes are runtime features that Node 22
already supports, so this declares what the code actually needs rather than changing
behaviour — but it is a real config change affecting every consumer's emit, not a riscv64
workaround, and it needs upstream's opinion rather than ours.

- [ ] Swap in `tsc`. `typescript@5.9.3` is already a root devDependency, and `tsgo -p X`
      is CLI-compatible with `tsc -p X`. Verify rather than assume — tsgo is a TS7
      preview and may differ on `erasableSyntaxOnly` / emit details.
- [ ] Implement as an indirection, not a fork-wide sed: a `PI_TSC` env var (default
      `tsgo`) consumed by the build scripts is the smallest upstreamable shape.
- [ ] Skip `npm run check` on riscv64 — biome has no riscv64 binary and lint is not
      needed to produce a build. Document that lint runs on x86 only.
- [ ] `canvas` (devDep of `packages/ai`): `apt install libcairo2-dev libpango1.0-dev
      libjpeg-dev libgif-dev librsvg2-dev`, expect a slow source compile. Only blocks
      that package's tests, not the build.
- [ ] Run `./test.sh`. Record what passes, what fails, what was skipped — do not report
      a partial run as green.
- [ ] Log wall + CPU time for the full build.

## Phase 1 RESULT — pi builds from source on riscv64

`BUILD_EXIT=0`. All ten packages compile and the esbuild bundle is produced, on the board,
2026-09-04. Three changes were needed, and only the first is about the architecture.

| Step | Measurement |
|---|---|
| `git clone --depth 1` + `npm install --ignore-scripts` | 316 packages, **268.23 s** wall (63.01 s user + 12.73 s sys), 196 MB |
| `npm run hydrate:model-data` | **7.52 s** |
| `npm run build:offline` | **593.85 s** wall (9m54s), **904.20 s user** + 26.32 s sys, 892 MB peak RSS |
| Artifacts | 10 `dist/` dirs; `packages/coding-agent/dist/bundle/{cli,client,coordinator,rpc-entry}.js` + chunks |
| `node dist/bundle/cli.js --version` | `0.84.4`, rc=0 |

### The three changes

1. **Shim `tsgo` to `tsc`** — the riscv64 blocker. Note the shim must not be written over
   `node_modules/.bin/tsgo` directly; see the symlink warning above.
2. **`target`/`lib` -> `ES2024`** in `tsconfig.base.json` — required by `tsc`, not by riscv64,
   per the x86_64 control.
3. **Run `hydrate:model-data` first** — a fresh clone has no `packages/ai/src/providers/data/*.json`
   and `build:offline` refuses without it. Upstream CI does this; it is not a port issue.

### What the numbers say about CI

904 s of CPU against 594 s of wall is ~1.5x parallelism, so this is mostly a serial
single-core TypeScript compile. A native riscv64 build is therefore ~10 minutes on an F3
plus 4.5 minutes of install — usable for verification, too slow for per-PR CI. Cross-building
on x86 and testing the artifact natively remains the right split, exactly as for the OpenClaw
image.

### The from-source build does real inference

```
$ node ~/pi-src/packages/coding-agent/dist/bundle/cli.js -p --no-session --no-tools \
    --model llamacpp-local/Llama-3.2-1B-Instruct-Q4_K_M "Say hello from RISC-V in five words."
"Hello from RISC-V."

WALL 32.21 s | USER 6.89 s | MAXRSS 121696 KB
```

Phase 1 is therefore **"builds and works"**, not merely "builds and starts".

### A self-inflicted detour, recorded so it is not repeated

Four runs before that one timed out (240 s, 300 s, 330 s, 330 s) and nearly went into this
document as a defect in the from-source build. They were not. Two controls caught it:

1. The **npm-installed** pi — already proven working in Phase 0 — timed out **identically**
   under the same flags. That exonerated the build.
2. Re-running the **exact Phase 0 command** (no added flags) completed in 44.53 s, proving
   the board and the server were fine.

The variable was isolated to a single change, by adding the suspects back one at a time:

| Configuration | Result |
|---|---|
| Baseline (npm pi, no extra flags) | **44.53 s**, answered |
| From-source build, same flags | **32.21 s**, answered |
| `+ --offline` only | **6.53 s**, answered — *faster*, it skips startup network work |
| `+ contextWindow: 4096, maxTokens: 64` only | **150 s timeout, 0 requests sent**, 4.31 s CPU |

So `--offline` was never the problem. **Setting `contextWindow`/`maxTokens` on a custom
provider model in `models.json` makes pi hang before it issues the HTTP request at all** —
the router log records no incoming request, and the process burns 4.31 s of CPU across
150 s of wall clock.

That is a reproducible finding worth reporting upstream, with one caveat this investigation
cannot lift: **it has only been observed on riscv64.** Whether it reproduces on x86_64 has
not been tested, so it must not be described as a riscv64 bug. Reproduce on x86 first; the
same discipline that corrected the TS1501 misattribution applies here.

The tell was in the numbers the whole time and was misread twice: 300 s of wall clock against
**5.25 s of user CPU** is a process blocked on I/O, not one doing slow work on a slow board.
When wall and CPU diverge by two orders of magnitude, suspect the harness, not the subject.

## Phase 2 — Docker image for riscv64

Reuse the pattern already proven in the OpenClaw work: separate `Dockerfile.riscv64`,
zero impact on the amd64/arm64 path.

- [ ] `FROM --platform=$BUILDPLATFORM` builder stage. This is *safer here than in
      OpenClaw*: both esbuild and tsc emit architecture-independent JS, and unlike
      rolldown/tsdown, esbuild has a real riscv64 binary — so native building is also
      an option, just slower.
- [ ] Runtime stage: `debian:trixie` (bookworm has no riscv64) + unofficial-builds Node,
      or reuse `gounthar/node-riscv64:22.22.0-trixie`.
- [ ] Decide native vs cross once Phase 1 gives real build timings. Cross-compiling is
      ~10x faster based on the OpenClaw measurements; native proves more.
- [ ] Health probe: pick one the image can actually pass. The OpenClaw image needed this
      fix (`fd16a5ce`) after the canonical probe cost ~14s against a 10s timeout.
- [ ] Publish to GHCR + Docker Hub, matching the OpenClaw tag scheme.

## Phase 3 — Standalone binary: blocked, with a workaround

`scripts/build-binaries.sh` uses `bun build --compile --target=bun-<platform>`, and the
release matrix is darwin/linux/windows x {x64, arm64}. Bun has no riscv64 target and
upstream #6266 is open, so **a true single-file `pi-linux-riscv64` cannot be produced
today.** This is not something the fork can solve.

Options, in the order I'd take them:

- [ ] **(a) Ship a Node-based tarball.** `pi-linux-riscv64.tar.gz` containing the esbuild
      bundle + assets + a launcher, requiring system Node. Closest to parity, low effort,
      honest about what it is. Recommended.
- [ ] (b) Node SEA (single-executable app, Node 22+). Real single file, but the bun entry
      is `src/bun/cli.ts` and would need a parallel Node entry — meaningful new surface.
- [ ] (c) Track oven-sh/bun#6266 and do nothing until it lands. Cheapest, indefinite.

Note for whichever path: `build-binaries.sh` force-installs the cross-platform clipboard
packages by explicit name and the riscv64 one is simply absent from that list — a
one-line addition, not a design problem.

## Phase 4 — Upstreaming (read this before touching GitHub)

**The contribution policy is unusually strict and must be respected literally.**
From `CONTRIBUTING.md`: all issues and PRs from new contributors are **auto-closed by
default**. A maintainer must reply `lgtm` before you may open a PR (`lgtmi` covers
issues only). Repeated policy violations or automation-driven volume get the **GitHub
account permanently blocked**. They also call out AI-generated submissions explicitly.

That file addresses AI agents directly ("if you use an agent, run it from the pi root
so it picks up AGENTS.md") — flagging that per our own external-repo rule. It reads as
a genuine maintainer policy rather than an injection attempt, and the sanction is real,
so the safe order is:

- [ ] Do **not** open a PR first.
- [ ] Land everything in the fork and get hardware evidence first (Phases 0-2).
- [ ] Open one short issue, in Bruno's own voice, one screen max, per their template.
      Content: riscv64 runs today, here are the numbers, here is the one build blocker.
      Say explicitly that you want to implement it.
- [ ] Wait for `lgtm`. Only then open a PR.
- [ ] Keep the upstream diff tiny — the `PI_TSC` indirection and the clipboard line in
      `build-binaries.sh`. The Docker image and the tarball stay in the fork; per their
      "core is minimal" philosophy those are extension/fork territory.

## Risks

- **tsc/tsgo divergence** — the whole Phase 1 unblock rests on them being interchangeable.
  Verify early; if they diverge, Phase 1 gets much more expensive.
- **Account blocking** — the upstream sanction for getting contribution etiquette wrong
  is permanent. Phase 4 order is not optional.
- **Bun** — outside our control. Phase 3 (a) exists so this does not block a release.
- **Board build times** — expect hours, not minutes. Budget accordingly; the boards
  cannot be the CI path.

## Open questions

- Is the runtime already green on riscv64, or not? Everything above is written twice
  depending on the Phase 0 answer.
- Does upstream *want* riscv64 at all? Zero prior issues means no signal either way.
