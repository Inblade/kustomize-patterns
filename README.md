# Kustomize Patterns

A reference layout for Kustomize-based environments, extracted from production
GitOps repositories. It demonstrates the patterns that cover ~95% of real
needs: base + overlays, strategic-merge and JSON6902 patches, generators,
image pinning, and reusable components.

## Structure

```
.
├── base/                          # environment-agnostic manifests
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── dev/
│   │   └── kustomization.yaml     # replicas patch, configMapGenerator, name suffix
│   └── prod/
│       └── kustomization.yaml     # JSON6902 patch, images transformer, resource limits
├── components/
│   └── monitoring/                # opt-in ServiceMonitor as a Component
│       ├── kustomization.yaml
│       └── servicemonitor.yaml
└── docs/
    └── patches-cheatsheet.md      # which patch type to use when, with examples
```

## Usage

```bash
# Render an environment (kubectl >= 1.21 has kustomize built in)
kubectl kustomize overlays/dev
kubectl kustomize overlays/prod

# Apply
kubectl apply -k overlays/prod

# Diff against the cluster before applying — make this a habit
kubectl diff -k overlays/prod
```

## The rules this layout follows

1. **Base is deployable-ish but neutral**: sane defaults, no environment
   assumptions, replica count that is safe anywhere (1).
2. **Overlays own everything environment-specific**: replicas, resources,
   config values, image tags. If you find yourself editing base for one
   environment, it belongs in an overlay.
3. **Image tags live only in `images:` transformers** — never hardcoded in
   overlay patches. This is what CI bumps
   (`cd overlays/prod && kustomize edit set image app=repo/app:v1.2.3`).
4. **ConfigMaps go through generators** so every content change gets a new
   hashed name and triggers a rollout automatically. No more "we changed the
   ConfigMap but pods kept the old one".
5. **Cross-cutting opt-in features are Components** (`kind: Component`), not
   copy-pasted patches — see `components/monitoring/`.
6. Prefer **strategic-merge patches** for containers/env; use **JSON6902**
   only where merge semantics fail (list surgery, annotations with slashes,
   deletions). Details in [docs/patches-cheatsheet.md](docs/patches-cheatsheet.md).

## License

MIT — see [LICENSE](LICENSE).
