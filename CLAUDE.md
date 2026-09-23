# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

Stratio's fork of [kubernetes-sigs/kubespray](https://github.com/kubernetes-sigs/kubespray) — an Ansible
collection of playbooks/roles that deploys a production Kubernetes cluster on existing machines. There is
no application code to build; the "artifact" is the Ansible tree itself, consumed downstream by KEOS.

Git remotes in this checkout: `or` = `Stratio/kubespray` (the fork, treat as origin), `spray` =
upstream `kubernetes-sigs/kubespray`, `vik` = personal fork. Work happens on `release-2.X` branches that
track the corresponding upstream release branch; the current one is `release-2.32`. Fork-specific commits
are prefixed with a Jira key, e.g. `[PLT-4861] …` (older ones use `[EOS-…]`). `git log spray/release-2.32..HEAD`
shows exactly what the fork carries on top of upstream — keep that delta small and rebase-friendly.

`release-2.26-1.35` is the previous base, kept alive only while KEOS 1.4.x consumes tag `v2.26.0-1.35-6`.
Do not add work there.

**Tags.** keos-installer pins the fork by git tag in its `ansible-galaxy-requirements.txt`, so the tag is
the contract. Scheme: `v<galaxy.yml version>-<supported k8s minor>-<respin>`, e.g. `v2.32.1-1.36-1`. The
first component tracks `galaxy.yml`, which on this base is `2.32.1` (not `2.32.0`).

## Environment setup

```ShellSession
python3 -m venv venv && source venv/bin/activate
pip install -r tests/requirements.txt      # pulls in requirements.txt (ansible 12 / ansible-core 2.19, pinned)
pre-commit install
```

Name the venv `venv/` — that exact path is in `.gitignore` and in `exclude_paths` of `.ansible-lint`. A venv
under any other name inside the repo gets linted, and ansible-lint will drown you in violations from
site-packages.

There is no `tests/requirements.yml` on this base; collection dependencies live in `dependencies:` of
`galaxy.yml` and are resolved when the collection is built.

Ansible and all lint tooling are version-pinned (`requirements.txt`, `tests/requirements.txt`,
`.pre-commit-config.yaml`). Do not bump them incidentally — CI runs the pinned versions.

**ansible-core 2.19 is a behavioral change, not just a version bump.** Conditionals must evaluate to real
booleans: `when: groups['broken_etcd']` (a bare list) is now an error, write `| length > 0`. And
`groups['x']` on a group that does not exist raises `object of type 'dict' has no attribute 'x'` — guard
with `'x' in groups` first; Ansible evaluates a list of `when:` conditions left to right and short-circuits.

## Commands

Lint / validate (this is the main "test" loop for most changes):

```ShellSession
pre-commit run -a                                            # the default-stage hooks
pre-commit run -a --hook-stage manual                        # ansible-lint, check-checksums-sorted, collection-build-install
pre-commit run yamllint --all-files                          # single hook; hook ids are in .pre-commit-config.yaml
pre-commit run ansible-lint --all-files --hook-stage manual
```

**Three hooks are `stages: [manual]` and do not run with a plain `pre-commit run -a`**: `ansible-lint`,
`check-checksums-sorted` and `collection-build-install`. They are the three that matter most. Run them
explicitly or you will think a broken change is clean.

There is no longer an `ansible-syntax-check` hook, so syntax-check by hand — especially
`recover-control-plane.yml`, which nothing else covers:

```ShellSession
for p in cluster upgrade-cluster reset recover-control-plane scale remove-node; do
  ansible-playbook -i inventory/local/hosts.ini --syntax-check "$p.yml"
done
```

Run a playbook against an inventory:

```ShellSession
ansible-playbook -i inventory/mycluster/hosts.yaml --become --become-user=root cluster.yml
ansible-playbook -i inventory/mycluster/hosts.yaml --become --become-user=root cluster.yml --tags download,preinstall
```

`--become` is mandatory for every top-level playbook. `inventory/local/hosts.ini` is a single-node
`ansible_connection=local` inventory useful for smoke-testing; it is also the cheapest way to check that a
variable still resolves after a refactor:

```ShellSession
ansible-playbook -i inventory/local/hosts.ini /tmp/probe.yml   # a play with roles: [kubespray_defaults] and a debug task
```

Molecule (role-level tests; requires vagrant + libvirt). `tests/scripts/molecule_run.sh` no longer exists —
run scenarios from the role directory:

```ShellSession
cd roles/container-engine/containerd && molecule test
cd roles/container-engine/containerd && molecule create && molecule converge && molecule login  # iterate/debug
```

Roles with molecule scenarios: `adduser`, `bastion-ssh-config`, `bootstrap_os`, and
`container-engine/{containerd,cri-dockerd,cri-o,gvisor,kata-containers,youki}` plus a shared
`container-engine` scenario. Everything else is only covered by the end-to-end CI jobs.

End-to-end tests are GitLab CI only (`.gitlab-ci/*.yml`): they provision real VMs on Equinix Metal /
OpenStack / KubeVirt, driven by `tests/scripts/testcases_run.sh` with a scenario file from
`tests/files/<CI_JOB_NAME>.yml`. They are not runnable locally without cloud credentials; use Vagrant
(`vagrant up`, config in `Vagrantfile`) for local end-to-end work instead.

## Architecture

**Playbook layering.** Top-level `*.yml` (`cluster.yml`, `upgrade-cluster.yml`, `scale.yml`, `reset.yml`,
`remove-node.yml`, `recover-control-plane.yml`, plus snake_case aliases) are one-line `import_playbook`
shims into `playbooks/`. The real orchestration is `playbooks/cluster.yml`: boilerplate → facts → etcd →
node → control-plane → kubeadm + CNI → calico_rr → windows → apps → late resolv.conf. Each play targets an
inventory group (`k8s_cluster`, `etcd`, `kube_control_plane`, `kube_node`, `calico_rr`, `bastion`) and every
play re-applies the `kubespray_defaults` role first. When adding a role, add it to the right play in
`playbooks/cluster.yml` **and** give it a tag — tags are the supported way to run a slice of the deploy.

**`roles/kubespray_defaults` is the variable spine** (snake_case; the old `kubespray-defaults` directory
survives only as a deprecation shim that prints a failure banner). It has no tasks worth speaking of; it
exists so that its defaults and vars are re-evaluated at the start of every play. Layout:

- `defaults/main/main.yml` — cluster-shape defaults (`kube_proxy_mode`, kubeadm phases, `kube_cert_dir`,
  `first_kube_control_plane`, …). Overridable from inventory.
- `defaults/main/download.yml` — image repos, image tags, download URLs, plus the big `downloads:` dict
  consumed by the `download` role. Overridable.
- `vars/main/checksums.yml` — per-arch (`amd64`/`arm64`/`ppc64le`, sometimes `arm`) SHA256 maps.
  **In `vars/`, so inventory `group_vars` cannot override it.**
- `vars/main/main.yml` — internal version machinery (`kube_major_version`, `pod_infra_supported_versions`,
  `etcd_supported_versions`, `sysctl_minimum_values`, …).

Any variable a user should be able to override belongs in `kubespray_defaults`' `defaults/`, not in the
consuming role's `defaults/`, otherwise role defaults silently shadow nothing and precedence surprises appear.

**The `download` role** iterates `downloads | combine(kubeadm_images)` and, per entry, either pulls a
container image or fetches a file and verifies its checksum. Each entry declares `enabled`, `container`/`file`,
`version`, `url`/`repo`+`tag`, `sha256`, and `groups` (which inventory groups need it). Air-gapped and
`download_run_once`/`download_localhost` caching modes all flow through this one dict — adding a new
binary or image means adding an entry there, a version var, and checksums, not writing a bespoke `get_url`.

**Where cluster behavior lives.** `roles/kubernetes/{preinstall,node,control-plane,kubeadm,kubeadm_common,
client,node-label,node-taint}` build the cluster itself; `roles/kubernetes-apps/*` are addons applied from
`kube_control_plane[0]`; `roles/network_plugin/*` writes CNI config on nodes while
`roles/kubernetes-apps/network_plugin/*` applies the corresponding manifests; `roles/container-engine/*`
installs containerd/cri-o/docker/runc/etc. `roles/etcd` (with `roles/etcd_defaults`) supports host-binary,
container and kubeadm etcd. `roles/validate_inventory` runs inventory assertions before anything else and
**aborts the run** if a variable upstream has removed is still set. `contrib/` (terraform,
inventory_builder, offline, dind, …) is out-of-band tooling, not part of a deploy.

**User configuration surface.** `inventory/sample/group_vars/` is the template users copy:
`all/all.yml` plus per-cloud files, and `k8s_cluster/k8s-cluster.yml`, `addons.yml`, `k8s-net-<plugin>.yml`.
Changing a default that users are expected to tune usually means touching both `kubespray_defaults` and the
matching sample file.

## Version bumps (the most common change in this fork)

**`vars/main/checksums.yml` is the single source of truth for versions.** Most version variables are now
derived from it rather than written down:

```yaml
kube_version:              "{{ (kubelet_checksums['amd64'] | dict2items)[0].key }}"    # newest entry
kube_version_min_required: "{{ (kubelet_checksums['amd64'] | dict2items)[-1].key }}"   # oldest entry
runc_version:              "{{ (runc_checksums['amd64'] | dict2items)[0].key }}"
crictl_version:            "{{ (crictl_checksums['amd64'].keys() | select('version', kube_major_next_version, '<'))[0] }}"
```

Consequences you cannot avoid:

- **The file must stay sorted descending**, per arch. The `check-checksums-sorted` hook
  (`scripts/assert-sorted-checksums.yml`) enforces it. Prepending an entry moves the default; appending one
  lowers `kube_version_min_required`.
- **Keys have no `v` prefix** (`1.36.4`, not `v1.36.4`) and **values carry the algorithm**
  (`sha256:<hex>`, not bare hex). This changed with the 2.32 base and it breaks any downstream code that
  indexes these maps with a `v`-prefixed version.
- You can no longer add a newer kubelet checksum without moving the default `kube_version`. The old fork
  habit of "add checksums for a new minor, leave `kube_version` alone" does not apply here. KEOS pins
  `kube_version` from its own group_vars regardless.

Generate hashes rather than hand-writing them. `scripts/download_hash.py` and `download_hash.sh` are gone;
the replacement is a proper package:

```ShellSession
pip install -e scripts/component_hash_update
update-hashes                               # all targets
update-hashes kubelet kubectl kubeadm       # a subset; names are the keys in components.py
```

`update-hashes` only discovers **new patch versions** of what is already in the file. To add a new minor,
hand-edit `vars/main/checksums.yml` first — insert the lowest relevant patch with a hash of `0` under every
arch already present for that target — then run the script to fill in the real hashes. Some lookups go
through the GitHub GraphQL API, so export a `GITHUB_TOKEN` or you will be rate-limited.

Gotchas:

- There is no `check-readme-versions` hook any more; the README component table is generated from
  `scripts/readme_versions.md.j2`.
- New Kubernetes minor support means adding checksums for `kubelet`/`kubectl`/`kubeadm`/`crictl`/`cri-o`
  across **every arch already present** in `checksums.yml`; a partially-filled arch fails at deploy time,
  not lint time. It also means adding the matching keys to `pod_infra_supported_versions` and
  `etcd_supported_versions` in `vars/main/main.yml` — a missing key there is an undefined-key failure at
  deploy time too.
- **CoreDNS is capped by kubeadm, not by kubespray.** kubeadm vendors `corefile-migration`, which knows a
  fixed set of CoreDNS versions; picking a newer one makes `kubeadm upgrade` fail on the last control-plane
  with `start version 'x.y.z' not supported`. Check
  `github.com/coredns/corefile-migration` in the target Kubernetes tag's `go.mod`, then that release's
  `migration/versions.go`. For reference: kubeadm 1.36 → corefile-migration 1.0.31 → CoreDNS ≤ 1.14.2;
  kubeadm 1.37 → 1.0.34 → CoreDNS ≤ 1.14.6.

## Conventions and lint gotchas

- `.ansible-lint` `skip_list` is closed: do not add rules to it. Silence a specific violation with an inline
  `# noqa: <rule>` plus a comment explaining why, or add a path/rule pair to `.ansible-lint-ignore`.
- Roles here intentionally do not use FQCNs, use camelCase vars matching k8s API fields, and put Jinja in
  task names — those rules are already skipped globally.
- yamllint runs `--strict` and **forbids octal values, implicit and explicit** — file modes must be quoted
  strings (`mode: "0644"`). ansible-lint v25 also rejects `yes`/`no` for booleans; write `true`/`false`.
- markdownlint (`.md_style.rb`/`.mdlrc`), shellcheck (`--severity=error`, `*.sh` only), misspell, `astl`,
  and a Jinja syntax check (`tests/scripts/check-templates.py`) all run in pre-commit.
- `docs/_sidebar.md` is generated by `scripts/gen_docs_sidebar.sh` via pre-commit — edit `docs/`, not the sidebar.
- `propagate-ansible-variables` (`scripts/propagate_ansible_variables.yml`) regenerates `Dockerfile` and
  `pipeline.Dockerfile` from templates. If it rewrites them, commit the result alongside your change.
- **The fork removes upstream's `check-galaxy-version` hook.** `scripts/galaxy_version.py` assumes plain
  `vX.Y.Z` tags and raises `ValueError` on Stratio's `vX.Y.Z-1.3N-M` scheme, which would block every
  `pre-commit run -a`. The script is left on disk, just unwired. Do not add the hook back.
