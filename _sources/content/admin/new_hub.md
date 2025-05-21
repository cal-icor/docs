# Creating a New Hub

## Why create a new hub?

When an institution would like to join the CAL-ICOR JupyterHub deployment, we
create a new hub for them. This is typically done when a new course or
department is created, or when a new instructor would like to use the
JupyterHub deployment for their course. The new hub will be created in the same
GKE cluster as the existing hubs, but will have its own set of resources
and configurations. This allows us to manage the resources and
configurations for each hub independently, while still benefiting from
the shared infrastructure of the GKE cluster.

## Prerequisites

Working installs of the following utilities:

- [chartpress](https://pypi.org/project/chartpress/)
- [cookiecutter](https://pypi.org/project/cookiecutter/)
- [gcloud](https://cloud.google.com/sdk/docs/install)
- [hubploy](https://github.com/berkeley-dsep-infra/hubploy)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [sops](https://github.com/mozilla/sops/releases)

The easiest way to install `chartpress`, `cookiecutter` and `hubploy` is to
run `pip install -r dev-requirements.txt` from the root of the `cal-icor-hubs`
repo.

Proper access to the following systems:

- Google Cloud IAM: *owner*
- Write access to the [CaliICOR hubs repo](https://github.com/cal-icor/cal-icor-hubs)
- Owner or admin access to the [cal-icor Github organization](https://github.com/cal-icor/)

## Configuring a New Hub

### Name the hub

Choose the hub name, which is typically the name of the institution joining our
community.  It may also include the name of a course or department.  This is a
permanant name, so be sure to check with the instructor and/or department
before proceeding.  The name should be all lowercase, and can include
letters, numbers, and hyphens.  The name should not include any spaces or
special characters.  The name should be unique within the CAL-ICOR
JupyterHub deployment.  The name should be short and easy to remember.

### Determine deployment needs

Before creating a new hub, have a discussion with the instition rep/instructor
about the system requirements, frequency of assignments and how much storage
will be required for the course. Typically, there are three general
"types" of hub: Heavy usage, general and small courses.

Small courses will usually have one or two assignments per semester, and
may only have 20 or fewer users.

General courses have up to ~500 users, but don't have large amount of
data or require upgraded compute resources.

Heavy usage courses can potentially have thousands of users, require
upgraded node specs and/or have Terabytes of data each semester.

Both general and heavy usage courses typically have weekly assignments.

Small courses (and some general usage courses) can use either or both of
a shared node pool and filestore to save money (Basic HDD filestore
instances start at 1T).

This is also a good time to determine if there are any specific software
packages/libraries that need to be installed, as well as what
language(s) the course will be using. This will determine which image to
use, and if we will need to add additional packages to the image build.

#### Placing this hub on shared resourses (node pool and filestore)

If you're going to use an existing node pool and/or filestore instance,
you can skip either or both of the following steps and pick back up at
[Create the hub deployment in the `cal-icor-hubs` repo](#create-the-hub-deployment-in-the-repo).

When creating a new hub, we also make sure to label the filestore and
GKE/node pool resources with both `hub` and
`<nodepool|filestore>-deployment`. 99.999% of the time, the values for
all three of these labels will be `<hubname>`.

### Creating a new node pool

Create the node pool:

``` bash
gcloud container node-pools create "user-<hubname>-<YYYY-MM-DD>" \
  --labels=hub=<hubname>,nodepool-deployment=<hubname> \
  --node-labels hub.jupyter.org/pool-name=<hubname>-pool \
  --machine-type "n2-highmem-8" \
  --enable-autoscaling --min-nodes "0" --max-nodes "20" \
  --project "cal-icor-hubs" --cluster "spring-2025" \
  --region "us-central1" --node-locations "us-central1-b" \
  --node-taints hub.jupyter.org_dedicated=user:NoSchedule --tags hub-cluster \
  --image-type "COS_CONTAINERD" --disk-type "pd-balanced" --disk-size "200"  \
  --metadata disable-legacy-endpoints=true \
  --scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" \
  --no-enable-autoupgrade --enable-autorepair \
  --max-surge-upgrade 1 --max-unavailable-upgrade 0 --max-pods-per-node "110"
```

### Creating a new filestore instance

Before you create a new filestore instance, be sure you know the
capacity required. The smallest amount you can allocate is 1T, but
larger hubs may require more. Confer with the admins and people
instructing the course and determine how much they think they will need.

We can easily scale capacity up, but not down.

From the command line, first fill in the instance name
(`<hubname>-<YYYY-MM-DD>`) and `<capacity>`, and then execute the
following command:

``` bash
gcloud filestore instances create <hubname>-<YYYY-MM-DD> \
  --zone "us-central1-b" --tier="BASIC_HDD" \
  --file-share=capacity=1TiB,name=shares \
  --network=name=default,connect-mode=DIRECT_PEERING
```

Or, from the web console, click on the horizontal bar icon at the top
left corner.

1. Access "Filestore" > "Instances" and click on "Create Instance".
2. Name the instance `<hubname>-<YYYY-MM-DD>`
3. Instance Type is `Basic`, Storage Type is `HDD`.
4. Allocate capacity.
5. Set the region to `us-central1` and Zone to `us-central1-b`.
6. Set the VPC network to `default`.
7. Set the File share name to `shares`.
8. Click "Create" and wait for it to be deployed.
9. Once it's deployed, select the instance and copy the "NFS mount
   point".

Your new (but empty) NFS filestore must be seeded with a pair of
directories. We run a utility VM for NFS filestore management; follow
the steps below to connect to this utility VM, mount your new filestore,
and create & configure the required directories.

You can run the following command in gcloud terminal to log in to the
NFS utility VM:

```bash
gcloud compute ssh nfsserver --zone=us-central1-b --tunnel-through-iap
```

Alternatively, launch console.cloud.google.com > Select *cal-icor-hubs* as the
project name.

1. Click on the three horizontal bar icon at the top left corner.
2. Access "Compute Engine" > "VM instances" > and search for
   "nfs-server-01".
3. Select "Open in browser window" option to access NFS server via
   GUI.

Back in the NFS utility VM shell, mount the new share:

``` bash
mkdir /export/<hubname>-filestore
mount <filestore share IP>:/shares /export/<hubname>-filestore
```

Create `staging` and `prod` directories owned by `1000:1000` under
`/export/<hubname>-filestore/<hubname>`. The path *might* differ if your
hub has special home directory storage needs. Consult admins if that's
the case. Here is the command to create the directory with appropriate
permissions:

``` bash
install -d -o 1000 -g 1000 \
  /export/<hubname>-filestore/<hubname>/staging \
  /export/<hubname>-filestore/<hubname>/prod
```

Check whether the directories have permissions similar to the below
directories:

``` bash
drwxr-xr-x 4 ubuntu ubuntu     45 Apr  3 20:33 jupyter-filestore
drwxr-xr-x 4 ubuntu ubuntu     33 Apr  4  2025 dvc-filestore
drwxr-xr-x 4 ubuntu ubuntu  16384 May 16 18:45 lacc-filestore
```

### Create the hub deployment in the repo

First, you will need to create a new hub deployment configuration file in
`cal-icor-hubs/_deploy_configs/`.  In that directory, there will be a template
named `institution-example.yaml`, and you can just `cp` that and name the new
file `<institution or hubnname>.yaml`.

After you create the new config, edit it and fill in the blanks (for the
authentication bits, please refer to the
[authentication section below](#authentication)).

From the root `cal-icor-hubs/` directory, you will run
`create_deployment.sh <institution or hubname>`. This sets up the hub's
configuration directory in `cal-icor-hubs/deployments/`.

:::{admonition} Important note about `create_deployment.sh`!
:class: warning
If this hub is being deployed on the default shared resources (base-pool and
shared Filestore) you just need to pass the deployment name to the script.

If you are deploying to a new node pool or different Filestore instance, you
need to add the `-m` flag, which will allow you to confirm each step of the
cookiecutter process and change whatever fields that you need.
:::

Here's an example for a hub being deployed on the default shared resouces:

``` bash
$ cat _deploy_configs/newschool.yaml
# This is an example of a configuration file for a JupyterHub deployment.
# This file should be renamed to <name of deployment>.yaml and kept in the
# _deploy_configs directory.
hub_name: newschool
institution: newschool
institution_url: https://example.edu
institution_logo_url: https://example.edu/logo.png
landing_page_branch: main
prod:
  client: asdf
  secret: asdffdsa
staging:
  client: 1324
  secret: 12344312
admin_emails:
  - sknapp@berkeley.edu
  - sean.smorris@berkeley.edu
authenticator_class: cilogon
authenticator_class_instance: "CILogonOAuthenticator"
idp_url:   http://login.microsoftonline.com/common/oauth2/v2.0/authorize
idp_allowed_domains:
  - example.edu
  - whee.edu
```

``` bash
./create_deployment.sh newschool
newschool cookiecutter template configured successfully.
Encrypted file saved as: deployments/newschool/secrets/prod.yaml
Deleted file: deployments/newschool/secrets/prod.plain.yaml
Secret file generation and encryption completed.
Encrypted file saved as: deployments/newschool/secrets/staging.yaml
Deleted file: deployments/newschool/secrets/staging.plain.yaml
Secret file generation and encryption completed.
```

If you pass the `-m` flag to the script, the cookiecutter template will be read
in from your config, and then prompt you to confirm or make changes to the
following information:

- `<hub_name>`: Enter the chosen name of the hub.
- `<institution>`: Enter the name of the institution.
- `<instution_url>`: Enter the URL of the institution.
- `<institution_logo_url>`: Enter the URL of the institution's logo.
- `<landing_page_branch>`: Default is `main`, do not change unless the hub requires a different authentication method than CILogon or Github Auth.
- `<project_name>`: Default is `cal-icor-hubs`, do not change.
- `<cluster_name>`: Default is `spring-2025`, do not change.
- `<pool_name>`: Name of the node pool (shared or individual) to deploy on.
- `hub_filestore_share`: Default is `shares`, do not change.
- `hub_filestore_ip`: Enter the IP address of the filestore instance. This is available from the web console.
- `hub_filestore_capacity`: Enter the allocated storage capacity. This is available from the web console.
- `authenticator_class`: Default is `cilogon`, do not change unless the hub requires a different authentication method.
- `authenticator_class_instance`: Default is `CILogonOAuthenticator`, do not change unless the hub requires a different authentication method.
- `idp_url`: The endpoint for Google or Microsoft authentication. This is available from the partnered institution.
- `idp`: Currently unused?
- `admin_emails`: Enter the email addresses of the admins for the hub. This is a comma-separated list.
- `client_id_staging`: Enter the client ID for the staging hub. This is available from Cilogon.
- `client_secret_staging`: Enter the client secret for the staging hub. This is available from Cilogon.
- `client_id_prod`: Enter the client ID for the production hub. This is available from Cilogon.
- `client_secret_prod`: Enter the client secret for the production hub. This is available from Cilogon.
- `idp_allowed_domains`: Enter the allowed domains for the hub. This is a comma-separated list.

This will generate a directory with the name of the hub you provided
with a skeleton configuration and all the necessary secrets.

### Configure filestore security settings and GCP billing labels

Skip this step if you are using an existing/shared filestore.

If you have created a new filestore instance, you will now need to apply
the `ROOT_SQUASH` settings. Please ensure that you've already created
the hub's root directory and both `staging` and `prod` directories,
otherwise you will lose write access to the share. We also attach labels
to a new filestore instance for tracking individual and full hub costs.

``` bash
gcloud filestore instances update <filestore-instance-name> --zone=us-central1-b  \
       --update-labels=hub=<hubname>,filestore-deployment=<hubname> \
       --flags-file=<hubname>/config/filestore/squash-flags.json
```

### Authentication

Go to the [CILogon](https://cilogon.org/) website and create a new
application.  This will give you a client ID and secret that you will
need to add to the `hub_name.yaml` file you created earlier.  The
application name should be the name of the hub, and the redirect URL
should be `https://<hubname>-staging.cal-icor.org/hub/oauth_callback`
and `https://<hubname>.cal-icor.org/hub/oauth_callback`.  The
application type should be `Web application`.

You will need to create two applications, one for the staging hub and one for the
production hub.

TODO: Add instructions for creating a Github OAuth app.

### CI/CD and single-user server image

CI/CD is managed through Github Actions, and the relevant workflows are located
in `.github/workflows/`.  Deploying all hubs are managed via Pull Request
Labels, which are applied automatically on PR creation.

To ensure the new hub is deployed, all that needs to be done is add a new entry
(alphabetically) in `.github/labeler.yml` under the `# add hub-specific labels
for deployment changes` stanza:

``` yaml
"hub: <hubname>":
  - "deployments/<hubname>/**"
```

#### Hubs using a custom single-user server image

TODO

#### Review the deployment's `hubploy.yaml`

Next, review `hubploy.yaml` inside your project directory to confirm that
looks cromulent.  An example from the `jupyter` hub:

``` yaml
images:
  images:
    - name: us-central1-docker.pkg.dev/cal-icor-hubs/user-images/base-user-image:<image tag OR "PLACEHOLDER">
```

### Create placeholder node pool

If you are deploying to a shared node pool, there is no need to perform
this step.

Node pools have a configured minimum size, but our cluster has the
ability to set aside additional placeholder nodes. These are nodes that
get spun up in anticipation of the pool needing to suddenly grow in
size, for example when large classes begin.

Otherwise, you'll need to add the placeholder settings in
`node-placeholder/values.yaml`.

The node placeholder pod should have enough RAM allocated to it that it
needs to be kicked out to get even a single user pod on the node - but
not so big that it can't run on a node where other system pods are
running! To do this, we'll find out how much memory is allocatable to
pods on that node, then subtract the sum of all non-user pod memory
requests and an additional 256Mi of "wiggle room". This final number
will be used to allocate RAM for the node placeholder.

1. Launch a server on https://*hubname*.cal-icor.org
2. Get the node name (it will look something like
   `gke-spring-2025-user-base-fc70ea5b-67zs`):
   `kubectl get nodes | grep *hubname* | awk '{print $1}'`
3. Get the total amount of memory allocatable to pods on this node and
   convert to bytes:

   ```bash
   kubectl get node <nodename> -o jsonpath='{.status.allocatable.memory}'
   ```

4. Get the total memory used by non-user pods/containers on this node.
    We explicitly ignore `notebook` and `pause`. Convert to bytes and
    get the sum:

   ```bash
    kubectl get -A pod -l 'component!=user-placeholder' \
      --field-selector spec.nodeName=<nodename> \
      -o jsonpath='{range .items[*].spec.containers[*]}{.name}{"\t"}{.resources.requests.memory}{"\n"}{end}' \
      | egrep -v 'pause|notebook'
    ```

5. Subtract the second number from the first, and then subtract another
   277872640 bytes (256Mi) for "wiggle room".
6. Add an entry for the new placeholder node config in `values.yaml`:

```yaml
new-institution:
  nodeSelector:
    hub.jupyter.org/pool-name: new-institution-pool
  resources:
    requests:
      # Some value slightly lower than allocatable RAM on the node pool
      memory: 60929654784
  replicas: 1
```

For reference, here's example output from collecting and calculating
the values for `data102`:

``` bash
(gcpdev) ➜  ~ kubectl get nodes | grep data102 | awk '{print$1}'
gke-spring-2024-user-data102-2023-01-05-e02d4850-t478
(gcpdev) ➜  ~ kubectl get node gke-spring-2024-user-data102-2023-01-05-e02d4850-t478 -o jsonpath='{.status.allocatable.memory}' # convert to bytes
60055600Ki%
(gcpdev) ➜  ~ kubectl get -A pod -l 'component!=user-placeholder' \
--field-selector spec.nodeName=gke-spring-2024-user-data102-2023-01-05-e02d4850-t478 \
-o jsonpath='{range .items[*].spec.containers[*]}{.name}{"\t"}{.resources.requests.memory}{"\n"}{end}' \
| egrep -v 'pause|notebook' # convert all values to bytes, sum them
calico-node
fluentbit       100Mi
fluentbit-gke   100Mi
gke-metrics-agent       60Mi
ip-masq-agent   16Mi
kube-proxy
prometheus-node-exporter
(gcpdev) ➜  ~ # subtract the sum of the second command's values from the first value, then subtract another 277872640 bytes for wiggle room
(gcpdev) ➜  ~ # in this case:  (60055600Ki - (100Mi + 100Mi + 60Mi + 16Mi)) - 256Mi
(gcpdev) ➜  ~ # (61496934400 - (104857600 + 104857600 + 16777216 + 62914560)) - 277872640 == 60929654784
```

Besides setting defaults, we can dynamically change the placeholder
counts by either adding new, or editing existing, [calendar
events](calendar_scaler.md).
This is useful for large courses which can have placeholder nodes set
aside for predicatable periods of heavy ramp up.

### Commit and deploy to `staging`

Commit the hub directory, and make a PR to the the `staging` branch in
the GitHub repo.

#### Hubs using a custom single-user server image

If this hub is using a custom image, and you're using `PLACEHOLDER` for the
image tag in `hubploy.yaml`, be sure to remove the hub-specific Github
label that is automatically attached to this pull request.  It will look
something like `hub: <hubname>`.  If you don't do this the deployment will
fail as the image sha of `PLACEHOLDER` doesn't exist.

After this PR is merged, perform the `git push` in your image repo.  This will
trigger the workflow that builds the image, pushes it to the Artifact Registry,
and finally creates a commit that updates the image hash in `hubploy.yaml` and
pushes to the `cal-icor-hubs` repo.  Once this is merged in to `staging`, the
deployment pipeline will run and your hub will finally be deployed.

#### Hubs inheriting an existing single-user server image

Your hub's deployment will proceed automatically through the CI/CD pipeline.

It might take a few minutes for HTTPS to work, but after that you
can log into it at <https://>\<hub_name\>-staging.cal-icor.org.
Test it out and make sure things work as you think they should.

### Commit and deploy to `prod`

Make a PR from the `staging` branch to the `prod` branch. When this
PR is merged, it'll deploy the production hub. It might take a few
minutes for HTTPS to work, but after that you can log into it at
<https://>\<hub_name\>.cal-icor.org. Test it out and make
sure things work as you think they should.

### Create the alerts for the new hub

From the `scripts` directory, run the following command to create the
alerts for the new hub:

``` bash
./create_alerts.py --create --namespaces <hubname>
```

After that's done, you need to enable the alerts in GCP:

``` bash
./create_alerts.py --enable --namespaces <hubname>
```
