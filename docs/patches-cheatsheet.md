# Kustomize Patches Cheatsheet

Which mechanism to use for which change, ordered from simplest to most surgical.
Rule of thumb: **use the least powerful tool that does the job** — simple
transformers > strategic merge > JSON6902.

## 0. Built-in shortcuts (use these first)

| Change | Kustomization field |
|--------|--------------------|
| Replica count | `replicas: [{name: app, count: 3}]` |
| Image name/tag/digest | `images: [{name: app, newTag: v1.2.3}]` |
| Namespace for everything | `namespace: app-prod` |
| Name prefix/suffix | `namePrefix:` / `nameSuffix:` |
| Labels everywhere | `labels:` (with `includeSelectors` control) |
| Annotations everywhere | `commonAnnotations:` |

If a built-in transformer covers it, a patch is the wrong tool.

## 1. Strategic-merge patch

Merges your fragment with the target using Kubernetes merge semantics —
container lists merge **by `name` key**, maps merge recursively.

```yaml
patches:
  - patch: |-
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: app
      spec:
        template:
          spec:
            containers:
              - name: app          # merge key — matches the existing container
                resources:
                  limits:
                    memory: 1Gi
```

Great for: resources, env, probes, adding sidecars, anything keyed by name.

**Delete** something with the `$patch: delete` directive:

```yaml
patches:
  - patch: |-
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: app
      spec:
        template:
          spec:
            containers:
              - name: istio-proxy
                $patch: delete
```

Limitations: lists **without** a merge key (e.g. `topologySpreadConstraints`,
`tolerations`) are *replaced wholesale*, not merged — surprises live here.

## 2. JSON6902 patch (RFC 6902 ops)

Explicit operations against exact paths. Requires a `target:` selector.

```yaml
patches:
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: app
    patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/imagePullPolicy
        value: Always
      - op: add
        path: /spec/template/spec/tolerations/-      # '-' = append to list
        value: {key: dedicated, operator: Equal, value: app, effect: NoSchedule}
      - op: remove
        path: /spec/template/spec/containers/0/livenessProbe
```

Use when strategic merge fails you:

- surgical list operations (append/insert/remove **by index**),
- removing a field entirely (`op: remove`),
- keys containing `/` — escape as `~1`:
  `path: /metadata/annotations/nginx.ingress.kubernetes.io~1proxy-body-size`,
- patching CRDs (no strategic-merge schema → merge treats all lists as
  replace anyway, so 6902 is at least explicit about it).

Downside: index-based paths (`containers/0`) are brittle — they break silently
in intent (though loudly in review) when someone reorders the list. Prefer
name-keyed strategic merge whenever the list has a merge key.

## 3. Patch many objects at once (target selectors)

`target` accepts wildcards and label selectors — one patch, many objects:

```yaml
patches:
  - target:
      kind: Deployment
      labelSelector: "app.kubernetes.io/part-of=demo-app"
    patch: |-
      - op: add
        path: /spec/template/spec/priorityClassName
        value: business-critical
```

Also works with `name: ".*"` regex. This is how you enforce cluster policies
(priorityClass, imagePullSecrets, seccomp) across a whole repo without
touching every base.

## 4. Generators and rollout triggers

```yaml
configMapGenerator:
  - name: app-config
    literals: [LOG_LEVEL=info]
    # files: [config/app.properties]   # or whole files
secretGenerator:
  - name: app-tls
    files: [tls.crt, tls.key]
    type: kubernetes.io/tls
```

Generated names get a content hash suffix (`app-config-7f2kb9m4tc`) and all
references are rewritten automatically → config changes roll pods. To opt out
(external consumers need a stable name):

```yaml
generatorOptions:
  disableNameSuffixHash: true   # you now own the rollout problem yourself
```

## 5. Debugging patches

```bash
kubectl kustomize overlays/prod                      # render and eyeball
kubectl kustomize overlays/prod | grep -A5 resources # target one section
kubectl diff -k overlays/prod                        # against live cluster
```

If a patch "does nothing": 90% of the time the `target`/metadata doesn't match
(wrong name after a namePrefix, wrong group/version) — kustomize does not
error on a patch that matches zero objects unless you add
`options: {allowNameChange: true}`-style strictness via
`patches[].target` being explicit. Render and grep, don't guess.
