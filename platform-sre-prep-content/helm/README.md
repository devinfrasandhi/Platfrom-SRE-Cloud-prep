# Helm — Interview Prep

## Refresher

### What it is
Helm is the package manager for Kubernetes. A **chart** is a bundle of templated Kubernetes manifests plus metadata. A **release** is a chart installed into a cluster with a specific set of values. Helm 3 stores release state as **Secrets in the release namespace** (no more Tiller — that was Helm 2).

### Chart anatomy
```
mychart/
├── Chart.yaml          # name, version, appVersion, dependencies
├── values.yaml         # default configuration values
├── charts/             # dependency charts (vendored)
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── _helpers.tpl    # named template definitions (partials)
│   ├── NOTES.txt       # post-install message
│   └── tests/          # helm test hooks
└── .helmignore
```

**Chart.yaml versioning** — `version` is the chart's own SemVer (bump when the chart changes); `appVersion` is the version of the app it deploys (informational).

### Templating essentials
Helm uses Go templates + Sprig functions:
- `{{ .Values.image.tag }}` — from values.yaml / `--set` / `-f` overrides
- `{{ .Release.Name }}`, `{{ .Release.Namespace }}` — release context
- `{{ .Chart.Name }}`, `{{ .Chart.Version }}` — chart metadata
- `{{ include "mychart.labels" . | nindent 4 }}` — call a named template from _helpers.tpl
- Control flow: `{{- if .Values.ingress.enabled }}` ... `{{- end }}`
- Loops: `{{- range .Values.env }}` ... `{{- end }}`
- `required "message" .Values.foo` — fail render if a value is missing
- `toYaml` / `nindent` — render nested structures with correct indentation

Values precedence (lowest → highest): chart `values.yaml` → parent chart overrides → `-f custom.yaml` files (later files win) → `--set` flags.

### Core commands
```
helm install <release> <chart> -f values-prod.yaml
helm upgrade --install <release> <chart>     # idempotent deploy
helm rollback <release> <revision>
helm history <release>
helm template <chart>                        # render locally, no cluster
helm lint <chart>
helm diff upgrade <release> <chart>          # (plugin) preview changes
helm dependency update
helm uninstall <release>
```

### Hooks and tests
Annotations like `helm.sh/hook: pre-install,pre-upgrade` run Jobs at lifecycle points (classic use: DB migrations). Hook weights order execution; hook delete policies clean them up. `helm test` runs pods annotated as test hooks to smoke-check a release.

### Helm + GitOps
With Argo CD or Flux, you typically stop running `helm install` by hand. Argo CD renders charts with `helm template` and applies the output (so Helm hooks map onto Argo sync waves/phases); Flux's HelmController does real Helm releases via `HelmRelease` CRs. Know the distinction — it comes up.

---

## Interview Questions

### Warm-up tier

**Q: Helm 2 vs Helm 3 — what changed and why does it matter?**
A: Helm 3 removed Tiller, the in-cluster server component that held broad cluster permissions and was a security nightmare. Helm 3 is client-only: it uses your kubeconfig credentials, so RBAC applies per-user. Release info moved to Secrets in the release's namespace, releases became namespace-scoped, and `helm install` now requires a release name (or `--generate-name`).

**Q: What's the difference between `helm template` and `helm install --dry-run`?**
A: `helm template` renders fully offline — no cluster contact, so functions like `lookup` return empty and no validation against the API server happens. `--dry-run` contacts the cluster: it validates rendered manifests against the API and can use cluster state. Use `template` in CI where there's no cluster access; use `--dry-run` (or better, `helm diff`) for pre-deploy checks against a real cluster.

**Q: How does `helm rollback` work under the hood?**
A: Every release revision is stored (as a Secret) containing the rendered manifests and values. Rollback creates a *new* revision whose content matches the target old revision, then applies it. Caveats: it doesn't roll back CRDs (Helm barely manages CRD lifecycle at all), it won't restore resources deleted out-of-band unless they're in the manifest, and hooks may re-run depending on annotations.

### Complex tier

**Q1: `helm upgrade` failed midway and the release is stuck in `pending-upgrade`. Production is degraded. Go.**
A: Immediate triage: `helm history <release>` to see revision states, `kubectl get pods` to assess actual damage. A stuck `pending-upgrade` usually means the Helm process died mid-flight (CI job killed, network drop) — the release lock is effectively the pending record itself. Options: (1) `helm rollback <release> <last-good-revision>` — usually the right move and clears the stuck state. (2) If rollback also fails, delete the pending release Secret (`kubectl delete secret sh.helm.release.v1.<release>.v<N>`) to unstick bookkeeping, then rollback/upgrade cleanly. (3) `--wait` + `--atomic` on future upgrades makes Helm auto-rollback on failure — but know that `--wait` blocks until resources are Ready, and with a bad readiness probe that means timeout, so pair with sane `--timeout`. Postmortem angle: why did a bad chart reach prod? Add `helm diff` + `helm template | kubeconform` validation in CI, and canary/staged rollout.

**Q2: You need one chart to deploy the same app to dev/staging/prod with different resources, replicas, ingress hosts, and some prod-only components. Structure it.**
A: One chart, environment values files: `values.yaml` holds safe defaults; `values-dev.yaml`, `values-prod.yaml` hold overrides; deploy with `-f values.yaml -f values-prod.yaml`. Prod-only components get feature flags: `{{- if .Values.pdb.enabled }}` around a PodDisruptionBudget, etc. Guard required per-env values with `required`. Anti-patterns to call out: separate charts per environment (drift guaranteed), templating logic that branches on a literal env name (`{{ if eq .Values.env "prod" }}` — prefer capability flags over env-name checks so behavior is explicit in values). In a GitOps setup this maps to one Application/HelmRelease per env pointing at the same chart with different values files — the Git diff between env values files *is* your env diff, which is the audit story interviewers like.

**Q3: Explain how you'd manage a chart's dependency on another chart (say, your app plus Redis), and the pitfalls.**
A: Declare in Chart.yaml:
```yaml
dependencies:
  - name: redis
    version: "18.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```
`helm dependency update` pulls it into `charts/` and writes `Chart.lock`. Configure the subchart via a top-level `redis:` block in values; expose parent values to subcharts via `global:`. Pitfalls: (1) subchart values are namespaced by chart name, and you can't template values passed to subcharts; (2) version-pin and commit `Chart.lock` or builds aren't reproducible; (3) for real production data stores, a chart-managed Redis is often the wrong call versus an operator or managed service — saying that shows judgment; (4) `condition` flags let one chart serve both "bundled Redis" for dev and "external Redis" for prod.

**Q4: A teammate's chart works with `helm install` but breaks under Argo CD. What are the likely causes?**
A: This is a great question for GitOps shops. Likely culprits: (1) **Hooks** — Argo CD translates Helm hooks into its own sync phases; unsupported ones or `hook-delete-policy` mismatches change behavior. (2) **`lookup` function** — renders against the live cluster in real Helm, but Argo CD uses `helm template`, where `lookup` returns nothing, so any logic depending on it silently changes. (3) **Random/generated values** — `randAlphaNum` in a template produces a new value every render, so Argo CD sees perpetual drift and loops on sync. (4) **CRD handling** — Helm's `crds/` directory install-once semantics differ from Argo's apply model. (5) **Release metadata** — anything reading `.Release.Service` or expecting Helm release Secrets to exist (Argo doesn't create them). Fixes: make charts pure functions of their inputs — no lookups, no randomness (take them as required values), hooks replaced with sync-wave annotations where needed.

**Q5: How do you prevent a bad values change from taking down prod — what's your chart testing story?**
A: Layers: (1) `helm lint` and `helm template | kubeconform --strict` in CI for schema-valid rendering. (2) `values.schema.json` in the chart — JSON Schema that Helm enforces on install/upgrade, catching wrong types and missing required fields at the earliest point. (3) Unit tests on rendered output (helm-unittest plugin, or terratest golden files) asserting critical properties — resource limits present, image tag pinned, PDB exists for prod flag. (4) `helm diff upgrade` output in the PR so a human sees the manifest-level change, not just the values change. (5) `helm test` smoke hooks post-deploy, and progressive delivery (Argo Rollouts / Flagger) so even a validated-but-wrong change gets caught by canary metrics before full rollout. Naming that last connection — deploy tooling backstopped by observability — is exactly what SRE interviewers want to hear.

**Q6: Chart needs to render a checksum of a ConfigMap into pod annotations. Why is this a common pattern, and write it.**
A: Kubernetes doesn't restart pods when a ConfigMap they mount changes. The pattern hashes the config into the pod template so any config change alters the template, triggering a rolling update:
```yaml
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```
Follow-ups to expect: alternatives (Reloader/Stakater operator, projecting config as env from a versioned Secret, or immutable ConfigMaps with hashed names), and the trade-off that checksum-restart couples config changes to rollouts, which is usually desired but can surprise during incident config tweaks.

---
