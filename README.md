# GCP + GitHub Actions + GKE – Practical Notes

This document collects common questions and clear answers about
managing credentials, ingress, and cross-project access
when working with **Google Cloud Platform (GCP)**,
**GitHub Actions**, **Terraform**, and **GKE/NGINX Ingress**.

---

## 1. Managing GCP Credentials in GitHub Actions

**Preferred approach – Workload Identity Federation (OIDC)**  
* No long-lived JSON key needed.
* Steps:
  1. Create a GCP Service Account with minimal roles.
  2. Set up a Workload Identity Pool & Provider that trusts GitHub’s OIDC token.
  3. In your workflow use `permissions: id-token: write` and authenticate with:
     ```bash
     gcloud auth login --workload-identity-provider <provider> --service-account <sa>
     ```
* Benefit: **short-lived tokens**, nothing hard-coded.

**Fallback** – GitHub Encrypted Secrets  
* Store the service-account key JSON as a secret (`GOOGLE_CREDENTIALS`).
* Activate with `gcloud auth activate-service-account --key-file=$GOOGLE_CREDENTIALS`.

---

## 2. Image Pull Secrets

* For images in **Artifact Registry/GCR** **inside GKE**:
  * Usually **no manual secret**.  
  * Ensure the node pool’s service account has `roles/artifactregistry.reader`.
* Workload Identity Federation affects **build/push** only, not node pulls.
* Use a Kubernetes `imagePullSecret` **only** when pulling from:
  * third-party private registries,
  * or a cluster without a Google-managed node identity.

---

## 3. GKE Ingress and NGINX

### Native GCP (GCE) Ingress
* Create a Kubernetes `Ingress` with class `gce`.
* GKE provisions a **Google HTTP(S) Load Balancer**, public IP, and managed SSL.

### NGINX Ingress Controller
* Deploy via Helm.
* Expose it with a **Service of type LoadBalancer**:
  * GKE provisions a **Google Network Load Balancer** (L4) and a public IP.
* NGINX handles all Layer-7 routing and TLS termination.

> **No extra authentication is required.**  
> The GKE control plane uses its own credentials to create the Cloud LB.

---

## 4. NGINX Public IP

* The **global public IP** is provided by the Google Network Load Balancer.
* Reserve a static IP with  
  `gcloud compute addresses create my-nginx-ip --global`
  and annotate the Service:
  ```yaml
  annotations:
    networking.gke.io/load-balancer-ipv4: "my-nginx-ip"
