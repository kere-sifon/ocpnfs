# NFS Test Troubleshooting Guide

This guide covers common issues encountered during setup and deployment, based on real implementation experiences.

## Table of Contents
1. [Namespace Issues](#namespace-issues)
2. [PersistentVolume Issues](#persistentvolume-issues)
3. [PersistentVolumeClaim Issues](#persistentvolumeclaim-issues)
4. [Pod Issues](#pod-issues)
5. [Authentication Issues](#authentication-issues)
6. [GitHub Actions Issues](#github-actions-issues)
7. [NFS Connectivity Issues](#nfs-connectivity-issues)

---

## Namespace Issues

### Error: Namespace exists and cannot be imported

**Symptom:**
```
Error: Unable to continue with install: Namespace "bell-test" in namespace "" exists 
and cannot be imported into the current release: invalid ownership metadata
```

**Cause:** The namespace already exists but wasn't created by Helm, so it lacks Helm labels/annotations.

**Solution:** Remove the namespace template from the Helm chart since the namespace is pre-created.

```bash
# The namespace template has been removed from this chart
# Namespace must exist before deployment
oc get namespace bell-test
```

**Prevention:** Always use pre-existing namespaces managed by the platform team.

---

## PersistentVolume Issues

### Error: User cannot get PersistentVolumes

**Symptom:**
```
Error: could not get information about the resource PersistentVolume "nfs-pv" in namespace "": 
persistentvolumes "nfs-pv" is forbidden: User "developer" cannot get resource 
"persistentvolumes" in API group "" at the cluster scope
```

**Cause:** Regular users don't have permissions to create cluster-scoped PersistentVolumes.

**Solution:** PV must be pre-created by a cluster administrator.

```bash
# Cluster admin creates PV
oc apply -f nfs-pv.yaml

# Verify PV exists
oc get pv ndex-nfs-pv
```

**Prevention:** Always have cluster admin create PVs before deployment. See [SETUP-GUIDE.md](SETUP-GUIDE.md).

### PV Shows "Available" but PVC Won't Bind

**Symptom:**
```
NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS
ndex-nfs-pv   1Gi        RWX            Retain           Available           
```

**Cause:** Usually a mismatch in:
- Storage size
- Access modes
- Storage class name

**Diagnosis:**
```bash
# Check PV details
oc get pv ndex-nfs-pv -o yaml

# Check PVC details
oc describe pvc nfs-pvc -n bell-test
```

**Solution:** See PVC issues below.

---

## PersistentVolumeClaim Issues

### Error: storageClassName does not match

**Symptom:**
```
Events:
  Warning  VolumeMismatch  Cannot bind to requested volume "ndex-nfs-pv": 
  storageClassName does not match
```

**Cause:** PV has no `storageClassName` (empty), but PVC is trying to use a storage class.

**Solution:** Set PVC's `storageClassName` to empty string.

```yaml
# In templates/pvc.yaml
spec:
  storageClassName: ""  # Must match PV (empty)
  volumeName: ndex-nfs-pv
```

**This has been fixed in the current chart.**

### Error: PVC spec is immutable

**Symptom:**
```
Error: UPGRADE FAILED: cannot patch "nfs-pvc" with kind PersistentVolumeClaim: 
PersistentVolumeClaim "nfs-pvc" is invalid: spec: Forbidden: spec is immutable 
after creation except resources.requests and volumeAttributesClassName for bound claims
```

**Cause:** Trying to change PVC spec (like storageClassName) after it's been created.

**Solution:** Delete and recreate the PVC.

```bash
# Delete pod first (it's using the PVC)
oc delete pod nfs-test-pod -n bell-test

# Delete PVC
oc delete pvc nfs-pvc -n bell-test

# Redeploy
helm upgrade --install nfs-test ./nfs-test-chart \
  --set nfs.server=<NFS_IP> \
  --set namespace=bell-test \
  --wait
```

**The workflow now includes an automatic cleanup step to handle this.**

### PVC Stuck in Pending

**Symptom:**
```
NAME      STATUS    VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS
nfs-pvc   Pending   ndex-nfs-pv   0                         
```

**Diagnosis:**
```bash
oc describe pvc nfs-pvc -n bell-test
```

**Common causes and solutions:**

1. **Volume name mismatch:**
   ```bash
   # Check PV name
   oc get pv
   # Update values.yaml if needed
   ```

2. **Storage size mismatch:**
   ```bash
   # PVC requests more than PV provides
   # Update pvc.storage in values.yaml
   ```

3. **Access mode mismatch:**
   ```bash
   # PV must support ReadWriteMany (RWX)
   oc get pv ndex-nfs-pv -o yaml | grep accessModes
   ```

4. **PV already bound:**
   ```bash
   # Check if PV is bound to another PVC
   oc get pv ndex-nfs-pv
   # If bound, delete the old PVC first
   ```

---

## Pod Issues

### Pod Stuck in Pending

**Symptom:**
```
NAME           READY   STATUS    RESTARTS   AGE
nfs-test-pod   0/1     Pending   0          5m
```

**Diagnosis:**
```bash
oc describe pod nfs-test-pod -n bell-test
```

**Common causes:**

1. **Unbound PVC:**
   ```
   Events: pod has unbound immediate PersistentVolumeClaims
   ```
   Solution: Fix PVC binding (see PVC issues above)

2. **No nodes available:**
   ```
   Events: 0/1 nodes are available: insufficient resources
   ```
   Solution: Check cluster resources or node selectors

3. **Image pull errors:**
   ```
   Events: Failed to pull image
   ```
   Solution: Verify image name and registry access

### Pod CrashLoopBackOff

**Symptom:**
```
NAME           READY   STATUS             RESTARTS   AGE
nfs-test-pod   0/1     CrashLoopBackOff   5          10m
```

**Diagnosis:**
```bash
# Check pod logs
oc logs nfs-test-pod -n bell-test

# Check previous logs if pod restarted
oc logs nfs-test-pod -n bell-test --previous
```

**Common causes:**

1. **NFS mount failure:**
   ```
   mount.nfs: Connection refused
   ```
   Solution: Check NFS server connectivity and export configuration

2. **Permission denied:**
   ```
   mount.nfs: access denied by server
   ```
   Solution: Check NFS export permissions and SCC assignment

3. **Script errors:**
   Check logs for bash script errors in the test commands

### Pod Shows Error Status

**Symptom:**
```
NAME           READY   STATUS   RESTARTS   AGE
nfs-test-pod   0/1     Error    0          2m
```

**Diagnosis:**
```bash
oc logs nfs-test-pod -n bell-test
oc describe pod nfs-test-pod -n bell-test
```

**Solution:** Review logs for specific error messages and fix accordingly.

### Permission Denied on NFS Mount

**Symptom:**
```
mount.nfs: access denied by server while mounting
```

**Cause:** Missing SCC permissions or NFS export restrictions.

**Solution:**

1. **Verify SCC assignment:**
   ```bash
   oc get scc hostmount-anyuid -o yaml | grep "system:serviceaccount:bell-test:nfs-sa"
   ```

2. **Request SCC from platform team:**
   ```bash
   # Cluster admin runs:
   oc adm policy add-scc-to-user hostmount-anyuid -z nfs-sa -n bell-test
   ```

3. **Check NFS export permissions:**
   ```bash
   # On NFS server
   cat /etc/exports
   # Should allow access from OpenShift nodes
   ```

---

## Authentication Issues

### Error: Unauthorized

**Symptom:**
```
Error: Unauthorized
```

**Cause:** Invalid or expired OpenShift token.

**Solution:**

1. **Generate new token:**
   ```bash
   oc sa get-token <service-account-name> -n bell-test
   ```

2. **Update GitHub secret:**
   - Go to Settings → Secrets → Actions
   - Update `OPENSHIFT_TOKEN` with new token

3. **Verify token works:**
   ```bash
   oc login --token=<NEW_TOKEN> --server=<SERVER_URL>
   ```

### Error: Forbidden

**Symptom:**
```
Error: Forbidden: User cannot perform action
```

**Cause:** Service account lacks necessary permissions.

**Solution:**

```bash
# Grant edit role to service account
oc policy add-role-to-user edit system:serviceaccount:bell-test:github-deployer -n bell-test

# Verify permissions
oc auth can-i create pods --as=system:serviceaccount:bell-test:github-deployer -n bell-test
```

---

## GitHub Actions Issues

### Runner Not Found

**Symptom:**
```
Waiting for a runner to pick up this job...
```

**Cause:** No runner with label `oc-run` is available.

**Solution:**

1. **Check runner status:**
   - Go to Settings → Actions → Runners
   - Verify runner is online and has label `oc-run`

2. **Restart runner service:**
   ```bash
   # On runner machine
   cd ~/actions-runner
   sudo ./svc.sh stop
   sudo ./svc.sh start
   sudo ./svc.sh status
   ```

3. **Check runner logs:**
   ```bash
   # On runner machine
   tail -f ~/actions-runner/_diag/Runner_*.log
   ```

### Workflow Fails at Helm Step

**Symptom:**
```
Error: Helm not found
```

**Cause:** Helm not installed on runner or not in PATH.

**Solution:**

```bash
# On runner machine
# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify installation
helm version

# Add to PATH if needed
export PATH=$PATH:/usr/local/bin
```

### Workflow Fails at OC Login

**Symptom:**
```
Error: oc: command not found
```

**Cause:** OpenShift CLI not installed on runner.

**Solution:**

```bash
# On runner machine
# Install oc CLI
curl -LO https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable/openshift-client-linux.tar.gz
tar xvf openshift-client-linux.tar.gz
sudo mv oc kubectl /usr/local/bin/

# Verify installation
oc version
```

---

## NFS Connectivity Issues

### Cannot Reach NFS Server

**Symptom:**
```
mount.nfs: Connection timed out
```

**Diagnosis:**

```bash
# From runner or OpenShift node
ping <NFS_SERVER_IP>

# Test NFS port
nc -zv <NFS_SERVER_IP> 2049

# Check NFS exports
showmount -e <NFS_SERVER_IP>
```

**Solutions:**

1. **Firewall blocking:**
   ```bash
   # On NFS server, allow NFS ports
   sudo firewall-cmd --permanent --add-service=nfs
   sudo firewall-cmd --permanent --add-service=rpc-bind
   sudo firewall-cmd --permanent --add-service=mountd
   sudo firewall-cmd --reload
   ```

2. **Network routing:**
   - Verify OpenShift nodes can reach NFS server network
   - Check network policies in OpenShift

3. **NFS service not running:**
   ```bash
   # On NFS server
   sudo systemctl status nfs-server
   sudo systemctl start nfs-server
   sudo systemctl enable nfs-server
   ```

### NFS Export Not Accessible

**Symptom:**
```
mount.nfs: access denied by server
```

**Solution:**

```bash
# On NFS server
# Check exports file
cat /etc/exports

# Should contain something like:
# /exports/ndex/data *(rw,sync,no_root_squash,no_subtree_check)

# Reload exports
sudo exportfs -ra

# Verify exports
showmount -e localhost
```

### Stale NFS Handle

**Symptom:**
```
Stale NFS file handle
```

**Solution:**

```bash
# Delete pod to unmount
oc delete pod nfs-test-pod -n bell-test

# On NFS server, restart NFS
sudo systemctl restart nfs-server

# Redeploy
helm upgrade --install nfs-test ./nfs-test-chart --set nfs.server=<IP> --set namespace=bell-test --wait
```

---

## Getting Help

### Collect Diagnostic Information

When reporting issues, collect this information:

```bash
# Pod status
oc get pod nfs-test-pod -n bell-test -o yaml > pod.yaml

# Pod logs
oc logs nfs-test-pod -n bell-test > pod.log

# PVC status
oc get pvc nfs-pvc -n bell-test -o yaml > pvc.yaml

# PV status
oc get pv ndex-nfs-pv -o yaml > pv.yaml

# Events
oc get events -n bell-test --sort-by='.lastTimestamp' > events.log

# Describe pod
oc describe pod nfs-test-pod -n bell-test > pod-describe.txt
```

### Enable Debug Mode

Add debug output to the pod:

```yaml
# In templates/pod.yaml, add to command:
- set -x  # Enable bash debug mode
```

### Check Helm Release Status

```bash
# List releases
helm list -n bell-test

# Get release details
helm get all nfs-test -n bell-test

# Check release history
helm history nfs-test -n bell-test
```

---

## Prevention Best Practices

1. **Always verify prerequisites** before deployment
2. **Use the cleanup step** in the workflow to avoid immutable spec errors
3. **Test NFS connectivity** before creating PV
4. **Keep tokens secure** and rotate regularly
5. **Monitor runner health** and logs
6. **Document any custom configurations** for your environment
7. **Test in non-production** first

---

## Additional Resources

- [SETUP-GUIDE.md](SETUP-GUIDE.md) - Initial setup instructions
- [DEPLOYMENT-GUIDE.md](DEPLOYMENT-GUIDE.md) - Deployment procedures
- [OpenShift Documentation](https://docs.openshift.com/)
- [Helm Documentation](https://helm.sh/docs/)
- [NFS Documentation](https://linux.die.net/man/5/nfs)