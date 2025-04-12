# **Kubernetes GitOps Deployment with ArgoCD and Helm on GCP**

## **Introduction: Why GitOps?**

As modern applications grow in complexity, managing Kubernetes deployments manually becomes a challenge. Traditional deployment strategies often rely on imperative commands (`kubectl apply`, `helm install`), which can lead to:

- **Drift between environments** – Changes applied manually in staging may not match production.
- **Lack of visibility** – No single source of truth for deployment history.
- **Rollback challenges** – Reverting a bad deployment is not straightforward.

### **What is GitOps?**

GitOps is a **declarative** approach to managing Kubernetes applications, where:

- The **Git repository is the single source of truth** for all deployments.
- Any change to Kubernetes manifests in Git is automatically **applied and reconciled** in the cluster.
- Rollbacks and disaster recovery are as simple as reverting a commit in Git.

### **Key Benefits of GitOps**

- **Automated, consistent deployments** – No manual `kubectl` commands.
- **Rollback with Git history** – Deploy any previous version by reverting a commit.
- **Increased security** – All changes must go through Git, providing **auditability**.
- **Self-healing infrastructure** – If someone modifies Kubernetes resources manually, GitOps corrects the drift.

### **Why ArgoCD?**

ArgoCD is a **Kubernetes-native GitOps tool** that continuously syncs application manifests from Git to Kubernetes. It provides:

- **Automated deployments** from Git.
- **Real-time monitoring** of application status.
- **Multi-cluster support** for managing multiple environments.

---

## **1. Setting Up the GKE Cluster with Terraform**

We'll start by provisioning a **GKE cluster** using **Terraform**, ensuring infrastructure is managed as code.

### **1.1. Define Terraform Configuration**

Let’s organize our Terraform code. Create a folder structure like this:

```
terraform/
├── gke.tf
├── outputs.tf
├── providers.tf
├── variables.tf
├── vpc.tf
```

Instead of using the default VPC, we’ll define our own network with isolated subnet for our GKE workload (`vpc.tf`):

```hcl
# VPC
resource "google_compute_network" "vpc" {
  project                 = var.project_id
  name                    = "${var.project_id}-vpc"
  auto_create_subnetworks = "false"
}

# Subnet
resource "google_compute_subnetwork" "subnet" {
  project       = var.project_id
  name          = "${var.project_id}-subnet"
  region        = var.region
  network       = google_compute_network.vpc.name
  ip_cidr_range = "10.10.0.0/24"
}

```

With networking in place, it’s time to spin up a GKE cluster (`gke.tf`):

```hcl
# GKE cluster
resource "google_container_cluster" "primary" {
  project  = var.project_id
  name     = "${var.project_id}-gke"
  location = var.region

  # We can't create a cluster with no node pool defined, but we want to only use
  # separately managed node pools. So we create the smallest possible default
  # node pool and immediately delete it.
  remove_default_node_pool = true
  initial_node_count       = 1

  network    = google_compute_network.vpc.name
  subnetwork = google_compute_subnetwork.subnet.name
}

resource "google_container_node_pool" "primary_nodes" {
  project  = var.project_id
  name     = google_container_cluster.primary.name
  location = var.region
  cluster  = google_container_cluster.primary.name

  node_count = var.gke_num_nodes

  node_config {
    oauth_scopes = [
      "https://www.googleapis.com/auth/logging.write",
      "https://www.googleapis.com/auth/monitoring",
    ]

    preemptible  = true
    machine_type = "n1-standard-1"
  }
}
```

### **1.2. Deploy the Cluster**

First authenticate with your GCP account:

```sh
gcloud auth application-default login
```

Initialize and apply Terraform:

```sh
terraform init
terraform apply -auto-approve
```

Once done, configure `kubectl`:

```sh
gcloud container clusters get-credentials primary-cluster --region us-central1
```

Verify cluster status:

```sh
kubectl get nodes
```
