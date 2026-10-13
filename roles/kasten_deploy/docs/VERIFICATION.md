# Phase 0: verification and captures

Target version: Kasten 9.0.7 (docs.kasten.io/9.0.7, which resolves to `latest`).

Lab: OpenShift 4.20.30, single node (Kubernetes v1.33.13). Kasten 9.0.7 installed through the OLM operator
(`k10-kasten-operator-rhmp.v9.0.7`, channel `stable`, CatalogSource `certified-operators`, InstallPlan Automatic).
Control node tooling: `oc` 4.21.10, `helm` v4.2.3. Python venv `.venv` (git-ignored) with ansible-core 2.21.5,
ansible-lint 26.10.0, yamllint 1.38.0 and `kubernetes.core` 6.6.0 (system Python 3.9 is too old for recent ansible-core).
Run `ansible` with `< /dev/null` in non-interactive shells, otherwise it fails on non-blocking stdio.

All lab captures were read-only. Test objects (`phase0-*` policies) were created and deleted; the policy list
returned to its initial state. Export `receiveString` values are redacted. No Secret was captured.

Legend: [capture] observed on the lab, [doc] read in the 9.0.7 documentation, [unverified] still open.

## 1. Verified points

| # | Point | Source | Status | Decision |
|---|---|---|---|---|
| 1 | Operator package name | `captures/packagemanifest-k10.json` | [capture] | `k10-kasten-operator-rhmp`. It appears in two catalogs (`certified-operators`, `redhat-marketplace`), so `kasten_operator_catalog_source` is mandatory. Channel `stable`, current CSV v9.0.7. |
| 1b | Package `k10-kasten-operator-term-rhmp` | absent on lab; not cited in the 9.0.7 offline operator page | [unverified] | The 9.0.7 offline page only names `k10-kasten-operator-rhmp`. Preflight accepts both names, the package-to-edition mapping goes to `OPEN_ITEMS.md`. |
| 2 | CR `apiVersion`, `kind`, `spec` | `captures/alm-examples.json`, `cr-k10-spec.json` | [capture] | `apik10.kasten.io/v1alpha1`, kind `K10`, CRD `k10s.apik10.kasten.io`. The `spec` mirrors Helm values (`global.persistence`, `metering.mode`, `route`, `auth`, `limiter`). The live CR has `metering.mode: airgap`. |
| 2b | Operator reconciliation | CR annotations | [capture] | `helm.sdk.operatorframework.io/reconcile-period: 2m`: Helm-SDK operator, release object `sh.helm.release.v1.<cr-name>.*`. The release name is the CR `metadata.name`. |
| 2c | OLM install modes | CSV | [capture] | OwnNamespace and SingleNamespace only. OperatorGroup targets the Kasten namespace. |
| 3 | Reports policy | `captures/policy-k10-system-reports-policy.json` | [capture] | Name `k10-system-reports-policy`, action `report` with `reportParameters.statsIntervalDays`, `frequency: @daily`, `selector: {}`. |
| 3b | Reports retention | test `phase0-reports` | [capture] partial | `retention: {daily: 7}` is accepted. Effect on report pruning unverified. |
| 3c | Report licensing field | latest `Report` on lab | [capture] | `results.licensing.{status, nodeCount, nodeLimit, expiry, type}`. `status: Valid` observed. |
| 4 | "All applications" selector | CRD `oc explain` + tests | [capture] partial | CRD semantics: an empty selector matches all objects, a null selector matches none. `selector: {}` and `appNamespace In ["*"]` are both accepted. Runtime behaviour (system namespaces, Kasten namespace, new namespaces) is unverified. Proposal: `selector: {}`. |
| 5 | Cluster-scoped policy | `captures/policy-cluster-scoped-ressources.json`, `oc explain`, test | [capture] | Selector `appNamespace In [kasten-io-cluster]`. Filter fields under `backupParameters.filters`: `include|exclude` x `Resources|ClusterResources|ExtraResources`, each with `group`, `name`, `resource`, `version`, `matchLabels`, `matchExpressions`. `includeResources` with `{group, resource}` is accepted. Whether defaults are kept alongside an inclusion is unverified. Selector values accept globs (`NotIn ["default*"]` on the lab). By default all StorageClasses, CRDs, ClusterRoles and ClusterRoleBindings are captured [doc]. Restoring via import policy requires an explicit option [doc]. |
| 6 | S3 Profile | `captures/profile-storj*.json` | [capture] | `spec.type: Location`, `locationSpec.type: ObjectStore`, `objectStore.{objectStoreType: S3, endpoint, region, name, path, pathType: Directory, protectionPeriod, skipSSLVerify}`, `credential.{secretType: AwsAccessKey, secret.{apiVersion, kind, name, namespace}}`. Status at `status.validation`. |
| 6b | Immutability | `protectionPeriod: 168h0m0s` | [capture] | Go duration. `protection_days` maps to `<days*24>h0m0s`. |
| 6c | S3-compatible | custom endpoint with `objectStoreType: S3` | [capture] | Same `objectStoreType` as S3, only `endpoint` differs. `s3compatible` is an alias in the variable contract. |
| 6d | Credentials Secret keys | not read | [unverified] | Keys of the AwsAccessKey Secret are to be confirmed with a test Secret. |
| 7 | KDR policy (`quick_local`) | `captures/policy-k10-disaster-recovery-policy.json` | [capture] | Matches the prompt. The UI adds `subFrequency`; Kasten adds `createdBy` and `lastModifyHash`. CRD fields of `kdrSnapshotConfiguration`: `exportCatalogSnapshot`, `takeLocalCatalogSnapshot`. |
| 7b | KDR name constraint | tests on lab | [capture] | Any policy targeting the Kasten namespace under another name fails with `Please use K10 DR to backup the Kasten Namespace` (`validateDRPolicy`). The role must create exactly `k10-disaster-recovery-policy`. |
| 7c | Other KDR modes | none | [unverified] | `quick_exported`, `quick_none`, `legacy` cannot be tested without editing the real KDR policy. Rendered from the prompt templates only. |
| 7d | KDR Secret | `k10-dr-secret` | [capture] | Keys `key` and `source` observed. Value of `source` for `raw` not read. |
| 8 | License | [doc] Advanced Install Options | [doc] | Secret with the license text in field `license`. Multiple license Secrets may coexist and any valid one is used. Checked periodically, no restart. The docs show Helm labels on the example Secret but do not require them. A plain Secret is therefore the retained mechanism. Default name in the docs example: `k10-custom-license`. |
| 8b | License on lab | `k10-license`, `k10-trial-license` | [capture] | Helm-managed Secrets, key `license`. Manual-Secret test not run (user decision). |
| 9 | Multi-cluster flow | [doc] Getting Started | [doc] | Matches the prompt: three Helm values for the primary, `Bootstrap` alternative in `kasten-io-mc`, join token Secret of type `dist.kio.kasten.io/join-token` (docs use `generateName`), `mc-join` Secret and `mc-join-config` ConfigMap on the secondary, `mc-join-status` fields `status` (`Joining`, `Joined`, `Rejected`), `msg`, `timestamp`, `mc-join.hash`. `mc-cluster-info` created on success. Primary `Cluster` object carries `dist.kio.kasten.io/cluster-type: primary` (also observed on lab). |
| 9b | Join token naming | none | [unverified] | Deterministic name instead of `generateName` not tested (user decision). |
| 9c | Multi-cluster license | [doc] Concepts, License Management | [doc] | One license on the primary distributes one-hour leases to secondaries. Excess capacity of secondaries joins a global pool. Secondaries must reach the primary periodically. Requires an ingress on the primary. Retained behaviour: the role installs the license on the primary only and documents the lease model. Secondaries keep their own license logic untouched. |
| 10 | RBAC roles | [doc] RBAC + lab ClusterRoles | [doc] [capture] | Installed names are `<release>-admin`, `<release>-basic`, `<release>-config-view`, plus `k10-mc-admin` and `k10-virtualmachines-admin` (lab release is `k10`). `k10-basic-config` is only an example in the docs, it is NOT installed. The role resolves each referenced ClusterRole via `k8s_info` and fails if missing. `k10-basic` needs a RoleBinding per namespace, `k10-config-view` needs a ClusterRoleBinding. Kasten creates `kasten-io-k10-k10-admin` and `...-k10-mc-admin` ClusterRoleBindings and `...-k10-ns-admin` RoleBinding. |
| 11 | Air-gap Helm values | [doc] Air-Gapped Install | [doc] | `global.airgapped.repository`, `metering.mode=airgap`, `secrets.dockerConfig` (base64 of config.json), `global.imagePullSecret` (chart creates Secret `k10-ecr` from `secrets.dockerConfig`). Missing `metering.mode=airgap` offline: "Unable to validate license". Kubernetes 1.35+: protected namespaces need the pull Secret copied and attached to the `default` ServiceAccount (Kanister blueprints, unattached volumes, hooks creating pods). Documented in the README, not automated. |
| 12 | Free license | [doc] | [doc] | Embedded Free license: 50 nodes for 30 days then 5 nodes. Support window of about 6 months requires regular updates. |
| 13 | Dashboard URL | route on lab | [capture] | Route `k10-route` host `k10-route-kasten-io.apps.<lab-domain>`, path `/k10/`. The multicluster ingress URL is `<URL>/<release>` (here `/k10`). |
| 14 | Mirroring on lab | `oc get imagedigestmirrorset,imagecontentsourcepolicy` | [capture] | None: the lab is connected, real air-gap is not testable. Only the value logic is. |

## 2. Test results on the lab

Objects named `phase0-*` in the Kasten namespace, all deleted afterwards.

| Test | Result |
|---|---|
| Policy `selector: {}` | Accepted, `Success` |
| Policy `appNamespace In ["*"]` | Accepted, `Success` |
| Cluster-scoped policy with `includeResources` | Accepted, `Success`; `name` normalised to `""` |
| Reports policy with `retention.daily` | Accepted, `Success` |
| KDR policies (`exported`, `none`, `legacy`) under a test name | `Failed`: reserved name required |
| Manual license Secret | Not run (user decision) |
| Join token with fixed name | Not run (user decision) |

`validation: Success` proves schema acceptance, not runtime behaviour.

## 3. Deviations from the prompt

1. `k10-basic-config` is not an installed ClusterRole. It is removed from the valid `role` choices as a built-in. Any ClusterRole name is accepted if it exists (the role never creates ClusterRoles in v1).
2. The package `k10-kasten-operator-rhmp` exists in two catalogs; the catalog source is mandatory.
3. S3 and S3-compatible share `objectStoreType: S3`.
4. The KDR policy name is enforced by Kasten.
5. The lab already runs Kasten (operator). Phase 3 cannot start from a clean install and Phase 7 cannot join a real secondary: both are covered by render and check-mode tests only, marked unverified (user decision).
6. The lab is connected: real air-gap is not testable.
7. Language: code, variables, comments, README and docs are all in English (user decision, supersedes prompt rule 8).

## 4. Still open (goes to OPEN_ITEMS.md)

- Edition mapping for `k10-kasten-operator-term-rhmp`.
- Runtime behaviour of `selector: {}` versus `*` for all applications.
- Cluster-scoped inclusion: defaults kept or not.
- KDR modes `quick_exported`, `quick_none`, `legacy` against a live cluster.
- KDR policy update (no UI disable/enable) when the mode changes.
- Reports `retention` effect.
- Credentials Secret keys for AwsAccessKey, other profile types.
- Manual license Secret behaviour on a live cluster; deterministic join token Secret name.
- Helm `check_mode` diff behaviour.
- CRDs of `config.kio.kasten.io`: shipped by chart or created at runtime (present on the lab; fresh install not observed).
- Operator-mode equivalents of Helm multi-cluster and KDR values (`multicluster.primary.*`, `kastenDisasterRecovery.quickMode.enabled`).
