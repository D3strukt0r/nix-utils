# AGENTS.md

Guidance for AI coding agents working in this repository.

## What this repo is

`nix-utils` is a tiny Nix flake shared across `d3strukt0r` projects. It is **not** an application — it has no JS, no Docker image, no website, no tests. The whole repo is essentially `flake.nix` plus community/CI plumbing.

It exposes:

- `lib.oci.secondsToNanos` and `lib.oci.createdFromDate` — pure Nix helpers for filling out OCI image config fields.
- `packages.<system>.fixOciImageHistory` — a `pkgs.writers.writePython3` derivation. It is a stdin→stdout filter for `dockerTools.streamLayeredImage` tarballs that:
  1. copies `history[].comment → history[].created_by` so dive / Docker Desktop show readable per-layer commands,
  2. appends a synthetic `HEALTHCHECK` history entry (Trivy DS-0026 false-positive workaround — Trivy reads `history[].created_by`, not `Config.Healthcheck`),
  3. recomputes the config-blob sha256 and rewrites `manifest.json`,
  4. drops `index.json` and `oci-layout` from the archive.

Supported systems: `x86_64-linux`, `aarch64-linux`, `riscv64-linux`.

## Common commands

```sh
# Validate the flake (eval + builds every output reachable from checks).
nix flake check --print-build-logs

# Build the package on the host system.
nix build --print-build-logs .#fixOciImageHistory

# Update the nixpkgs input pin (writes flake.lock).
nix flake update

# Quick smoke run against any streamLayeredImage script:
./your-stream-script | $(nix build --print-out-paths .#fixOciImageHistory) > out.tar
```

There is no test suite, no linter, and no build step beyond `nix build`. CI (`.github/workflows/ci.yml`) runs `nix flake check` + `nix build .#fixOciImageHistory` on `ubuntu-latest` only.

## Architecture notes worth knowing before editing

- **`flake.nix` keeps `nixpkgs` pinned to a branch (`nixos-25.11`), not a SHA.** Consumers are expected to set `inputs.nix-utils.inputs.nixpkgs.follows = "nixpkgs"` to avoid carrying a duplicate `nixpkgs` lock entry. Don't switch to a SHA pin.
- **`forAllSystems` only enumerates the three Linux systems above.** Adding macOS would mean adding it to that list — but `fixOciImageHistory` only makes sense on Linux (it processes Docker/OCI tarballs).
- **`fixOciImageHistory` is intentionally a single inlined Python script** inside `writePython3`. The `flakeIgnore` list silences specific pep8 codes (E302/E305/E401/E501/E702) that come from the compact style; if you reformat the script, revisit that list.
- **The synthetic HEALTHCHECK history entry is derived from `cfg.config.Healthcheck`** so the two never drift. If you add a new history entry there, mirror that pattern (read from canonical config, don't hand-craft strings the consumer has to keep in sync).
- **The script recomputes the config-blob digest and rewrites `manifest.json` accordingly.** Any change that mutates the config JSON must keep that digest recomputation step — otherwise `docker load` fails with a digest mismatch.

## Conventions

- Conventional Commits for commit messages (per README).
- The org-level `D3strukt0r/.github` repo holds `CONTRIBUTING.md` and `CODE_OF_CONDUCT.md` — those links in `README.md` point there on purpose; don't try to "fix" them by adding local copies.
- `.editorconfig` enforces LF, 2-space indent, UTF-8, final newline. Markdown files keep trailing whitespace (intentional hard linebreaks).
