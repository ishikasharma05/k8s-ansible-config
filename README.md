# k8s-ansible-config

Ansible playbooks and roles that configure EKS worker nodes for the [`k8s-gitops-platform`](https://github.com/ishikasharma05/k8s-gitops-config) capstone project — OS hardening, container runtime setup, and monitoring agent configuration, run against nodes provisioned by Terraform.

## Structure
```
k8s-ansible-config/
├── inventory/
│   └── aws_ec2.yml          # dynamic inventory — discovers EC2 worker nodes by tag
├── group_vars/
│   └── all/
│       └── vault.yml        # encrypted — Slack webhook URL for Alertmanager
├── roles/
│   ├── node-setup/          # OS hardening, Docker/containerd install
│   ├── node-exporter/       # Node Exporter binary + systemd service
│   ├── prometheus-config/   # templates scrape target config
│   └── alertmanager-config/ # templates alertmanager.yml with Slack receiver
├── site.yml                 # ties all roles together
└── ansible.cfg
```

## Prerequisites
- Worker nodes already provisioned via Terraform (EKS cluster `gitops-platform-eks`, `ap-south-1`)
- AWS CLI configured with access to describe/tag EC2 instances (dynamic inventory queries AWS directly)
- Ansible + `amazon.aws` collection installed

## Usage

**1. Confirm the dynamic inventory finds your nodes**
```bash
ansible-inventory -i inventory/aws_ec2.yml --graph
```

**2. Run the full playbook**
```bash
ansible-playbook -i inventory/aws_ec2.yml site.yml --ask-vault-pass
```
You'll be prompted for the Ansible Vault password (protects `slack_webhook_url` in `group_vars/all/vault.yml`).

**3. Confirm success**
Check the `PLAY RECAP` at the end of the run — `failed=0` on every host means all four roles applied cleanly.

## Roles

| Role | What it does |
|---|---|
| `node-setup` | Basic OS hardening, installs Docker/containerd |
| `node-exporter` | Installs Node Exporter as a systemd service on each worker node |
| `prometheus-config` | Renders Prometheus scrape target config from a Jinja2 template |
| `alertmanager-config` | Renders `alertmanager.yml` (with the Slack receiver) locally, and prints the `kubectl` command to load it into the cluster as a Secret |

## Managing the vault
```bash
# View current secret content
ansible-vault view group_vars/all/vault.yml

# Edit in place
ansible-vault edit group_vars/all/vault.yml

# If the password is lost — no recovery possible, must recreate
rm group_vars/all/vault.yml
ansible-vault create group_vars/all/vault.yml
# then re-add:
#   slack_webhook_url: "https://hooks.slack.com/services/..."
```

## Note on Node Exporter
This repo installs Node Exporter directly on the hosts via systemd. When the `kube-prometheus-stack` Helm chart is later installed on the cluster (see the main platform repo), its own Node Exporter DaemonSet is disabled to avoid a port-9100 conflict with this systemd-managed instance.