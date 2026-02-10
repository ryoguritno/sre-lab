# Task 05: Cloud Infrastructure — Multi-Cloud Terraform Modules

## Objective

```
GOAL:         Create Terraform modules for deploying Talos Linux clusters
              on AWS, GCP, Azure, and Hetzner with consistent interface
SCOPE:        infrastructure/terraform/modules/
              infrastructure/terraform/environments/
OUT OF SCOPE: Kubernetes components (reuse from local tasks)
```

---

## Context

```
WHY:          Enable production deployment of SRE Lab on any major cloud
              provider with infrastructure-as-code best practices
TICKET:       SRE-005
URGENCY:      Medium
DEADLINE:     Week 3-4
```

### Background

The SRE Lab should be deployable to any cloud with the same Kubernetes stack. Infrastructure layer differences:
- **Compute**: EC2, GCE, Azure VM, Hetzner Cloud
- **Networking**: VPC/VNet/VPC, subnets, security groups
- **Load Balancing**: NLB/Cloud LB for Kubernetes API
- **Storage**: EBS/PD/Managed Disk for PVCs

Talos Linux works on all major clouds via:
- Official cloud images (AMI, GCE image, etc.)
- cloud-init compatible configuration
- Terraform provider for Talos config management

### Blocked By

- [x] Task 01-04: Local environment working (validates K8s configs)
- [ ] Cloud provider accounts and credentials

---

## Acceptance Criteria

- [ ] Terraform module for Talos cluster (generic)
- [ ] AWS implementation (VPC, EC2, NLB, IAM)
- [ ] GCP implementation (VPC, GCE, LB, IAM)
- [ ] Azure implementation (VNet, VM, LB, IAM)
- [ ] Hetzner implementation (Network, Server, LB)
- [ ] Consistent module interface across all providers
- [ ] Talos machine configs generated via Terraform
- [ ] Control plane HA with load balancer
- [ ] Worker node auto-scaling ready (ASG/MIG/VMSS)
- [ ] Outputs: kubeconfig, talosconfig, endpoints
- [ ] State stored in cloud backend (S3, GCS, etc.)
- [ ] CI/CD ready (GitHub Actions workflow from mega-prompt)
- [ ] Cost-optimized defaults for dev environment

### Definition of Done

- [ ] `terraform apply` creates working cluster on each provider
- [ ] `talosctl` can reach cluster
- [ ] `kubectl` can manage cluster
- [ ] Same K8s manifests work on all providers

---

## Constraints

```
MUST:
  - Use official Talos cloud images
  - Use Talos Terraform provider for config generation
  - Create isolated VPC/VNet per environment
  - Use cloud-native load balancer for API server
  - Store Terraform state in cloud backend
  - Tag all resources with: Environment, Project, ManagedBy
  - Support 1-node dev and 3-node HA control plane
  - Generate talosconfig and kubeconfig as outputs
  
MUST NOT:
  - Hard-code credentials (use env vars or profiles)
  - Create public-facing K8s API without IP allowlist
  - Use deprecated instance types
  - Mix resources across environments in same state
  
PREFER:
  - Spot/Preemptible instances for workers (cost savings)
  - Private subnets for nodes, public only for LB
  - Separate modules for networking vs compute
  - Use data sources for AMI/image lookup
```

---

## Technical Details

### Module Architecture

```
infrastructure/terraform/
├── modules/
│   ├── talos-cluster/           # Generic cluster logic
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── talos-config.tf      # Talos provider config
│   │
│   ├── aws/
│   │   ├── vpc/                 # AWS VPC module
│   │   ├── compute/             # EC2 instances
│   │   ├── loadbalancer/        # NLB for API
│   │   └── iam/                 # Instance profiles
│   │
│   ├── gcp/
│   │   ├── vpc/
│   │   ├── compute/
│   │   ├── loadbalancer/
│   │   └── iam/
│   │
│   ├── azure/
│   │   ├── vnet/
│   │   ├── compute/
│   │   ├── loadbalancer/
│   │   └── identity/
│   │
│   └── hetzner/
│       ├── network/
│       ├── compute/
│       └── loadbalancer/
│
└── environments/
    ├── aws-dev/
    │   ├── main.tf
    │   ├── variables.tf
    │   ├── terraform.tfvars
    │   ├── backend.tf
    │   └── outputs.tf
    ├── aws-prod/
    ├── gcp-dev/
    ├── gcp-prod/
    ├── azure-dev/
    ├── hetzner-dev/
    └── ...
```

### Common Module Interface

```hcl
# All provider-specific modules should accept these inputs
variable "cluster_name" {
  type = string
}

variable "environment" {
  type = string
}

variable "talos_version" {
  type    = string
  default = "v1.6.0"
}

variable "kubernetes_version" {
  type    = string
  default = "1.29.0"
}

variable "control_plane_count" {
  type    = number
  default = 3
}

variable "worker_count" {
  type    = number
  default = 2
}

variable "control_plane_instance_type" {
  type = string  # Provider-specific default
}

variable "worker_instance_type" {
  type = string
}

variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "pod_cidr" {
  type    = string
  default = "10.244.0.0/16"
}

variable "service_cidr" {
  type    = string
  default = "10.96.0.0/12"
}

# All modules should provide these outputs
output "talosconfig" {
  value     = talos_client_configuration.this.talos_config
  sensitive = true
}

output "kubeconfig" {
  value     = talos_cluster_kubeconfig.this.kubeconfig_raw
  sensitive = true
}

output "cluster_endpoint" {
  value = "https://${module.loadbalancer.dns_name}:6443"
}

output "control_plane_ips" {
  value = module.compute.control_plane_ips
}
```

### Provider Comparison

| Feature | AWS | GCP | Azure | Hetzner |
|---------|-----|-----|-------|---------|
| Image Source | AMI (official) | GCE Image | Marketplace | Snapshot/ISO |
| Control Plane LB | NLB | TCP LB | Standard LB | Hetzner LB |
| Instance Types | t3.medium+ | e2-medium+ | Standard_B2s+ | CX21+ |
| Spot Support | Yes (ASG) | Yes (MIG) | Yes (VMSS) | No |
| State Backend | S3 + DynamoDB | GCS | Azure Blob | S3 compatible |
| DNS | Route53 | Cloud DNS | Azure DNS | External |

### Files / Resources in Scope

| File/Resource | Action | Notes |
|---------------|--------|-------|
| `infrastructure/terraform/modules/talos-cluster/` | Create | Generic Talos config |
| `infrastructure/terraform/modules/aws/` | Create | AWS-specific modules |
| `infrastructure/terraform/modules/gcp/` | Create | GCP-specific modules |
| `infrastructure/terraform/modules/azure/` | Create | Azure-specific modules |
| `infrastructure/terraform/modules/hetzner/` | Create | Hetzner-specific modules |
| `infrastructure/terraform/environments/*/` | Create | Per-environment configs |
| `.github/workflows/deploy-cloud.yml` | Create | CI/CD workflow |
| `scripts/cloud-deploy.sh` | Create | Deployment wrapper |

---

## Implementation Approach (Per Provider)

### AWS Implementation

```
1. VPC Module
   - VPC with DNS support
   - Public + private subnets (2 AZs)
   - Internet Gateway, NAT Gateway
   - Security groups

2. Compute Module
   - Launch template with Talos AMI
   - Control plane instances (or ASG)
   - Worker ASG with mixed instances
   - Instance profile for cloud integration

3. Load Balancer Module
   - NLB for K8s API (TCP 6443)
   - Target group for control planes
   - Health checks

4. Talos Config
   - Generate machine configs
   - Apply via cloud-init user data
   - Bootstrap cluster
```

### GCP Implementation

```
1. VPC Module
   - VPC with custom subnets
   - Firewall rules
   - Cloud NAT for egress

2. Compute Module
   - Instance template with Talos image
   - Control plane instances
   - Worker MIG (managed instance group)
   - Service account

3. Load Balancer Module
   - Regional TCP LB
   - Backend service
   - Health checks

4. Talos Config (same as AWS)
```

### Azure Implementation

```
1. VNet Module
   - Virtual network
   - Subnets
   - NSGs
   - NAT Gateway

2. Compute Module
   - Control plane VMs
   - Worker VMSS
   - Managed identity

3. Load Balancer Module
   - Standard Load Balancer
   - Backend pool
   - Health probes

4. Talos Config (same pattern)
```

### Hetzner Implementation

```
1. Network Module
   - Hetzner Network
   - Subnets
   - Firewall

2. Compute Module
   - Control plane servers
   - Worker servers
   - Placement groups

3. Load Balancer Module
   - Hetzner Load Balancer
   - Services + targets

4. Talos Config (same pattern)
```

---

## Reference Materials

| Type | Link | Notes |
|------|------|-------|
| Talos AWS Guide | https://www.talos.dev/v1.6/talos-guides/install/cloud-platforms/aws/ | |
| Talos GCP Guide | https://www.talos.dev/v1.6/talos-guides/install/cloud-platforms/gcp/ | |
| Talos Azure Guide | https://www.talos.dev/v1.6/talos-guides/install/cloud-platforms/azure/ | |
| Talos Hetzner Guide | https://www.talos.dev/v1.6/talos-guides/install/cloud-platforms/hetzner/ | |
| Talos Terraform Provider | https://registry.terraform.io/providers/siderolabs/talos/latest | |
| Example: Talos on AWS | https://github.com/siderolabs/contrib/tree/main/examples/terraform/aws | Official |

---

## Risks & Rollback

### Potential Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Cloud cost overrun | Med | Med | Use small instances, destroy dev nightly |
| Talos image not available | Low | High | Pre-copy to account, use supported regions |
| LB health check fails | Med | Med | Tune intervals, check security groups |
| State lock issues | Low | Med | Use cloud-native locking |

### Rollback Plan

```
1. terraform destroy -auto-approve
2. Verify all resources deleted
3. Check for orphaned resources (LBs, EIPs)
4. Fix configuration
5. Re-deploy
```

---

## Cost Estimates (Dev Environment)

| Provider | Instance Config | Monthly Est. |
|----------|-----------------|--------------|
| AWS | 1x t3.medium CP, 1x t3.small worker | ~$30-40 |
| GCP | 1x e2-medium CP, 1x e2-small worker | ~$25-35 |
| Azure | 1x Standard_B2s CP, 1x Standard_B1s worker | ~$30-40 |
| Hetzner | 1x CX21 CP, 1x CX11 worker | ~$10-15 |

---

## Validation Steps

### Per-Provider Testing

```bash
# Set provider
export CLOUD=aws  # or gcp, azure, hetzner
export ENV=dev

# Initialize
cd infrastructure/terraform/environments/${CLOUD}-${ENV}
terraform init

# Plan
terraform plan -out=tfplan

# Apply
terraform apply tfplan

# Get credentials
terraform output -raw talosconfig > ~/.talos/config
terraform output -raw kubeconfig > ~/.kube/config

# Verify Talos
talosctl health

# Verify Kubernetes
kubectl get nodes

# Deploy observability stack (same as local)
task k8s:bootstrap
task k8s:observability

# Cleanup
terraform destroy
```

---

## Notes

```
- Start with ONE provider (recommend AWS or Hetzner for cost)
- Reuse all Kubernetes manifests from local tasks
- Cloud-specific differences are ONLY at infrastructure layer
- Consider Crossplane for multi-cloud abstraction (future)

Priority order for implementation:
1. AWS (most common, best Talos docs)
2. Hetzner (cheapest, great for homelab users)
3. GCP (good Talos support)
4. Azure (enterprise requirement)

Cost optimization:
- Use spot/preemptible for workers
- Destroy dev environments when not in use
- Use scheduled scaling for non-prod
```

---

*Created: 2024-01-20*
*Status: Draft*
*Depends on: Task 01-04 (for K8s manifest validation)*
