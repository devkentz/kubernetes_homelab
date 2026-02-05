---
name: k8s-homelab-builder
description: Guide and automate the setup of a Kubernetes bare-metal home lab including MetalLB, Longhorn, Bind9, and Cert-Manager.
---

# Kubernetes Home Lab Builder

This skill helps you build a Kubernetes bare-metal home lab by following the `K8S_HOMELAB_GUIDE`. It provides step-by-step guidance and automation for installing key components.

## Workflow

Follow these steps to build your home lab. You can ask to perform any specific step or go through them sequentially.

### 1. Prerequisites Check

**Goal**: Ensure all nodes have necessary OS packages and kernel modules.

**Instructions**:
1.  Ask the user if they want to check the prerequisites on their nodes.
2.  If yes, guide them to run the following command on *each* node (or help them run it if you have SSH access, but usually just provide the command):
    ```bash
    sudo apt update && sudo apt install -y open-iscsi nfs-common jq
    sudo systemctl enable --now iscsid
    ```
3.  Verify that `iscsi_tcp` and `nbd` modules are loaded.

### 2. MetalLB Setup (Load Balancer)

**Goal**: Enable `Type: LoadBalancer` services.

**Instructions**:
1.  **Install MetalLB**:
    ```bash
    kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml
    ```
2.  **Configure IP Pool**:
    - The skill includes a template at `assets/metallb-pool.yaml`.
    - **Action**: Read this file, ask the user for their desired IP range (default: `192.168.1.200-192.168.1.250`), modify the file content in memory (or write a temporary file), and apply it using `kubectl apply -f ...`.

### 3. Longhorn Setup (Storage)

**Goal**: Provide distributed block storage.

**Instructions**:
1.  **Environment Check**:
    ```bash
    curl -sSfL https://raw.githubusercontent.com/longhorn/longhorn/v1.7.2/scripts/environment_check.sh | bash
    ```
2.  **Install Longhorn**:
    ```bash
    kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.7.2/deploy/longhorn.yaml
    ```
3.  **Set Default Storage Class**:
    ```bash
    kubectl patch storageclass longhorn -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
    ```

### 4. Bind9 Setup (Internal DNS)

**Goal**: Set up an authoritative DNS server for the cluster.

**Instructions**:
1.  Explain that Bind9 should ideally run *outside* the cluster (e.g., Docker on NAS/Pi).
2.  **Generate Configs**:
    - Use templates in `assets/bind9/named.conf` and `assets/bind9/db.homelab.local`.
    - **Action**: Ask the user for the TSIG Key Secret (or help them generate it with `tsig-keygen -a HMAC-SHA256 externaldns-key`) and the DNS Server IP.
    - Update the templates with these values.
    - Offer to save these files to a local directory (e.g., `./bind9-config`) so the user can run the Docker container.
3.  **Run Bind9**: Provide the Docker run command (see `references/guide.md`).

### 5. External-DNS Setup

**Goal**: Automate DNS record updates.

**Instructions**:
1.  **Install with Helm**:
    - Ask for the Bind9 Server IP and TSIG Secret (if not already known from Step 4).
    - Run the Helm install command:
    ```bash
    helm install external-dns external-dns/external-dns \
      --set provider=rfc2136 \
      --set rfc2136.host=<BIND9_IP> \
      --set rfc2136.zone=homelab.local \
      --set rfc2136.tsigKeyname=externaldns-key \
      --set rfc2136.tsigSecret=<TSIG_SECRET> \
      --set rfc2136.tsigAlgorithm=hmac-sha256
    ```

### 6. Cert-Manager & Private Root CA

**Goal**: Issue trusted SSL certificates for internal services.

**Instructions**:
1.  **Install Cert-Manager**:
    ```bash
    helm install cert-manager jetstack/cert-manager \
      --namespace cert-manager \
      --create-namespace \
      --set installCRDs=true
    ```
2.  **Bootstrap CA**:
    - Use `assets/ca/cluster-issuers.yaml`.
    - Apply it: `kubectl apply -f assets/ca/cluster-issuers.yaml` (adjust path as needed).
3.  **Distribute CA Bundle**:
    - Install `trust-manager`.
    - Create the Bundle resource (see `references/guide.md` section 4.3).

## References

For detailed explanations and background, see [K8S_HOMELAB_GUIDE](references/guide.md).