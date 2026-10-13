# Open items

Everything below is still unverified or not implemented. Source: `VERIFICATION.md` (Phase 0) and the phase reports.
Lab limits: single OpenShift 4.20.30 cluster, connected (no real air-gap), already running Kasten 9.0.7 through the operator.
No vanilla Kubernetes cluster and no second OpenShift cluster were available.

## Unverified Kasten behaviour

- Edition mapping of `k10-kasten-operator-term-rhmp`. The 9.0.7 offline operator page only names `k10-kasten-operator-rhmp`.
  Preflight accepts both package names until this is confirmed.
- Runtime behaviour of an "all applications" policy: `selector: {}` versus `appNamespace In ["*"]` (both accepted by the API),
  and whether system namespaces, the Kasten namespace and namespaces created later are included.
- Cluster-scoped policies: whether an `includeResources` filter replaces or complements the default resources, and whether
  an `actions.kio.kasten.io` / `defaults` inclusion keeps the defaults.
- KDR modes `quick_exported`, `quick_none` and `legacy` against a live cluster. A policy targeting the Kasten namespace is only
  accepted under the name `k10-disaster-recovery-policy`, so they cannot be tested without editing the real KDR policy.
- Whether updating the KDR policy is enough to change the catalog snapshot mode (the UI requires disabling then enabling KDR).
- Effect of `retention` on the Reports policy (accepted by the API, pruning not observed).
- Keys of the credentials Secret for `AwsAccessKey` profiles, and other profile types (Veeam Repository, Azure, GCP, ...).
- Behaviour of a manually created license Secret on a live cluster, with and without Helm labels. The documentation only requires
  the `license` field. Not tested on the lab to avoid touching the real license.
- Deterministic name for the multi-cluster join token Secret instead of `generateName`. Not tested on the lab to avoid touching the
  primary.
- Whether the `config.kio.kasten.io` CRDs are shipped by the chart or created at runtime (present on the lab, fresh install not observed).
- Operator-mode equivalents of the Helm values `multicluster.primary.*` and `kastenDisasterRecovery.quickMode.enabled`.
- Helm `check_mode` behaviour (diff plugin needed).

## Not implemented

- KDR passphrase sources `vault`, `aws` and `azure`: the variable is exposed, preflight fails with an explicit message.
- KDR restore (`KastenDRReview`, `KastenDRRestore`).
- Profile deletion, profile types other than `s3` and `s3compatible`.

## Test coverage gaps

- Phase 3 (operator): cannot start from a clean install on the lab. Render and check-mode only, plus read-only comparison with the live CR.
- Phase 7 (multi-cluster): no second cluster. Render and check-mode only.
- Real air-gap and vanilla Kubernetes: not tested. Only the Helm value logic is tested.
