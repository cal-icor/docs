# Customize the Hub Docker Image

We use a customized JupyterHub docker image so we can install extra packages
such as authenticators. The image is located in `images/hub`. It *must* inherit
from the JupyterHub image used in [Zero to JupyterHub](https://z2jh.jupyter.og).

## Updating the Hub Image Version

You should get the latest version of the z2jh image from the
[Helm chart repository](https://hub.jupyter.org/helm-chart/).
The image is built and pushed to the Google Artifact Registry
`us-central1-docker.pkg.dev/cal-icor-hubs/core/`. The image is tagged with the
`<version>-<git commit>` of the JupyterHub Helm chart. The image is built
with [chartpress](https://github.com/jupyterhub/chartpress), which is a wrapper
around `docker build` and `docker push`. The image is built with the `--push`
flag, which pushes the image to the Google Artifact Registry.

After the image is built, the `hub/values.yaml` file is modified to
include the new image name and tag. This is done by `chartpress` as well.

### Steps to Update the Hub Image

#### Getting Authenticated

Some of the following commands may be required to configure your
environment to run the above chartpress workflow successfully:

``` bash
gcloud auth login
gcloud auth configure-docker us-central1-docker.pkg.dev
gcloud auth application-default login
gcloud auth configure-docker
```

#### Updating, Building, and Pushing the Hub Image

1. Run `gcloud auth configure-docker us-central1-docker.pkg.dev` *once
   per machine* to setup docker for authentication with the
   [gcloud credential helper](https://cloud.google.com/artifact-registry/docs/docker/authentication).
2. Create a new feature branch in the `cal-icor-hubs` repository.
3. Update the Helm chart version in both `images/hub/Dockerfile` and
   `hub/requirements.yaml` to the latest version. The version should
   be the same as the version of the JupyterHub Helm chart you are
   using. Create a git commit with these changes.
4. On a PC run `chartpress --push`. On a Mac run, `chartpress --push --builder docker-buildx --platform linux/amd64`.
   This will build and push the hub image, and modify `hub/values.yaml` appropriately.
5. Make a commit with the `hub/values.yaml` file, so the new hub image
   name and tag are committed.
6. Proceed to deployment as normal.
