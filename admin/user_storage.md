# User Storage

## Overview

We provide users with 10G of base user storage.  This is managed by our deployment of the [`jupyter-home-nfs` Helm chart](https://github.com/2i2c-org/jupyterhub-home-nfs), which is located in the `jupyterhub-home-nfs` [subdirectory in the `cal-icor-hubs` repo](https://github.com/cal-icor/cal-icor-hubs/tree/staging/jupyterhub-home-nfs).

This chart deploys a pod named `home-nfs-<hash>-<hash>` in the `jupyterhub-home-nfs` namespace, with four containers running in total (`nfs-server`, `enforce-xfs-quota`, `auto-xfs-resizer` and `node-exporter`).  The backing disk mounted by NFS is defined in `values.yaml`.

Any time a PR is opened to update something in that directory, Github Actions will add the `jupyterhub-home-nfs-deployment` label and, when merged to `staging`, will automatically deploy the chart via the [`deploy-jupyterhub-home-nfs.yaml` workflow](https://github.com/cal-icor/cal-icor-hubs/blob/staging/.github/workflows/deploy-jupyterhub-home-nfs.yaml).

## Updating the Chart version

If you need to update the published version of the Helm chart, find the version you'd like to deploy in the [container registry](https://github.com/2i2c-org/jupyterhub-home-nfs/pkgs/container/jupyterhub-home-nfs%2Fjupyterhub-home-nfs), and then update the chart version in [`jupyterhub-home-nfs/Chart.yaml`](https://github.com/cal-icor/cal-icor-hubs/blob/staging/jupyterhub-home-nfs/Chart.yaml#L8).  Create a PR and merge to `staging` to deploy!

## Changing directory quotas and excluding directories from the quota enforcer

To change the overall storage quota for all users, you simply update the `hard_quota` in [`jupyterhub-home-nfs/values.yaml`](https://github.com/cal-icor/cal-icor-hubs/blob/staging/jupyterhub-home-nfs/values.yaml#L44), create a PR and then merge to `staging` to deploy.

If you need to give a specific user a different quota, there are a few additional steps to perform.

1. Get the user's jupyterhub user ID (**not** their email address!).  This will look something like `jupyter-username-host-edu---<hash>`.  This can be found by finding their user in `k9s`, via `kubectl` or sshing in to the `nfs-server` container and [checking the NFS mount directly](#shell-access-to-the-nfs-server).  They will need to have logged in at least once before for this to be created.
2. Using [sops](https://github.com/mozilla/sops/releases), edit [`jupyterhub-home-nfs/secrets/quota-overrides.yaml`] and add the login ID and desired quota to the dict.
3. Save your changes, create a PR and deploy the chart on a merge to `staging`.

Sometimes you need the quota enforcer to ignore a directory (or directories).  Typically, this is just for common folders like `_shared`, but could potentially be used for one-off and short term cases.  Just add another element to the `exclude` list in [`jupyterhub-home-nfs/values.yaml`](https://github.com/cal-icor/cal-icor-hubs/blob/staging/jupyterhub-home-nfs/values.yaml#L47).

## NFS diagnostics, troubleshooting and admin tasks

### Disk usage, monitoring and alerting

There is a policy named `NFS home-dir disk >=90% full (jupyterhub-homedirs-2026-04-08)` that will send alerts to both email and PagerDuty when the underlying disk hit 90% capacity.  You should check the [Grafana Home Directory Usage](https://grafana.jupyter.cal-icor.org/d/e2984e42-dfa8-4155-a956-a5bc1c710d3f/home-directory-usage-dashboard?orgId=1&from=now-6h&to=now&timezone=utc&var-PROMETHEUS_DS=P1809F7CD0C75ACF3&var-hub=$__all) panel to see if this is a shared directory, or [shell in to the NFS server](#shell-access-to-the-nfs-server) and look around in `/export` to find the offending folder.

If it's just due to the general growth of data stored on the drive, you can grow the underlying disk with the following command:

``` bash
gcloud compute disks resize jupyterhub-homedirs-2026-04-08 --zone=us-central1-b --size=<new-GB>
```

The auto-xfs-resizer sidecar runs `xfs_growfs /export` every 5 min and expands XFS to the new size automatically.

### Shell access to the NFS server

If you need to connect to the NFS server to debug a user's filesystem or perform any tasks, there are two methods to get a shell on the `nfs-server` container:

1. Using `k9s`, select the `home-nfs-<hash>-<hash>` pod and hit enter. Then select the `nfs-server` container and hit `s`.
2. Run the following `kubectl` command:

``` bash
kubectl exec -it -n jupyterhub-home-nfs $(kubectl get pod -n jupyterhub-home-nfs -l app=nfs-server -o jsonpath='{.items[0].metadata.name}') -- /bin/bash
```

The NFS mounts are all located under `/export/<hubname>` on the NFS server.

### Convenience packages to install on a fresh `jupyterhub-home-nfs` deployment

The first time you deploy this, or after any subsequent `helm upgrades`, the `nfs-server` container will be missing a bunch of useful system tools.  It's strongly recommended that immediately after deployment, that you log in to the `nfs-server` shell, and run the following command:

``` bash
apt-get update && apt-get install -y vim screen rsync less tree
```

### Changing the size of the backing disk

If you need to grow the size of the disk that stores the user homedirs, there are two ways to do that:

1. Through the GCP console:  Compute Engine -> Disks -> jupyterhub-homedirs-2026-04-08 -> edit
2. Running the following `gcloud` command, replacing `SIZE_GB` with the new size of the disk:

``` bash
  gcloud compute disks resize jupyterhub-homedirs-2026-04-08 \
    --size SIZE_GB \
    --zone us-central1-b \
    --project cal-icor-hubs
```
