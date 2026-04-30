# NFS Test Setup Guide

This guide walks through the complete setup process for deploying the NFS test chart on OpenShift, based on the actual implementation steps.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [One-Time Platform Setup](#one-time-platform-setup)
3. [GitHub Repository Setup](#github-repository-setup)
4. [Self-Hosted Runner Setup](#self-hosted-runner-setup)
5. [Verification](#verification)

---

## Prerequisites

### Required Access
- **OpenShift Cluster**: Access to an OpenShift cluster (e.g., Bell CaaS platform)
- **Namespace**: Pre-created namespace (e.g., `bell-test`)
- **NFS Server**: Accessible NFS server with configured export path
- **GitHub**: Repository with Actions enabled

### Required Tools
- `oc` CLI (OpenShift command-line tool)
- `helm` 3.x
- `git`
- Access to create GitHub Actions runners

---

## One-Time Platform Setup

These steps must be completed by a **cluster administrator** before the first deployment.

### 1. Create Namespace (if not exists)

```bash
oc create namespace bell-test
```

### 2. Create NFS PersistentVolume

The PV must be created by a cluster admin as it's a cluster-scoped resource.

**Create file: `nfs-pv.yaml`**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ndex-nfs-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: <NFS_SERVER_IP>  # Replace with your NFS server IP
    path: /exports/ndex/data  # Replace with your NFS export path
```

**Apply the PV:**

```bash
oc apply -f nfs-pv.yaml
```

**Verify PV creation:**

```bash
oc get pv ndex-nfs-pv
# Expected: STATUS should be "Available"
```

**Important Notes:**
- The PV has **no storageClassName** (empty/unset)
- The PV name is `ndex-nfs-pv` (referenced in values.yaml)
- Access mode is `ReadWriteMany` (RWX)
- Reclaim policy is `Retain` (data persists after PVC deletion)

### 3. Grant SCC Permissions

The service account needs `hostmount-anyuid` SCC to mount NFS volumes.

**Request from Bell CaaS Platform Team:**

```bash
oc adm policy add-scc-to-user hostmount-anyuid -z nfs-sa -n bell-test
```

**Verify SCC assignment:**

```bash
oc get scc hostmount-anyuid -o yaml | grep -A 5 users
```

You should see `system:serviceaccount:bell-test:nfs-sa` in the users list.

---

## GitHub Repository Setup

### 1. Create GitHub Repository

1. Create a new repository (e.g., `ocpnfs`)
2. Clone this repository or push the code to your new repo

### 2. Configure GitHub Secrets

Go to **Settings → Secrets and variables → Actions → New repository secret**

Create the following secrets:

| Secret Name | Description | Example Value |
|------------|-------------|---------------|
| `OPENSHIFT_TOKEN` | Service account token for authentication | `sha256~abc123...` |
| `OPENSHIFT_SERVER` | OpenShift API server URL | `https://api.cluster.example.com:6443` |
| `OPENSHIFT_PROJECT` | Target namespace/project name | `bell-test` |
| `NFS_SERVER_IP` | NFS server IP address | `172.24.127.64` |

#### How to Get OPENSHIFT_TOKEN

**Option 1: Using existing service account**
```bash
# Get token from service account
oc sa get-token <service-account-name> -n bell-test
```

**Option 2: Create a dedicated service account**
```bash
# Create service account
oc create sa github-deployer -n bell-test

# Grant edit role
oc policy add-role-to-user edit system:serviceaccount:bell-test:github-deployer -n bell-test

# Get token
oc sa get-token github-deployer -n bell-test
```

#### How to Get OPENSHIFT_SERVER

```bash
oc whoami --show-server
```

---

## Self-Hosted Runner Setup

The workflow requires a self-hosted runner with the label `oc-run`.

### 1. Prepare Runner Machine

**Requirements:**
- Linux machine (VM or physical)
- Network access to OpenShift cluster
- Network access to NFS server
- Network access to GitHub

**Install required tools:**

```bash
# Install oc CLI
curl -LO https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable/openshift-client-linux.tar.gz
tar xvf openshift-client-linux.tar.gz
sudo mv oc kubectl /usr/local/bin/

# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify installations
oc version
helm version
```

### 2. Register GitHub Actions Runner

1. Go to your GitHub repository
2. Navigate to **Settings → Actions → Runners → New self-hosted runner**
3. Select **Linux** as the operating system
4. Follow the provided commands to download and configure the runner

**Example commands:**

```bash
# Create a folder
mkdir actions-runner && cd actions-runner

# Download the latest runner package
curl -o actions-runner-linux-x64-2.311.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-linux-x64-2.311.0.tar.gz

# Extract the installer
tar xzf ./actions-runner-linux-x64-2.311.0.tar.gz

# Configure the runner
./config.sh --url https://github.com/YOUR_USERNAME/ocpnfs --token YOUR_TOKEN

# IMPORTANT: When prompted for labels, add: oc-run
# Example: Enter any additional labels (ex. label-1,label-2): oc-run

# Install and start the runner as a service
sudo ./svc.sh install
sudo ./svc.sh start
```

### 3. Verify Runner

1. Go to **Settings → Actions → Runners**
2. You should see your runner with status "Idle" and label "oc-run"

---

## Verification

### 1. Test OpenShift Connectivity

From the runner machine:

```bash
# Login using token
oc login --token=<YOUR_TOKEN> --server=<YOUR_SERVER>

# Switch to project
oc project bell-test

# Verify access
oc get pods
```

### 2. Test NFS Connectivity

From the runner machine:

```bash
# Test NFS mount (requires nfs-common package)
sudo apt-get install -y nfs-common  # Ubuntu/Debian
# or
sudo yum install -y nfs-utils       # RHEL/CentOS

# Test mount
sudo mount -t nfs <NFS_SERVER_IP>:/exports/ndex/data /mnt
ls -la /mnt
sudo umount /mnt
```

### 3. Verify PV Status

```bash
oc get pv ndex-nfs-pv
```

Expected output:
```
NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
ndex-nfs-pv   1Gi        RWX            Retain           Available                                   10m
```

### 4. Verify SCC Assignment

```bash
oc get scc hostmount-anyuid -o yaml | grep "system:serviceaccount:bell-test:nfs-sa"
```

Should return the service account reference.

---

## Next Steps

Once setup is complete, proceed to [DEPLOYMENT-GUIDE.md](DEPLOYMENT-GUIDE.md) to deploy the NFS test chart.

## Common Setup Issues

See [TROUBLESHOOTING-GUIDE.md](TROUBLESHOOTING-GUIDE.md) for solutions to common problems.