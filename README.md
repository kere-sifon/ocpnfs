# NFS Test Chart for OpenShift

This repository contains a Helm chart and GitHub Actions workflow for testing NFS PersistentVolume mounting on a local OpenShift (CRC) cluster.

## Project Structure

```
.
├── .github/
│   └── workflows/
│       └── nfs-test-deploy.yml    # GitHub Actions workflow
└── nfs-test-chart/
    ├── Chart.yaml                  # Helm chart metadata
    ├── values.yaml                 # Configuration values
    └── templates/
        ├── namespace.yaml          # Namespace definition
        ├── serviceaccount.yaml     # Service account for NFS access
        ├── pv.yaml                 # NFS PersistentVolume
        ├── pvc.yaml                # PersistentVolumeClaim
        └── pod.yaml                # Test pod with NFS mount
```

## Prerequisites

- OpenShift CRC cluster running locally
- Self-hosted GitHub Actions runner configured
- Helm 3.x installed on the runner
- NFS server accessible from the cluster

## GitHub Secrets Configuration

Configure the following secrets in your GitHub repository:

- `OCP_KUBEADMIN_PASSWORD`: OpenShift kubeadmin password
- `NFS_SERVER_IP`: IP address of your NFS server

## Default Configuration

- **Namespace**: `nfs-test`
- **NFS Path**: `/exports/ndex/data`
- **Storage**: `1Gi`
- **Pod Image**: `registry.access.redhat.com/ubi8/ubi:latest`
- **Mount Path**: `/mnt/nfs`

## Usage

### Deploy via GitHub Actions

1. **Automatic deployment**: Push changes to the `main` branch in the `nfs-test-chart/` directory
2. **Manual deployment**: Go to Actions → NFS Test Deploy → Run workflow → Select "deploy"

### Manual Deployment

```bash
# Login to OpenShift
oc login https://api.crc.testing:6443 --username=kubeadmin --insecure-skip-tls-verify=true

# Apply SCC to service account
oc adm policy add-scc-to-user hostmount-anyuid -z nfs-sa -n nfs-test

# Install the Helm chart
helm upgrade --install nfs-test ./nfs-test-chart \
  --set nfs.server=<NFS_SERVER_IP> \
  --create-namespace \
  --wait
```

### Teardown

**Via GitHub Actions**: Run workflow → Select "teardown"

**Manual**:
```bash
helm uninstall nfs-test -n nfs-test
oc delete pv nfs-pv
oc delete namespace nfs-test
```

## What the Test Does

The test pod performs the following operations:

1. Mounts the NFS volume at `/mnt/nfs`
2. Writes a test file with timestamp
3. Reads back the test file to verify access
4. Lists directory contents
5. Sleeps for 3600 seconds (1 hour) to allow inspection

## Viewing Test Results

```bash
# Check pod status
oc get pod nfs-test-pod -n nfs-test

# View pod logs
oc logs nfs-test-pod -n nfs-test

# Check PV/PVC status
oc get pv,pvc -n nfs-test
```

## Customization

Edit `nfs-test-chart/values.yaml` to customize:

- NFS server and path
- Storage size
- Pod image
- Mount path
- Resource names

## Troubleshooting

### Pod fails to start
- Check if NFS server is accessible: `ping <NFS_SERVER_IP>`
- Verify NFS export is configured correctly
- Check SCC permissions: `oc get scc hostmount-anyuid -o yaml`

### Permission denied errors
- Ensure the service account has the correct SCC applied
- Verify NFS export permissions allow the pod's UID/GID

### PVC remains in Pending state
- Check PV status: `oc get pv nfs-pv`
- Verify NFS server and path in values.yaml
- Check events: `oc get events -n nfs-test`

## License

MIT