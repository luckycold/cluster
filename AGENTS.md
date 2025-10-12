# Repository Guidelines

## Project Structure & Module Organization
Configuration lives under `clusters/main`. Talos bootstrap files sit in `clusters/main/talos`; generated artifacts stay in `generated/`, and custom patches in `patches/`. Kubernetes manifests are grouped by domain inside `clusters/main/kubernetes`, where each workload folder keeps a Flux `ks.yaml` plus an `app/` subtree with the namespace, `helm-release.yaml`, and related ConfigMap or Secret overlays. Shared Git, Helm, and OCI sources are declared once in `repositories/`.

## Build, Test, and Development Commands
Run `kustomize build clusters/main/kubernetes` before every commit to catch YAML or schema errors. Use `flux diff ks authelia --path clusters/main/kubernetes/apps/authelia/app` (swap in your target path) to preview Flux changes. Secrets or config edits should be encrypted again with `sops --encrypt --in-place <file>` after editing via `sops <file>`. For Talos changes, validate with `talosctl validate --file clusters/main/talos/talconfig.yaml`.

## Coding Style & Naming Conventions
Follow YAML with 4-space indentation and lowercase keys. Kustomization manifests use `ks.yaml`; Helm releases stay in `helm-release.yaml`; SOPS-managed payloads end with `.secret.yaml` or `values.yaml`. Keep resource blocks alphabetized where practical, use blank lines sparingly for legibility, and avoid mixing secrets and plain config in the same file.

## Testing Guidelines
CI is intentionally light—the placeholder workflow only gates PRs. Rely on local validation: `kustomize build`, targeted `flux diff`, and `kubectl apply --server-side --dry-run=client -k <path>` against a staging cluster when available. For encrypted manifests, confirm `sops -d <file> | kubeconform --strict` when schemas exist, and note any manual checks (helm tests, smoke deploys) in the PR.

## Commit & Pull Request Guidelines
Match the existing Conventional Commit style (`fix:`, `chore(flux):`, etc.) and scope changes to the area touched. Keep commits focused on one logical change so Flux diffs stay readable. Pull requests must describe intent, reference related issues, and summarize validation (commands run, screenshots for UI workloads, or dashboard links). Flag secret rotations and call out new Age recipients. Wait for the placeholder workflow and automerge status before merging.

## Secrets & Configuration
Store the Age private key locally as `age.agekey`; never commit raw secrets. Files covered by `.sops.yaml` must remain encrypted—run `sops updatekeys` whenever recipients change. For new services, create separate `.secret.yaml` manifests instead of embedding secrets in `helm-release.yaml`, and reference the chart value that consumes them.
