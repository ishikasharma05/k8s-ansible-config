# k8s-ansible-config

Ansible configuration for EKS worker nodes — part of the Capstone Project 5
platform (see [k8s-gitops-platform](../k8s-gitops-platform) for Terraform/EKS
and [k8s-gitops-config](../k8s-gitops-config) for ArgoCD-managed manifests).

## What this does

Prometheus and Alertmanager themselves run **inside** the cluster via Helm
(see the GitOps repo). This repo does **not** install them. Its job is
node-level configuration that Helm can't reach:

| Role | Purpose |
|---|---|
| `node-setup` | OS hardening (swap off, sysctl, UFW, kernel modules), containerd install/config |
| `node-exporter` | Installs and runs the Prometheus node_exporter binary as a systemd service on every worker node |
| `prometheus-config` | Generates a `file_sd` target list (from dynamic inventory) so in-cluster Prometheus can scrape node_exporter on every EC2 worker |
| `alertmanager-config` | Renders `alertmanager.yml` with a Slack receiver, to be applied as a K8s Secret consumed by the Helm release |

## Setup

```bash
pip install boto3 botocore
ansible-galaxy collection install -r requirements.yml
ansible-vault create group_vars/vault.yml   # add slack_webhook_url here
```

## Inventory

Uses the `amazon.aws.aws_ec2` dynamic inventory plugin, filtered on the
`Role=eks-worker` tag set in Terraform (`terraform/eks.tf` — worker ASG/launch
template must tag instances accordingly).

Verify inventory resolves before running anything:
```bash
ansible-inventory -i inventory/aws_ec2.yml --graph
```

## Run

```bash
ansible-playbook -i inventory/aws_ec2.yml site.yml --ask-vault-pass
```

## Status

- [ ] Verified against live EKS worker nodes
- [ ] Slack webhook configured and alert tested end-to-end
- [ ] node_targets.json synced into cluster ConfigMap
- [ ] alertmanager.yml synced into cluster Secret
