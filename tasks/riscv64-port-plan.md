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
| esbuild 0.28.1 (runtime dep of `chord`) | Yes | `@esbuild/linux-riscv64@0.28.1`, cpu=riscv64 [ran `npm view`] |
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

- [ ] BananaPi F3: install Node riscv64 >= 22.19 (unofficial-builds, debian trixie)
- [ ] `time npm i -g @earendil-works/pi-coding-agent --ignore-scripts` — log wall + CPU time
- [ ] `pi --version`, `pi --help`
- [ ] Confirm which of the two happened: clipboard native loaded, or nulled out
- [ ] Confirm `@esbuild/linux-riscv64` resolved (note: `--ignore-scripts` skips esbuild's
      postinstall verifier, so check the binary path resolves by hand)
- [ ] One real agent turn. Point it at the local llama-server from the existing
      article series rather than a paid API — that also feeds part 2/3 of the series.

Outcome is binary: "already works, here is the evidence" (then Phase 1-3 are the real
port) or a concrete failure list that rewrites everything below.

## Phase 1 — Build from source on riscv64

One blocker: **every package's `build` script calls `tsgo`**, which has no riscv64
binary. 13 call sites across `packages/*/package.json` and the root `check`. [grepped]

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
