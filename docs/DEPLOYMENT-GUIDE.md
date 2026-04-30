# NFS Test Deployment Guide

This guide covers deploying the NFS test chart using GitHub Actions or manually via command line.

## Table of Contents
1. [Deployment via GitHub Actions](#deployment-via-github-actions)
2. [Manual Deployment](#manual-deployment)
3. [Verifying Deployment](#verifying-deployment)
4. [Viewing Test Results](#viewing-test-results)
5. [Teardown](#teardown)

---

## Deployment via GitHub Actions

### Automatic Deployment (Push Trigger)

The workflow automatically triggers when you push changes to the `nfs-test-chart/` directory on the `main` branch.

```bash
# Make changes to the chart
vim nfs-test-chart/values.yaml

# Commit and push
git add nfs-test-chart/
git commit -m "Update NFS configuration"
git push origin main
```

The workflow will automatically:
1. Checkout the code
2. Login to OpenShift
3. Clean up existing resources (pod and PVC)
4. Deploy the Helm chart
5. Wait for pod to be ready
6. Display pod logs and status

### Manual Deployment (Workflow Dispatch)

1. Go to your GitHub repository
2. Click **Actions** tab
3. Select **NFS Test Deploy** workflow
4. Click **Run workflow** button
5. Select branch: `main`
6. Select action: `deploy`
7. Click **Run workflow**

### Monitoring Workflow Execution

1. Click on the running workflow
2. Click on the `nfs-test` job
3. Expand each step to see detailed logs

**Key steps to monitor:**
- **Login to OpenShift**: Verifies authentication
- **Clean up existing resources**: Removes old pod/PVC
- **Deploy NFS Test**: Helm installation progress
- **Wait for Pod Ready**: Pod startup status
- **Print Pod Logs**: NFS mount test results

---

## Manual Deployment

### Prerequisites

Ensure you have completed the [SETUP-GUIDE.md](SETUP-GUIDE.md) first.

### Step 1: Clone Repository

```bash
git clone https://github.com/kere-sifon/ocpnfs.git
cd ocpnfs
```

### Step 2: Login to OpenShift

```bash
oc login --token=<YOUR_TOKEN> --server=<YOUR_SERVER>
oc project bell-test
```

### Step 3: Clean Up Existing Resources (if any)

```bash
# Delete existing pod (if exists)
oc delete pod nfs-test-pod --ignore-not-found=true

# Delete existing PVC (if exists)
oc delete pvc nfs-pvc --ignore-not-found=true

# Wait for cleanup
sleep 5
```

**Why cleanup is needed:**
- PVC spec is immutable after creation
- If storageClassName needs to change, PVC must be recreated
- Pod must be deleted first to release the PVC

### Step 4: Deploy Helm Chart

```bash
helm upgrade --install nfs-test ./nfs-test-chart \
  --set nfs.server=172.24.127.64 \
  --set namespace=bell-test \
  --wait \
  --timeout=5m
```

**Parameters:**
- `nfs.server`: Your NFS server IP (override default)
- `namespace`: Target namespace (should match your project)
- `--wait`: Wait for resources to be ready
- `--timeout=5m`: Maximum wait time

### Step 5: Wait for Pod Ready

```bash
oc wait --for=condition=Ready pod/nfs-test-pod \
  -n bell-test \
  --timeout=300s
```

---

## Verifying Deployment

### Check All Resources

```bash
oc get all,pv,pvc -n bell-test
```

Expected output:
```
NAME               READY   STATUS    RESTARTS   AGE
pod/nfs-test-pod   1/1     Running   0          2m

NAME                      TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes        ClusterIP   10.96.0.1    <none>        443/TCP   30d

NAME                                          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM              STORAGECLASS   REASON   AGE
persistentvolume/ndex-nfs-pv                  1Gi        RWX            Retain           Bound    bell-test/nfs-pvc                          10m

NAME                            STATUS   VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/nfs-pvc   Bound    ndex-nfs-pv   1Gi        RWX                           2m
```

### Check PVC Binding

```bash
oc get pvc nfs-pvc -n bell-test
```

**Expected:**
- STATUS: `Bound`
- VOLUME: `ndex-nfs-pv`
- CAPACITY: `1Gi`

### Check Pod Status

```bash
oc get pod nfs-test-pod -n bell-test
```

**Expected:**
- READY: `1/1`
- STATUS: `Running`
- RESTARTS: `0`

### Describe Pod (Detailed Info)

```bash
oc describe pod nfs-test-pod -n bell-test
```

Look for:
- **Events**: Should show successful volume mount
- **Volumes**: Should show `nfs-volume` mounted from PVC
- **Conditions**: All should be `True`

---

## Viewing Test Results

### View Pod Logs

```bash
oc logs nfs-test-pod -n bell-test
```

**Expected output:**
```
=== NFS Mount Test Started ===
Mount point: /mnt/nfs
Testing write access...
✓ Write test successful: /mnt/nfs/test-1714507200.txt
Testing read access...
Test data written at Wed Apr 30 20:00:00 UTC 2026
✓ Read test successful
Directory contents:
total 12
drwxr-xr-x 2 root root 4096 Apr 30 20:00 .
drwxr-xr-x 3 root root 4096 Apr 30 20:00 ..
-rw-r--r-- 1 root root   45 Apr 30 20:00 test-1714507200.txt
=== NFS Mount Test Completed Successfully ===
Sleeping for 3600 seconds...
```

### Follow Logs in Real-Time

```bash
oc logs -f nfs-test-pod -n bell-test
```

Press `Ctrl+C` to stop following.

### Check Files on NFS Server

From the NFS server or a machine with NFS mount access:

```bash
ls -la /exports/ndex/data/
```

You should see the test files created by the pod.

---

## Teardown

### Via GitHub Actions

1. Go to **Actions** tab
2. Select **NFS Test Deploy** workflow
3. Click **Run workflow**
4. Select action: `teardown`
5. Click **Run workflow**

This will:
- Uninstall the Helm release
- Remove the pod, PVC, and service account
- Leave the PV intact (managed by cluster admin)

### Manual Teardown

```bash
# Uninstall Helm release
helm uninstall nfs-test -n bell-test

# Verify resources are removed
oc get all,pvc -n bell-test
```

**Note:** The PV `ndex-nfs-pv` is NOT deleted as it's managed by the cluster administrator.

### Complete Cleanup (Cluster Admin Only)

If you need to completely remove everything including the PV:

```bash
# Delete PV (cluster admin only)
oc delete pv ndex-nfs-pv

# Remove SCC assignment (cluster admin only)
oc adm policy remove-scc-from-user hostmount-anyuid -z nfs-sa -n bell-test
```

---

## Redeployment

To redeploy after teardown:

1. Ensure PV still exists: `oc get pv ndex-nfs-pv`
2. Follow the deployment steps again
3. The workflow's cleanup step will handle any leftover resources

---

## Customizing Deployment

### Override Values

You can override any value in `values.yaml`:

```bash
helm upgrade --install nfs-test ./nfs-test-chart \
  --set nfs.server=192.168.1.100 \
  --set nfs.path=/exports/custom/path \
  --set pvc.storage=5Gi \
  --set pod.image=registry.access.redhat.com/ubi9/ubi:latest \
  --set namespace=bell-test \
  --wait
```

### Using a Values File

Create a custom values file:

```yaml
# custom-values.yaml
nfs:
  server: "192.168.1.100"
  path: "/exports/custom/path"

pvc:
  storage: "5Gi"

pod:
  image: "registry.access.redhat.com/ubi9/ubi:latest"
  mountPath: "/data"
```

Deploy with custom values:

```bash
helm upgrade --install nfs-test ./nfs-test-chart \
  -f custom-values.yaml \
  --set namespace=bell-test \
  --wait
```

---

## Next Steps

- If deployment fails, see [TROUBLESHOOTING-GUIDE.md](TROUBLESHOOTING-GUIDE.md)
- For production use, consider adding monitoring and alerting
- Review logs regularly to ensure NFS connectivity remains stable