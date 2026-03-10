# OpenShift ISO Share

Deploy an nginx-based file server with a PersistentVolumeClaim to store and share ISO images via HTTP.

## Quick Start

```bash
# Create a new project (or use existing)
oc new-project iso-share

# Deploy all resources
oc apply -k .

# Get the share URL (file server) and HTML proxy URL
oc get route iso-share
oc get route html-proxy
```

## Uploading an ISO

After the pod is running, copy your ISO to the PVC:

```bash
# Get the pod name
POD=$(oc get pods -l component=iso-share -o jsonpath='{.items[0].metadata.name}')

# Upload your ISO
oc cp /path/to/your-image.iso $POD:/usr/share/nginx/html/

# Or with custom image: rsync for large files (resumable)
oc rsync /path/to/your-image.iso $POD:/usr/share/nginx/html/
```

> **Note:** `oc rsync` requires the [custom image with rsync](#optional-custom-image-with-rsync). Use `oc cp` with the default image.

## Accessing the ISO

1. Get the Route hostname: `oc get route iso-share -o jsonpath='{.spec.host}'`
2. Open `https://<hostname>/` to see the directory listing
3. Click your ISO filename to download, or use: `https://<hostname>/your-image.iso`

### Large files (97 MB chunks)

For files larger than 97 MB, the listing page shows:

- **Chunks (97 MB)**: separate links for each 97 MB part (Part aa, Part ab, …) so you can download or resume in segments.
- **Full file**: a single link that streams the entire file (server reads in 97 MB chunks, so you get one continuous download).

Use chunks when you want to download in parts or combine them later; use **Full file** for one direct download.

**Download all chunks (one click):** For chunked files, the **Download all** button fetches each part in order, combines them in memory, and triggers a single download with the original filename. You get one save dialog and one file—no special browser permissions. For very large files this uses RAM equal to the file size; if the tab runs out of memory, use the individual **Part aa**, **Part ab**, … links and combine them locally (e.g. `cat file.iso.part* > file.iso`). Chunk files use the suffix `.partaa`, `.partab`, `.partac`, … so they sort correctly when concatenating.

## Backing up the ISO

### Copy from the pod to your machine (`oc cp`)

```bash
POD=$(oc get pods -l component=iso-share -o jsonpath='{.items[0].metadata.name}')
oc cp $POD:/usr/share/nginx/html/your-image.iso ./backup-your-image.iso
```

### Copy directory with rsync (`oc rsync`)

If you built the [custom image with rsync](#optional-custom-image-with-rsync), `oc rsync` supports resumable transfers. Otherwise use `oc cp`:

```bash
POD=$(oc get pods -l component=iso-share -o jsonpath='{.items[0].metadata.name}')
# With custom image (rsync - resumable):
oc rsync $POD:/usr/share/nginx/html/ ./local-backup-dir/
# With default image (oc cp):
oc cp $POD:/usr/share/nginx/html/ ./local-backup-dir/
```

### Download via the Route (HTTP)

If you have network access to the Route:

```bash
ROUTE=$(oc get route iso-share -o jsonpath='{.spec.host}')
curl -k -o backup-your-image.iso "https://$ROUTE/your-image.iso"
```

### PVC snapshot

For a volume-level backup, create a VolumeSnapshot (requires a StorageClass that supports snapshots, e.g. OpenShift Data Foundation). Use the OpenShift Console: **Storage → Persistent Volume Claims →** select `iso-storage` **→ Actions → Create Snapshot**. Or apply a `VolumeSnapshot` manifest for your storage provider.

## HTML proxy (browser)

A separate pod runs a **browser-based web proxy** on port 8080. Open the proxy URL in a browser, enter any URL in the form, and view the site in the same page (proxied in an iframe).

- **Route**: `oc get route html-proxy -o jsonpath='{.spec.host}'` — open `https://<host>/` in a browser.
- **In-cluster**: Service `html-proxy:8080` — use `http://html-proxy.<namespace>.svc.cluster.local:8080` from other pods.

The proxy runs a small Python server (non-root) that serves the browser UI and fetches URLs for the iframe.

## Optional: Custom image with rsync

The default nginx image does not include rsync. To enable `oc rsync` for resumable uploads and backups, build a custom image:

```bash
# From the repo root, start the binary build (uploads Dockerfile + context)
oc start-build iso-share --from-dir=. --follow

# Update the deployment to use the custom image
oc set image deployment/iso-share nginx=iso-share:latest

# Wait for rollout
oc rollout status deployment/iso-share
```

After this, `oc rsync` works for both uploads and backups.

## Customization

- **Storage size**: Edit `pvc.yaml` and change `storage: 10Gi` to your needs
- **Route hostname**: Add `spec.host: your-custom-name.apps.example.com` to `route.yaml` for a custom URL
- **Red Hat image**: If your cluster restricts external images, change the deployment to use `registry.access.redhat.com/ubi9/nginx-126:latest` (adjust mount path to `/opt/app-root/etc/nginx.d/` if needed)
