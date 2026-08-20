# Azure Hub-Spoke DNS Automation

An Ansible Automation Platform (AAP) project for deploying an internal BIND DNS
server and configuring Azure VM clients in a Hub-Spoke topology.

## Layout

| Path | Purpose |
| --- | --- |
| `playbooks/` | AAP job-template entry points. |
| `roles/bind_dns/` | Idempotent BIND deployment and configuration role. |
| `inventories/production.azure_rm.yml` | Azure dynamic inventory source. |
| `group_vars/all.yml` | Non-secret DNS configuration. |
| `collections/requirements.yml` | Collection dependencies. |
| `execution-environment.yml` | AAP execution environment definition. |
| `.github/workflows/` | Pull-request quality gates. |

The root-level `tasks/`, `templates/`, and `defaults/` directories are retained
temporarily from the imported Galaxy role. The AAP playbooks use only
`roles/bind_dns`; remove the legacy content after testing the migrated role.

## Prerequisites

- Ubuntu or Debian DNS VMs. The initial `bind_dns` role deliberately supports
  these platforms only.
- Azure tags: `Role=dns_server` for BIND VMs and `Role=dns_client` for clients.
- NSG rules allowing TCP and UDP port 53 from client subnets to DNS servers.
- An AAP Machine credential for SSH and Azure Resource Manager credential for
  dynamic inventory.
- Reserved/static DNS-server private IPs. Update `group_vars/all.yml` before use.

## AAP setup

1. Create a Git project pointing to this repository and enable **Update revision on launch**.
2. Configure an Azure inventory source using `inventories/production.azure_rm.yml`.
3. Build/register an execution environment from `execution-environment.yml`.
4. Create Job Templates in order:
   - `00 - Pre-Check Audit` → `playbooks/pre_check.yml`
   - `01 - Deploy BIND DNS Server` → `playbooks/dns_server.yml`
   - `02 - Configure DNS Clients` → `playbooks/dns_clients.yml`
   - `03 - Post-Check Validation` → `playbooks/post_check.yml`
5. Link those templates in an AAP workflow; add a production approval after
   successful pre-checks.

## CI → AAP auto-sync and launch

The GitHub Actions workflow (`.github/workflows/ansible-lint.yml`) automatically
syncs the AAP project and launches the workflow template after lint passes on
a push to `main`.

### Required GitHub Secrets

Configure these in **Settings → Secrets and variables → Actions** on the repository:

| Secret | Description | Example |
| --- | --- | --- |
| `AAP_HOST` | AAP/AWX base URL (no trailing slash) | `https://aap.example.com` |
| `AAP_TOKEN` | AAP personal access token or OAuth2 token | `ey...` |
| `AAP_PROJECT_ID` | Numeric ID of the AAP Git project | `12` |
| `AAP_WORKFLOW_ID` | Numeric ID of the AAP workflow job template | `8` |

> **Tip:** Find the IDs from the AAP UI URL — e.g. `/#/projects/12/details`
> gives you `AAP_PROJECT_ID=12`.

## Security

- Never commit SSH keys, Azure credentials, vault passwords, or plaintext secrets.
- Store secrets in AAP credentials or an Ansible-Vault encrypted file.
- Do not permit recursive DNS from `any` or expose DNS publicly.
- Replace all sample domain/IP values before deployment.
- Use Azure VNet/NIC DNS configuration through IaC or Azure API. The client
  playbook configures `systemd-resolved` only on managed Linux VMs.

## Local validation

```bash
ansible-galaxy collection install -r collections/requirements.yml
ansible-lint playbooks roles
yamllint .
ansible-playbook --syntax-check playbooks/pre_check.yml
```
