# Dynamic NFS Provisioning with nfs-csi StorageClass

This document describes the dynamic NFS provisioning approach using the `nfs-csi` StorageClass in OpenShift.

## Overview

This approach uses OpenShift's NFS CSI driver to dynamically provision PersistentVolumes from an NFS export. The CSI driver automatically creates subdirectories for each PVC, eliminating the need for manual PV creation.

## Architecture

```
NFS Server (192.168.1.100)
└── /exports/ndex/data/
    ├── pvc-<uuid-1>/  (auto-created by CSI driver)
    ├── pvc-<uuid-2>/  (auto-created by CSI driver)
    └── ...

OpenShift Cluster
├── StorageClass: nfs-csi (pre-configured)
├── PVC: nfs-dynamic-pvc (requests storage)
│   └── Bound to: pvc-<uuid> (auto-created PV)
└── Pod: nfs-dynamic-test-pod
    └── Mounts: /mnt/nfs → pvc-<uuid>
```

## Advantages

✅ **No cluster-admin required** - Regular users can create PVCs  
✅ **Automatic PV creation** - CSI driver handles PV lifecycle  
✅ **No PV claimRef issues** - Each PVC gets a unique subdirectory  
✅ **Multi-tenant friendly** - Isolated storage per PVC  
✅ **Works with restricted pod security** - No privileged containers needed

## Prerequisites

### 1. NFS Server Configuration

The NFS server must be configured to allow OpenShift pods to write to the export.

#### NFS Export Configuration

Edit `/etc/exports` on the NFS server:

```bash
sudo nano /etc/exports
```

Add or update the export with these options:

```
/exports/ndex/data *(rw,sync,no_subtree_check,no_root_squash,insecure,all_squash,anonuid=1000650000,anongid=1000650000)
```

**Option explanations:**
- `rw` - Read-write access
- `sync` - Synchronous writes (safer but slower)
- `no_subtree_check` - Improves reliability
- `no_root_squash` - Allows root on client to be root on server
- `insecure` - Allows connections from ports > 1024 (required for OpenShift)
- `all_squash` - Maps all client UIDs to anonymous UID
- `anonuid=1000650000` - Anonymous UID (OpenShift's default UID range)
- `anongid=1000650000` - Anonymous GID

#### Directory Permissions

Set appropriate permissions on the NFS export:

```bash
# Make the export directory writable by all
sudo chmod -R 777 /exports/ndex/data/

# Ensure all subdirectories are writable
sudo find /exports/ndex/data -type d -exec chmod 777 {} \;
```

#### Apply Changes

```bash
# Reload NFS exports
sudo exportfs -ra

# Verify exports
sudo exportfs -v

# Restart NFS server (if needed)
sudo systemctl restart nfs-server
```

### 2. OpenShift StorageClass

Verify the `nfs-csi` StorageClass exists:

```bash
oc get storageclass nfs-csi -o yaml
```

Expected output:
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
provisioner: nfs.csi.k8s.io
parameters:
  server: "192.168.1.100"
  share: "/exports/ndex/data"
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

If it doesn't exist, create it:

```bash
cat <<EOF | oc apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
provisioner: nfs.csi.k8s.io
parameters:
  server: "192.168.1.100"
  share: "/exports/ndex/data"
reclaimPolicy: Delete
volumeBindingMode: Immediate
EOF
```

## Helm Chart Configuration

### values.yaml

```yaml
namespace: bell-test

nfs:
  server: "192.168.1.100"
  path: "/exports/ndex/data"

serviceAccount:
  name: nfs-sa

pvc:
  name: nfs-dynamic-pvc
  storageClassName: nfs-csi
  accessModes:
    - ReadWriteMany
  storage: 1Gi

pod:
  name: nfs-dynamic-test-pod
  image: registry.access.redhat.com/ubi8/ubi:latest
  mountPath: /mnt/nfs
```

### PVC Template (templates/pvc.yaml)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: {{ .Values.pvc.name }}
  namespace: {{ .Values.namespace }}
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: {{ .Values.pvc.storageClassName }}
  resources:
    requests:
      storage: {{ .Values.pvc.storage }}
```

### Pod Template (templates/pod.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ .Values.pod.name }}
  namespace: {{ .Values.namespace }}
spec:
  serviceAccountName: {{ .Values.serviceAccount.name }}
  restartPolicy: Never
  containers:
  - name: nfs-test
    image: {{ .Values.pod.image }}
    command: ["/bin/bash", "-c"]
    args:
    - |
      echo "Testing NFS mount..."
      echo "Test data" > {{ .Values.pod.mountPath }}/test.txt
      cat {{ .Values.pod.mountPath }}/test.txt
      echo "Success!"
      sleep 3600
    volumeMounts:
    - name: nfs-volume
      mountPath: {{ .Values.pod.mountPath }}
  volumes:
  - name: nfs-volume
    persistentVolumeClaim:
      claimName: {{ .Values.pvc.name }}
```

## Deployment

### Using Helm

```bash
# Install or upgrade
helm upgrade --install nfs-test ./nfs-test-chart \
  --set namespace=bell-test \
  --wait

# Verify deployment
oc get pvc -n bell-test
oc get pod -n bell-test
oc logs nfs-dynamic-test-pod -n bell-test
```

### Using GitHub Actions

The workflow automatically deploys when changes are pushed to the `dynamic-nfs-provisioning` branch:

```yaml
on:
  push:
    branches:
      - dynamic-nfs-provisioning
    paths:
      - 'nfs-test-chart/**'
  workflow_dispatch:
    inputs:
      action:
        description: 'Action to perform'
        required: true
        default: 'deploy'
        type: choice
        options:
          - deploy
          - teardown
```

## Verification

### Check PVC Status

```bash
oc get pvc nfs-dynamic-pvc -n bell-test
```

Expected output:
```
NAME                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS
nfs-dynamic-pvc     Bound    pvc-edcd588a-3452-4a09-84f4-cba9c7f41530   1Gi        RWX            nfs-csi
```

### Check Pod Status

```bash
oc get pod nfs-dynamic-test-pod -n bell-test
```

Expected output:
```
NAME                       READY   STATUS    RESTARTS   AGE
nfs-dynamic-test-pod       1/1     Running   0          2m
```

### Check Pod Logs

```bash
oc logs nfs-dynamic-test-pod -n bell-test
```

Expected output:
```
=== NFS Mount Test Started ===
Mount point: /mnt/nfs
=== Pod User Information ===
uid=1000650000(1000650000) gid=0(root) groups=0(root),1000650000
=== Mount Information ===
172.24.127.64:/exports/ndex/data/pvc-<uuid> on /mnt/nfs type nfs4 (...)
=== Directory Permissions ===
drwxrwsr-x. 2 1000650000 1000650000 4096 ...
=== Testing write access ===
✓ Write test successful: /mnt/nfs/test-1777589372.txt
=== Testing read access ===
Test data written at Thu Apr 30 22:49:32 UTC 2026
✓ Read test successful
=== NFS Mount Test Completed ===
```

### Verify on NFS Server

```bash
# Check the auto-created PVC directory
ls -la /exports/ndex/data/

# Should show directories like:
drwxrwxrwx 3 1000650000 1000650000 4096 ... pvc-edcd588a-3452-4a09-84f4-cba9c7f41530/
```

## Troubleshooting

### Permission Denied Errors

If you see "Permission denied" when writing to the NFS mount:

1. **Check NFS export configuration:**
   ```bash
   cat /etc/exports
   # Should include: all_squash,anonuid=1000650000,anongid=1000650000
   ```

2. **Check directory permissions:**
   ```bash
   ls -la /exports/ndex/data/
   # Should be 777 or owned by 1000650000
   ```

3. **Check pod UID:**
   ```bash
   oc exec nfs-dynamic-test-pod -n bell-test -- id
   # Should show uid=1000650000
   ```

4. **Fix permissions:**
   ```bash
   sudo chmod -R 777 /exports/ndex/data/
   sudo exportfs -ra
   ```

### PVC Stuck in Pending

If the PVC remains in "Pending" state:

1. **Check StorageClass exists:**
   ```bash
   oc get storageclass nfs-csi
   ```

2. **Check CSI driver pods:**
   ```bash
   oc get pods -n openshift-cluster-csi-drivers | grep nfs
   ```

3. **Check PVC events:**
   ```bash
   oc describe pvc nfs-dynamic-pvc -n bell-test
   ```

### Pod CrashLoopBackOff

If the pod keeps restarting:

1. **Check pod logs:**
   ```bash
   oc logs nfs-dynamic-test-pod -n bell-test
   ```

2. **Check mount status:**
   ```bash
   oc exec nfs-dynamic-test-pod -n bell-test -- mount | grep nfs
   ```

3. **Verify NFS server is accessible:**
   ```bash
   showmount -e 192.168.1.100
   ```

## Cleanup

### Delete Resources

```bash
# Delete pod
oc delete pod nfs-dynamic-test-pod -n bell-test

# Delete PVC (this also deletes the auto-created PV)
oc delete pvc nfs-dynamic-pvc -n bell-test

# Delete service account
oc delete sa nfs-sa -n bell-test
```

### Using Helm

```bash
helm uninstall nfs-test -n bell-test
```

### Clean NFS Server

The CSI driver does NOT automatically delete the PVC subdirectory on the NFS server. You must manually clean it:

```bash
# On NFS server
sudo rm -rf /exports/ndex/data/pvc-*
```

## Comparison with Other Approaches

| Feature | Dynamic Provisioning | Static PV/PVC | CSI Inline Volume |
|---------|---------------------|---------------|-------------------|
| Cluster-admin required | ❌ No | ✅ Yes | ❌ No |
| PV management | Automatic | Manual | None |
| Multi-tenant | ✅ Yes | ⚠️ Limited | ⚠️ Limited |
| Pod security | Restricted | Restricted | Privileged |
| Cleanup complexity | Medium | High | Low |
| **Recommended for** | **Production** | Legacy | Testing |

## Best Practices

1. **Use dynamic provisioning for production** - Easier to manage and more secure
2. **Configure NFS export with all_squash** - Ensures consistent permissions
3. **Set directory permissions to 777** - Allows all pods to write
4. **Monitor PVC usage** - Clean up unused PVCs to free NFS space
5. **Use meaningful PVC names** - Helps identify resources on NFS server
6. **Document UID ranges** - Know what UIDs your pods run as
7. **Test permissions before deployment** - Verify write access works

## Security Considerations

- **all_squash with anonuid** maps all users to a single UID, reducing security isolation
- **777 permissions** allow any user to read/write, suitable for testing but not production
- For production, consider:
  - Using separate NFS exports per namespace
  - Implementing proper UID/GID mapping
  - Using network policies to restrict NFS access
  - Enabling NFS Kerberos authentication

## References

- [OpenShift NFS CSI Driver Documentation](https://docs.openshift.com/container-platform/latest/storage/container_storage_interface/persistent-storage-csi-nfs.html)
- [Kubernetes CSI Documentation](https://kubernetes-csi.github.io/docs/)
- [NFS Server Configuration](https://linux.die.net/man/5/exports)

---

**Branch:** `dynamic-nfs-provisioning`  
**Last Updated:** 2026-04-30  
**Status:** ✅ Working - Write/Read tests passing