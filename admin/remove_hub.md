# Removing an Existing Hub Deployment

Sometimes an institution ends its partnership with us, or never uses the hub it
signed up for. Audit the deployments at the end of every term and remove any
idle hub you're certain nobody will use again.

The `remove_deployment.sh` script in `cal-icor-hubs` does almost all of the
work. The [manual steps](#removing-a-hub-by-hand) further down cover the same
ground, for when you need to redo one piece by hand.

## Prerequisites

### Software Packages and Authentication

You will need the basic set of admin tooling and authentication set up as
described in the [Creating a New Hub](new_hub) document.

The script also needs `kubectl` pointed at the
`gke_cal-icor-hubs_us-central1_spring-2025` context and `gcloud` set to the
`cal-icor-hubs` project.

## Run the script

From the root of `cal-icor-hubs`, on an up-to-date `staging` branch with a
clean working tree:

``` bash
./remove_deployment.sh -g <github-user> <hubname>
```

Add `-D` the first time. A dry run prints every step and changes nothing.

Before it changes anything, the script asks you to type the hub name. It also
stops before each of its two merges and asks `y` or `n`.

### Options

| Flag | What it does |
|---|---|
| `-g`, `--github_user` | Your GitHub username. Required. |
| `-D`, `--dry-run` | Print each step without running it. |
| `-s`, `--skip-archive` | Delete the NFS homedirs without archiving them. The script makes you type the hub name a second time. |
| `-f`, `--finish` | Run only part 2 (NFS and the deployment folder). Use it after you merge the first PR yourself. |
| `-n`, `--no-pr` | Push the branches, but don't open or merge any PRs. |

### What the script does

First, the script checks that:

- the `remove-<hubname>-gha` and `remove-<hubname>-deployment` branches don't
  exist yet, locally or on your fork
- you're on `staging` with a clean working tree, and `deployments/<hubname>`
  exists
- `kubectl` and `gcloud` point at the `cal-icor-hubs` cluster and project

If any check fails, it prints what's wrong and exits before touching anything.

Part 1 removes the infrastructure and opens the first PR:

1. Deletes the hub's alert policy and uptime check.
2. Deletes the hub's CILogon client.
3. Uninstalls the `<hubname>-prod` and `<hubname>-staging` Helm releases.
4. On a new branch, `remove-<hubname>-gha`, removes the hub from
   `.github/labeler.yml`, the five issue templates and the NFS quota paths.
   Then it deletes the `hub: <hubname>` GitHub label.
5. Opens a PR, waits for its checks, shows you its labels and asks before
   merging. After the merge, it syncs `staging` and deletes the branch.

Part 2 removes the NFS homedirs and the deployment folder:

1. Looks for user servers still running in the hub's namespaces. If it finds
   any, it lists them and asks whether to continue.
2. Waits for the NFS server to drop the hub's quota path. The quota enforcer
   recreates every path in its config, so a directory deleted any earlier
   comes back empty.
3. Archives `/export/<hubname>` to `/export/<hubname>.tar.gz`, checks that the
   archive reads back, then deletes the directory.
4. On a new branch, `remove-<hubname>-deployment`, removes
   `deployments/<hubname>`.
5. Opens a PR, waits for its checks and asks before merging. After the merge,
   it syncs `staging` and deletes the branch.

The labeler change has to merge first. If both changes went into one PR, the
labeler action would recreate the hub's label on it.

If the hub shares another hub's NFS directory (`rstudio` uses `jupyter/prod`,
for example), the script skips the NFS steps and leaves the directory alone.

### If you stop partway

If you answer `n` at the first merge prompt, merge that PR yourself, sync
`staging`, then run part 2:

``` bash
./remove_deployment.sh -g <github-user> --finish <hubname>
```

### After the script finishes

1. Merge `staging` to `prod`. Until you do, a prod deploy of every hub
   reinstalls `<hubname>-prod`.
2. Remove the hub's token from the `cloudbank-pilot-hub-users` service in
   `enc-pilots.json`. The steps are in the
   [cal-icor-hubs README](https://github.com/cal-icor/cal-icor-hubs#keeping-it-in-sync-with-cloudbank-pilot-hub-users).
3. If the script couldn't delete the CILogon client, the hub probably has one
   of our older clients. See [Delete the CiLogon client](#delete-the-cilogon-client).

The archive stays in `/export` on the NFS server. Delete it once the
institution no longer needs the data.

## Removing a hub by hand

### The canonical list of steps to remove a hub deployment

1. [Delete the alerts](#delete-the-alert-policy).
2. [Delete the `prod` and `staging` Helm deployments](#delete-the-helm-deployments).
3. [Archive or delete the `prod` folder on the NFS server](#archive-or-delete-nfs-storage).
4. [Remove deployment from GitHub labeler action, GitHub labels and issue templates](#remove-deployment-label-label-action-entry-and-issue-templates).
5. [Remove deployment folder under `cal-icor-hubs/deployments/`](#remove-deployment-from-hub-repo).
6. [Review local changes and create a PR](#review-your-changes).
7. [Review and merge](#review-and-merge) the changes from steps (5) and (6).
8. [Delete the CiLogon client](#delete-the-cilogon-client)

### Delete the alert policy

Go back to the GCP console, and under Monitoring -> Alerting, click on the
deployment's Policy and then click on Delete
If you don't disable the alerts, a page will be sent off when GCP is unable to
reach the `prod` deployment of the hub you're removing.

Open up the [GCP console](https://console.cloud.google.com/) and using the
sidebar, navigate to Monitoring -> Alerting.  Search for the deployment's
Policy, click on the link, and then click on "Delete".

### Delete the Helm deployments

Ensure you're logged in to GCP on the command line, and run the following two
commands (replace `<hubname>` with the hub name):

``` bash
helm delete -n <hubname>-prod <hubname>-prod
helm delete -n <hubname>-staging <hubname>-staging
```

This effectively deletes the hub's kubernetes deployments.

### Archive or delete NFS storage

This step depends on what the institution want to do, if anything, with the
user homedirs on the NFS server.

If there is some user data, then it would be best to archive it for a certain
amount of time (probably a year at minimum).  If the hub has never been used,
or only a couple of instructors had logged in to "kick the tires", then the
NFS directories should be fine to delete completely.

To archive the deployment's NFS directories, run the following commands:

``` bash
pod_name=$(kubectl get pod -n jupyterhub-home-nfs -l app.kubernetes.io/component=nfs-server -o "jsonpath={.items[0].metadata.name}")
kubectl exec -n jupyterhub-home-nfs ${pod_name} -- sh -c "tar -zcvf /export/<hubname>.tar.gz /export/<hubname> && ls -l /export/<hubname>.tar.gz"
```

Be sure that the deployment's homedir archive has been created before deleting
anything.

Then, you can delete the directory by running:

``` bash
kubectl exec -n jupyterhub-home-nfs ${pod_name} -- sh -c "rm -rf /export/<hubname>"
```

### Remove deployment label, label action entry and issue templates

This step is required before removing anything else from the `cal-icor-hubs`
repository.  If you combine all of the changes/deletions as described in this
doc in to a single PR you will then need to both manually delete the labels in
the PR, as well as the labels themselves.  This is because GitHub helpfully
re-creates the deleted labels when the labeler action runs against a fresh PR.

First, create a new feature branch from `staging` in your local clone of
`cal-icor-hubs` before continuing:

``` bash
git checkout -b remove-<hubname>-gha
```

Then edit `.github/labeler.yml` and remove the hub's entry located towards the
end of this file.

Next, we will remove the GitHub labels and the URLs in the GitHub Issue
template folder.

``` bash
gh label delete "hub: <hubname>"
```

Edit the follow files found in the `.github/ISSUE_TEMPLATE/` folder and remove
the hub's URL from each one.  While not strictly necessary at this point in the
process, we lump it in here as it's part of the GitHub ecosystem:

``` bash
additional_storage_request.yaml
admin_request.yaml
cpu_template.yml
memory_request.yml
package_request.yml
```

#### Review your changes, create a PR and merge to `staging`

Confirm that the changes you've made are all correct with `git status` and
`git diff`, and then create a PR with these changes in the `cal-icor-hubs` repo
and merge it to `staging` before contiuing.

### Remove deployment from hub repo

:::{admonition} Sync your repo!
:class: attention
Be sure to sync your local repo to `upstream` and create a new feature branch before continuing!
:::

Now you can delete the folder under `deployments/` for this hub:

``` bash
git rm -rf deployments/<hubname>
```

### Review your changes

Run `git diff` and ensure everything looks good!  After that, add/commit and
push to the `cal-icor-hubs` repository and create a PR.

### Review and merge

Ensure that just the expected files are going to be removed from the repository
and that the hub's labels aren't added to the PR.

Once you're happy that things look good, merge to `staging`.  This can be
merged to prod at your leisure.

### Delete the CiLogon client

:::{admonition} Older CiLogon clients may not be able to be manually deleted!
:class: attention
Since our original CiLogon clients were created by the CiLogon team, we aren't
access or delete them through our CLI tool.  You'll need to reach out to them
directly at <help@cilogon.org> with the URL of the removed hub.
:::

Run the following command to delete the deployment's CiLogon client:

``` bash
./scripts/cilogon_clients.py remove <hubname>
```
