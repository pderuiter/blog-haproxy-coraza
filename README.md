# blog-haproxy-coraza

Companion code for the blog post
[Securing HAProxy with Coraza-SPOA: A Step-by-Step Guide](https://blog.bsdserver.nl/securing-haproxy-with-coraza-spoa-a-step-by-step-guide/).

It builds the post's setup on one host: HAProxy from the official HAProxy
packages, the Coraza-SPOA agent (v0.9.1, built from source with Go) with the
Coraza recommended configuration in blocking mode and the OWASP Core Rule Set
v4.25.0, and an HAProxy frontend that sends every request to Coraza through
SPOE and returns 403 when Coraza denies it.

## What is in here

| Path | Purpose |
|------|---------|
| `terraform/` | One Ubuntu 24.04 VM on Proxmox VE (bpg/proxmox provider, cloud-init), outputs an Ansible inventory |
| `ansible/site.yml` | One playbook for the whole setup |
| `ansible/roles/haproxy_install` | HAProxy from the vbernat PPA (Ubuntu) or haproxy.debian.net (Debian) |
| `ansible/roles/coraza_spoa` | Steps 1-7: build, user, directories, `config.yaml`, `coraza.conf`, CRS, logrotate and systemd unit |
| `ansible/roles/haproxy_coraza` | Steps 8-10: frontend and backends in `haproxy.cfg`, `/etc/haproxy/coraza.cfg`, `haproxy -c`, restart |

## Requirements

- Terraform >= 1.6 and a Proxmox VE API token (only if you want Terraform to create the VM)
- ansible-core >= 2.15 (tested with 2.21.6)
- Target host: Ubuntu 24.04, amd64, with Internet access (Go, coraza-spoa, CRS and
  the PPA are downloaded). The build needs about 2 GB of RAM.

## Quick start

```bash
# 1. Machine (optional; skip if you already have a host)
cd terraform
cp terraform.tfvars.example terraform.tfvars   # fill in your values
terraform init
terraform apply
terraform output -raw ansible_inventory > ../ansible/inventory.yml

# 2. HAProxy and Coraza-SPOA
cd ../ansible
cp inventory.example.yml inventory.yml         # if you skipped Terraform
ansible-galaxy collection install -r requirements.yml
ansible-playbook site.yml

# 3. Test
curl -s -o /dev/null -w '%{http_code}\n' "http://10.0.0.10/"                              # 200
curl -s -o /dev/null -w '%{http_code}\n' "http://10.0.0.10/?id=1%27%20OR%20%271%27=%271"   # 403
curl -s -o /dev/null -w '%{http_code}\n' "http://10.0.0.10/?file=../../etc/passwd"         # 403
```

Blocked requests are logged in `/var/log/coraza-spoa/server.log`.

## Variables

Set in `ansible/group_vars/all/vars.yml` (defaults in each role's `defaults/main.yml`):

| Variable | Default | Meaning |
|----------|---------|---------|
| `haproxy_install_version` | `3.3` | HAProxy branch. 3.3 is the newest the PPA builds for Ubuntu 24.04 (maintained until about Q1 2027); `3.2` is LTS |
| `coraza_spoa_version` | `v0.9.1` | coraza-spoa tag to build. Changing it rebuilds and reinstalls the agent |
| `coraza_spoa_go_version` | `1.27.4` | Go toolchain for the build (1.25 or newer) |
| `coraza_spoa_coraza_version` | `v3.8.1` | Coraza tag that `coraza.conf-recommended` is taken from |
| `coraza_spoa_crs_version` | `v4.25.0` | OWASP Core Rule Set tag |
| `coraza_spoa_rule_engine` | `On` | `SecRuleEngine`; `DetectionOnly` only logs |
| `coraza_spoa_bind` | `127.0.0.1:9000` | Agent address (must match `haproxy_coraza_agent_address`) |
| `coraza_spoa_keep_build_tools` | `false` | Keep Go and the source tree in `/opt/coraza-spoa-build` after the build |
| `haproxy_coraza_bind` | `*:80` | Frontend bind address |
| `haproxy_coraza_backend_servers` | `[]` | Real servers to protect (`name`, `address`). Empty means the post's static "Welcome!" backend |

The agent is built only when `/usr/local/share/coraza-spoa/VERSION` does not
match `coraza_spoa_version`, so a re-run does not need Go and changes nothing.

This setup has no secrets, so `group_vars/all/vault.yml.example` is empty.

## Differences from the post

- The Core Rule Set comes from the tag's release tarball instead of a
  `git clone --depth 1 --branch` of the same tag; the files are the same.
- The frontend and backends are added to the packaged `haproxy.cfg` as a marked
  block, validated with `haproxy -c` before it is written.

## Terraform variables

See `terraform/variables.tf` and `terraform/terraform.tfvars.example`. The VM is
defined in the `vms` map (name, IP in CIDR notation, Ansible groups, optional
cores/memory/disk); the playbook targets the `waf` group.

## Tested

`site.yml` was run against a fresh Ubuntu 24.04 systemd container (HAProxy
3.3.17, coraza-spoa v0.9.1 built with Go 1.27.4, CRS v4.25.0). A clean request
returned 200, SQL injection, XSS and path traversal returned 403. The second
run, and a run after restarting both services, reported `changed=0`; changing
the installed version marker triggered a rebuild, followed by `changed=0`.
ansible-lint (production profile), yamllint, `terraform validate` and tflint are
clean. The Terraform code has not been applied against a real Proxmox cluster.

## License

MIT
