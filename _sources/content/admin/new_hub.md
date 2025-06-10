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

### TL;DR

1. [Choose a new hub name](#name-the-hub).
2. [Determine deployment needs](#determine-deployment-needs), and create a new node pool and/or filestore instance if necessary.
3. Be sure your gcloud config is set to the cal-icor-hubs gcloud project: `gcloud config set project cal-icor-hubs`
4. If using a custom single-user server image, create the [new image and repository](new_image).
5. If needed, [create a node placeholder scaler entry](#create-node-placeholder-scaler-entry) (only if a [new node pool was created](#creating-a-new-node-pool)).
6. [Create a new hub deployment configuration file](#create-the-hub-deployment-configuration) in `cal-icor-hubs/_deploy_configs/`.
    - [Determine the authentication method](#authentication) and create a new CILogon or Github OAuth application.
7. Execute the file: `./create_deployment.sh -g <github username> <institution or hubname>`. This script automatically:
    - [Create the hub's `staging` and `prod` directories in the filestore](#create-hubs-staging-and-prod-directories-in-the-filestore).
    - Commit and create the PR in `staging`.
8. Review the PR created on GitHub by reviewing the new files
9. Merge the PR. The Github Action executes the deployment to staging.
10. Test the staging hub.
11. Create a PR from `staging` to `prod` and merge
12. [Create the alerts](#create-the-alerts-for-the-new-hub) for the `prod` deployment of the new hub by executing: `./create-alerts.sh <hub-name>`

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

#### Placing this hub on shared resourses (node pool and/or filestore)

If you're going to use an existing node pool and/or filestore instance,
you can skip either or both of the following steps and pick back up at
[Create the hub deployment in the `cal-icor-hubs` repo](#create-the-hub-deployment-configuration).

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
(`<filestore-instance-name>-<YYYY-MM-DD>`) and `<capacity>`, and then execute the
following command:

``` bash
gcloud filestore instances create <filestore-instance-name>-<YYYY-MM-DD> \
  --zone "us-central1-b" --tier="BASIC_HDD" \
  --file-share=capacity=<capacity>,name=shares \
  --network=name=default,connect-mode=DIRECT_PEERING
```

Your new (but empty) NFS filestore must be seeded with a pair of
directories. We run a utility VM for NFS filestore management; follow
the steps below to connect to this utility VM, mount your new filestore,
and create & configure the required directories.

You can run the following command in gcloud terminal to log in to the
NFS utility VM:

```bash
gcloud compute ssh nfsserver --zone=us-central1-b --tunnel-through-iap
```

Now, mount the new filestore instance:

``` bash
mkdir /export/<filestore-instance-name>-filestore
mount <filestore share IP>:/shares /export/<filestore-instance-name>-filestore
```

### Create hub's `staging` and `prod` directories in the filestore

Next, you will need to create the `staging` and `prod` directories
inside the filestore instance.  This is where the users' home directories will
be located.

``` bash
install -d -o 1000 -g 1000 \
  /export/<filestore-instance-name>-filestore/<hubname>/staging \
  /export/<filestore-instance-name>-filestore/<hubname>/prod
```

### Configure filestore security settings and GCP billing labels

:::{admonition} Skip this step if you are using an existing/shared filestore!
:class: warning
This step is only necessary if you are creating a new filestore instance. An
existing filestore instance will already have the `ROOT_SQUASH` settings
applied, and the labels will already be set.
:::

If you have created a new filestore instance, you will now need to apply
the `ROOT_SQUASH` settings. Please ensure that you've already
[created the hub's root directory and both `staging` and `prod`](#create-hubs-staging-and-prod-directories-in-the-filestore)
directories, otherwise you will lose write access to the share. We also attach
labels to a new filestore instance for tracking individual and full hub costs.

``` bash
gcloud filestore instances update <filestore-instance-name> --zone=us-central1-b  \
       --update-labels=hub=<hubname>,filestore-deployment=<hubname> \
       --flags-file=<hubname>/config/filestore/squash-flags.json
```

### Authentication

#### CiLogon Auth
Go to the [CILogon Registration](https://cilogon.org/oauth2/register) page and create a new
application.  

The page looks like this:
!["Image of CILogon Client Registration Page"](cilogon.png)

Here is an example, using CSU Long Beach, of how to complete the form for the **staging** hub:
- Client Application: California State University, Long Beach - Staging
- Contact Email: cal-icor-staff@lists.berkeley.edu
- Home Url: https://\<hubname\>-staging.jupyter.cal-icor.org
- Callback URLs: https://\<hubname\>-staging.jupyter.cal-icor.org/hub/oauth_callback
- Client Type: Confidential
- Scopes: email, openid, org.cilogon.userinfo
- Refresh Tokens: No

After clicking the "Register Client" button, you are re-directed to a page that contains your
Client ID and Secret; copy both of these to an appropriate place right away. You are going to insert them into the 
file `_deploy_configs/<hub_name>.yaml` you create in this [step](#create-the-hub-deployment-configuration).

You will need to create two applications for each hub, one for the staging hub and one for the
production hub. The example above is for staging. The changes for **production** are:
- Client Application: California State University, Long Beach
- Home Url: https://\<hubname\>.jupyter.cal-icor.org
- Callback URLs: https://\<hubname\>.jupyter.cal-icor.org/hub/oauth_callback

#### GitHub Auth
Sometimes we can not set up CILogon for a particular institution. The other option is to use Github OAuth.
- Create an Github OAuth App at [github.com/cal-icor](https://github.com/organizations/cal-icor/settings/applications) 
- Click the button: `New OAuth App`
- Complete the fields for the production OAuth
  - Application Name: \<hubname\>-auth
  - Homepage URL: https://\<hubname\>.jupyter.cal-icor.org
  - Application Description:  This manages authentication for \<hubname\>
  - Authorization callback URL:  https://\<hubname\>.jupyter.cal-icor.org/hub/oauth_callback
  - Enable Device Flow : Do not check this
  - Click the green button: `Register Application`
- You will be given a Client ID and Secret. Copy them into the 
file `_deploy_configs/<hub_name>.yaml` you create in this [step](#create-the-hub-deployment-configuration).

- Create a OAuth App for the staging environment as well. The only changes from above are:
  - Application Name: \<hubname\>-staging-auth
  - Homepage URL: https://\<hubname\>-staging.jupyter.cal-icor.org
  - Authorization callback URL:  https://\<hubname\>-staging.jupyter.cal-icor.org/hub/oauth_callback

### Create the hub deployment configuration

First, you will need to create a new hub deployment configuration file in
`cal-icor-hubs/_deploy_configs/`.  In that directory, there will be a template
named `institution-example.yaml`, and you can just `cp` that and name the new
file `<institution or hubnname>.yaml`.

Be sure to include the [authentication bits](#authentication) that you created
via either CILogon or Github OAuth in the configuration file.

From the root `cal-icor-hubs/` directory, you will run
`create_deployment.sh -g <github username> <institution or hubname>`. This sets up the hub's
configuration directory in `cal-icor-hubs/deployments/`.

:::{admonition} Important note about `create_deployment.sh`!
:class: warning
If this hub is being deployed on the default shared resources (base-pool and
shared filestore) you just need to pass the deployment name to the script.

If you are deploying to a new node pool or different Filestore instance, you
can change the `hub_filestore_instance` and `hub_filestore_ip` in the
`<hubname>.yaml` file you created above. The script will then use the correct
filestore instance and IP address when creating the deployment.
:::

Here's an example for a hub being deployed on the default shared resouces:

``` bash
$ cat _deploy_configs/newschool.yaml
# This is an example of a configuration file for a JupyterHub deployment.
# This file should be renamed to <name of deployment>.yaml and kept in the
# _deploy_configs directory.
hub_name: newschool
hub_filestore_instance: shared-filestore
hub_filestore_ip: 10.183.114.2
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
allow_all: ""
```

``` bash
./create_deployment.sh -g shaneknapp newschool
Creating directories for newschool on filestore.
Switched to a new branch 'add-newschool-deployment'
Created new branch: add-newschool-deployment
Populating deployment config for newschool.
Generating newschool cookiecutter template.
Generating and encrypting secrets for newschool.
Secret file generation and encryption beginning.
Encrypted file saved as: deployments/newschool/secrets/prod.yaml
Deleted file: deployments/newschool/secrets/prod.plain.yaml
Secret file generation and encryption beginning.
Encrypted file saved as: deployments/newschool/secrets/staging.yaml
Deleted file: deployments/newschool/secrets/staging.plain.yaml
Creating repo and github labels for newschool.
Added newschool to the labeler.yml file.
✓ Label "hub: newschool" created in cal-icor/cal-icor-hubs
Created GitHub label for newschool.
Staging new deployment files for newschool.
Adding deployments/newschool/ to staging.
Adding .github/labeler.yml to staging.
Committing changes for newschool with message Add newschool deployment..
yamllint.................................................................Passed
ruff.................................................(no files to check)Skipped
ruff-format..........................................(no files to check)Skipped
pyupgrade............................................(no files to check)Skipped
isort................................................(no files to check)Skipped
black................................................(no files to check)Skipped
flake8...............................................(no files to check)Skipped
Ensure secrets are encrypted with sops...................................Passed
codespell................................................................Passed
fix end of files.........................................................Passed
fix requirements.txt.................................(no files to check)Skipped
check for case conflicts.................................................Passed
check that executables have shebangs.................(no files to check)Skipped
[add-newschool-deployment 51f7d02] Add newschool deployment.
 9 files changed, 274 insertions(+)
 create mode 100644 deployments/newschool/config/common.yaml
 create mode 100644 deployments/newschool/config/filestore/squash-flags.json
 create mode 100644 deployments/newschool/config/prod.yaml
 create mode 100644 deployments/newschool/config/staging.yaml
 create mode 100644 deployments/newschool/hubploy.yaml
 create mode 100644 deployments/newschool/secrets/gke-key.json
 create mode 100644 deployments/newschool/secrets/prod.yaml
 create mode 100644 deployments/newschool/secrets/staging.yaml
Pushing add-newschool-deployment to origin
Enumerating objects: 21, done.
Counting objects: 100% (21/21), done.
Delta compression using up to 24 threads
Compressing objects: 100% (16/16), done.
Writing objects: 100% (17/17), 8.01 KiB | 8.01 MiB/s, done.
Total 17 (delta 5), reused 0 (delta 0), pack-reused 0
remote: Resolving deltas: 100% (5/5), completed with 4 local objects.
remote:
remote: Create a pull request for 'add-newschool-deployment' on GitHub by visiting:
remote:      https://github.com/shaneknapp/cal-icor-hubs/pull/new/add-newschool-deployment
remote:
To github.com:shaneknapp/cal-icor-hubs.git
 * [new branch]      add-newschool-deployment -> add-newschool-deployment
Creating pull request for newschool.
Creating a pull request for newschool on branch add-newschool-deployment
['gh', 'pr', 'new', '-t Add `newschool` deployment.', '-Rcal-icor/cal-icor-hubs', '-Hshaneknapp:add-newschool-deployment', '-Bstaging', '-lhub: newschool', '-b Add `newschool` deployment, brought to you by `create_deployment.py`.']

Creating pull request for shaneknapp:add-newschool-deployment into staging in cal-icor/cal-icor-hubs

https://github.com/cal-icor/cal-icor-hubs/pull/135
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
- `hub_filestore_instance`: Defaults to`shared-filestore`, change if you use a different
  filestore instance.
- `hub_filestore_ip`: Defaults to `10.183.114.2`, change if you use a different
  filestore instance.
- `hub_filestore_share`: Default is `shares`, do not change.
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

### CI/CD and single-user server image

CI/CD is managed through Github Actions, and the relevant workflows are located
in `.github/workflows/`.  Deploying all hubs are managed via Pull Request
Labels, which are applied automatically on PR creation.

This label will be created automatically when you run
`create_deployment.sh`, and will look like `hub: <hubname>`.

#### Hubs inheriting an existing single-user server image

If this hub will inherit an existing image, the `create_deployment.sh`
script will have created a `hubploy.yaml` file in the `deployments/<hubname>/`
directory.  This file will have the image tag from an existing deployment which
will contain the latest image hash.

The image specification is found in the cookiecutter template, located here:

`cal-icor-hubs/deployments/template/{{cookiecutter.hub_name}}/hubploy.yaml`

#### Hubs using a custom single-user server image

If this hub will be using its own image, then follow the
[instructions here](new_image) to create the new image and repository.  In this
case, the image tag will be `PLACEHOLDER` and will be updated AFTER your PR to
`cal-icor-hubs` is merged.

*NOTE:* The changes to the `cal-icor-hubs` repo are required to be merged
BEFORE the new image configuration is pushed to `main` in the image repo.  This
is due to the image building/pushing workflow requiring this deployment's
`hubploy.yaml` to be present in the `deployments/<hubname>/` subdirectory, as
it updates the image tag.

#### Review the deployment's `hubploy.yaml`

Next, review `hubploy.yaml` inside your project directory to confirm that
looks cromulent.  An example from the `jupyter` hub:

``` yaml
images:
  images:
    - name: us-central1-docker.pkg.dev/cal-icor-hubs/user-images/base-user-image:<image tag OR "PLACEHOLDER">
```

### Create node placeholder scaler entry

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

### Commit and deploy to `prod`

Make a PR from the `staging` branch to the `prod` branch. When this
PR is merged, it'll deploy the production hub. It might take a few
minutes for HTTPS to work, but after that you can log into it at
<https://>\<hub_name\>.cal-icor.org. Test it out and make
sure things work as you think they should.

### Create the alerts for the new hub

Once you've deployed the new hub to `prod`, you must create the uptime alerts
for the hub. From the `scripts` directory, run the following commands to create
and enable the alerts for the new hub:

``` bash
./create_alerts.py --create --namespaces <hubname>-prod
./create_alerts.py --enable --namespaces <hubname>-prod
```
