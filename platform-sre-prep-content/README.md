# Platform / SRE / Cloud Interview Prep

Interview preparation notes organized by topic, for platform engineer, SRE, and cloud engineer roles.

## Topics

| Folder | Contents |
|---|---|
| [terraform/](terraform/) | Terraform refresher + interview Q&A (state, modules, CI/CD, state surgery) |
| [helm/](helm/) | Helm refresher + interview Q&A (charts, templating, GitOps, failure recovery) |

More topics coming: Kubernetes, Linux troubleshooting, networking, observability, system design, DSA.

## Cross-cutting: Terraform + Helm together

**Q: Should Terraform deploy Helm charts (helm_release resource)? Argue both sides.**
A: For: single tool, one apply provisions cluster + workloads, good for bootstrap (cluster + ingress controller + cert-manager in one shot). Against: Terraform's diff model fits infrastructure lifecycles (hours/days) poorly with application deploy cadence (many per day); helm_release diffs are opaque; state lock contention couples app deploys to infra pipelines; failed releases wedge Terraform state. Mature answer: Terraform provisions the platform (cluster, node pools, IAM/IRSA, DNS, maybe the GitOps controller itself), then hands off — Argo CD/Flux owns everything inside the cluster. The boundary is "Terraform installs Argo, Argo installs everything else" — the app-of-apps bootstrap pattern.

**Q: How do you pass values from Terraform-created infra (e.g., an RDS endpoint, IAM role ARN) into Helm-deployed apps cleanly?**
A: Options ranked: (1) Terraform writes to a well-known contract point — SSM Parameter Store/Secrets Manager — and workloads consume via External Secrets Operator; loosest coupling, no pipeline ordering. (2) Terraform outputs → CI injects as Helm values; simple but couples pipelines. (3) Terraform writes a ConfigMap via the kubernetes provider that charts reference. (4) IRSA-style patterns where the "value" is really an identity binding, handled by annotation conventions. Bonus points for mentioning that the contract (parameter naming convention) should be versioned and documented, because that seam is where platform teams and app teams meet.

---

## Quick self-test checklist

Can you, without notes:
- Explain what's in a state file and why it's sensitive?
- Recover from a lost/corrupted state file?
- Write a `for_each` over a map of objects and reference one instance's output?
- Explain `moved` blocks and when you'd use `import` blocks?
- Draw your ideal Terraform repo/state layout for 3 envs and 5 teams?
- Write a Helm named template and call it with correct indentation?
- Explain values precedence order exactly?
- Debug a `pending-upgrade` stuck release?
- Explain why `lookup` and `randAlphaNum` break under Argo CD?
- Argue the Terraform-vs-GitOps boundary for cluster workloads?

If yes to all — you're ready for the IaC rounds.
