# kasten_deploy

Ansible role for offline deployment of Veeam Kasten (Helm or OpenShift OLM operator) and its post-install
configuration: license, multi-cluster, RBAC, location profiles, Kasten Disaster Recovery (KDR), Reports and policies.

Target Kasten version: 9.0.7 (documentation pinned on `docs.kasten.io/9.0.7`).

## Status

The role is delivered by phases. Each phase ends with a review before the next one starts.

| Phase | Content | State |
|---|---|---|
| 0 | Verification and lab captures | Done, see [VERIFICATION.md](roles/kasten_deploy/docs/VERIFICATION.md) |
| 1 | Skeleton, defaults, argument specs, preflight, tags | Done and tested |
| 2 | Helm air-gap install and license | Implemented. Value rendering and license check-mode tested. Real Helm install and idempotence not tested yet (needs the chart file) |
| 3 | OLM operator install | Not started (placeholder) |
| 4 | RBAC and profiles | Not started (placeholder) |
| 5 | KDR and Reports | Not started (placeholder) |
| 6 | Policies | Not started (placeholder) |
| 7 | Multi-cluster | Not started (placeholder) |
| 8 | Tests, final documentation | Not started |

Everything still unverified is listed in [OPEN_ITEMS.md](roles/kasten_deploy/docs/OPEN_ITEMS.md).
Tasks of phases 3 to 8 are `meta: noop` placeholders: enabling their variables currently does nothing.

## Out of scope (v1)

- Mirroring of images: the private registry must already be populated, the role only consumes it.
- Creation of the `CatalogSource` and of the `ImageDigestMirrorSet` / `ImageContentSourcePolicy` on OpenShift (cluster prerequisites).
- Dashboard authentication (OIDC, OpenShift OAuth, LDAP): possible through `kasten_values` passthrough, without dedicated logic.
- VolumeSnapshotClass, Kanister blueprints, transform sets, policy presets.
- KDR restore (`KastenDRReview`, `KastenDRRestore`), profile deletion, custom ClusterRoles.

## Prerequisites

- Control node: Python 3.11 or later, ansible-core 2.21 or later (tested with 2.21.5), `kubernetes.core` 6.6.0 or later,
  the Python `kubernetes` library, and the `helm` binary (tested with Helm 4.2.3). Nothing is downloaded by the role.
  The Python interpreter used by Ansible must have the `kubernetes` library: set `ansible_python_interpreter` accordingly
  (the example inventory uses `ansible_playbook_python`).
- The role never calls `kubectl` or `oc`. All cluster access goes through `kubernetes.core` modules.
- Connection: `kasten_kubeconfig` / `kasten_kube_context`, or the `K8S_AUTH_*` environment variables.
- Private registry already populated with the Kasten images (`k10tools image` can copy them) and with the few upstream images
  Kasten uses (for example Prometheus: without it, metrics are omitted from Reports).
- Local Kasten chart (`k10-<version>.tgz`) on the control node for the Helm method.
- OpenShift operator method: a `CatalogSource` carrying `k10-kasten-operator-rhmp`, and image mirroring, both prepared outside the role.
  The package exists in several catalogs, so `kasten_operator_catalog_source` is mandatory.
- Kubernetes 1.35 or later with a private registry: the pull Secret must be copied into each protected namespace and attached to the
  `default` ServiceAccount (Kanister blueprints, unattached volumes, hooks that create pods). This is documented by Kasten and not automated.

```bash
python3.14 -m venv .venv
.venv/bin/pip install -r requirements.txt ansible-lint yamllint
.venv/bin/ansible-galaxy collection install -r requirements.yml
```

## Usage

```bash
export KUBECONFIG=/path/to/kubeconfig
ansible-playbook playbooks/site.yml
```

Example variables are in [inventory/group_vars/kasten_clusters.yml](inventory/group_vars/kasten_clusters.yml). Secrets (license, KDR passphrase,
registry config, object store credentials) belong in an `ansible-vault` file and are never logged (`no_log`).

Tags: `preflight` (always runs), `install`, `license`, `multicluster`, `rbac`, `profiles`, `dr`, `reports`, `policies`, `validate`.

### Helm, offline

```yaml
kasten_version: "9.0.7"
kasten_install_method: helm
kasten_airgap_repository: registry.example.local:5000/kasten
kasten_helm_chart_path: /opt/kasten/k10-9.0.7.tgz
kasten_image_pull_secret_name: k10-ecr
kasten_docker_config_json_b64: "{{ vault_docker_config_b64 }}"
kasten_license_enabled: true
kasten_license_file: /opt/kasten/license.lic
```

## Values and precedence

`kasten_values` is deep merged with role values, then rendered as Helm values (or as the `spec` of the `K10` custom resource with the operator;
the lab confirms the `spec` mirrors the Helm values). From weakest to strongest:

1. Air-gap values of the role: `global.airgapped.repository`, `metering.mode=airgap`, and `global.imagePullSecret` / `secrets.dockerConfig` when set.
2. `kasten_values` provided by the user.
3. Multi-cluster primary values: `multicluster.primary.{create,name,ingressURL}`.
4. KDR mode value: `kastenDisasterRecovery.quickMode.enabled` (false for `legacy`, true for the Quick modes).

Without `metering.mode=airgap` in a disconnected cluster, Kasten reports "Unable to validate license".

## Variables

All variables use the `kasten_` prefix and are declared with types and descriptions in
[argument_specs.yml](roles/kasten_deploy/meta/argument_specs.yml). Defaults are in
[defaults/main.yml](roles/kasten_deploy/defaults/main.yml). Conditionally required variables are enforced by the preflight, which reports
all errors in a single message.

| Group | Variables |
|---|---|
| Platform | `kasten_platform` (auto, openshift, kubernetes), `kasten_install_method` (helm, operator), `kasten_version`, `kasten_namespace`, `kasten_release_name`, `kasten_kubeconfig`, `kasten_kube_context` |
| Offline | `kasten_airgap_repository`, `kasten_image_pull_secret_name`, `kasten_docker_config_json_b64`, `kasten_helm_chart_path`, `kasten_helm_wait_timeout`, `kasten_values` |
| Operator | `kasten_operator_package`, `kasten_operator_catalog_source`, `kasten_operator_catalog_namespace`, `kasten_operator_channel`, `kasten_operator_starting_csv`, `kasten_operator_install_plan_approval` |
| License | `kasten_license_enabled`, `kasten_license_text`, `kasten_license_file`, `kasten_license_secret_name` |
| Multi-cluster | `kasten_mc_role`, `kasten_mc_cluster_name`, `kasten_mc_primary_ingress_url`, `kasten_mc_primary_host`, `kasten_mc_secondary_cluster_ingress`, `kasten_mc_primary_ingress_override`, `kasten_mc_join_timeout_seconds`, `kasten_mc_delete_join_token_after_join` |
| RBAC | `kasten_rbac_bindings` |
| Profiles | `kasten_profiles` (types `s3` and `s3compatible`, which share `objectStoreType: S3`) |
| KDR | `kasten_dr_enabled`, `kasten_dr_mode`, `kasten_dr_profile`, `kasten_dr_passphrase_source`, `kasten_dr_passphrase`, `kasten_dr_run_once`, `kasten_dr_policy` |
| Reports and policies | `kasten_reports_enabled`, `kasten_reports_policy`, `kasten_reports_wait_first_report`, `kasten_policies`, `kasten_cluster_scoped_extra_includes` |

Internal variables set by the role (`set_fact`, `register`) use the `kasten_deploy_` prefix.

## Behaviour notes

- **License.** Kasten reads license Secrets dynamically from a field named `license`, several Secrets can coexist and any valid one counts.
  The role creates a plain Secret (not a Helm value) so the license stays out of the release values. Validation reads the latest `Report`
  (`results.licensing.status`) and needs Reports enabled; without it the role only warns. With multi-cluster, a license installed on the primary
  distributes one-hour leases to the secondaries, which must reach the primary periodically.
- **KDR.** The policy must be named `k10-disaster-recovery-policy`: Kasten rejects any other policy targeting its own namespace. Align the KDR
  frequency with the lowest RPO of the cluster. Keep the Cluster ID, the passphrase (or secret manager reference) and the KDR profile details, they
  are needed for any restore. Reports are not protected by KDR. Prefer an immutable profile for KDR.
- **Policy "all applications".** An export policy that targets all applications creates exports for every namespace: estimate storage volume and S3
  costs before production.
- **Helm check mode.** Behaviour of `kubernetes.core.helm` in check mode is not verified (see open items).
- **RBAC.** Kasten installs `<release>-admin`, `<release>-basic` and `<release>-config-view` ClusterRoles. `k10-basic-config` is only a documentation
  example and is not installed. The role never modifies the bindings created by Kasten (`k10:admins`).

## Kubernetes vanilla (RKE2 or equivalent)

The same code runs on vanilla Kubernetes: set `kasten_install_method: helm` (the operator method requires OpenShift and fails preflight otherwise).
Platform detection looks for the `config.openshift.io` API; set `kasten_platform: kubernetes` to skip it. On vanilla clusters manual cluster commands
use `kubectl`; on OpenShift use `oc`. This path has not been tested yet: no vanilla cluster was available.

## Tests

```bash
export KUBECONFIG=/path/to/kubeconfig VIRTUAL_ENV=$PWD/.venv
.venv/bin/yamllint .
.venv/bin/ansible-lint
.venv/bin/ansible-playbook playbooks/site.yml --syntax-check
.venv/bin/ansible-playbook tests/preflight.yml -i localhost,       # 30 preflight cases, needs a reachable cluster
.venv/bin/ansible-playbook tests/render_values.yml -i localhost,   # values rendering against tests/expected, no cluster
```

Expected results are written before the first run. When a result differs, the role is corrected, not the test.
`ansible-lint` uses the `production` profile; the only skipped rule is `var-naming[no-role-prefix]` because the public contract uses `kasten_`.
In non-interactive shells, run `ansible` with `< /dev/null`.

## Repository layout

```
roles/kasten_deploy/   defaults, meta (argument specs), tasks, templates, docs
  docs/VERIFICATION.md Phase 0 results and decisions
  docs/OPEN_ITEMS.md   everything still unverified or not implemented
  docs/captures/       objects captured on the lab (tokens and Secrets excluded)
playbooks/             site.yml (one cluster)
inventory/             example inventory and group_vars (placeholders only)
tests/                 preflight matrix, values rendering tests, expected outputs
```

## License

Apache-2.0, see [LICENSE](LICENSE).
