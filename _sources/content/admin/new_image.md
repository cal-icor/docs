# Create a New Single User Image

You might need to create a new user image when deploying a new hub, or changing
from a shared single user server image. We use
[repo2docker](https://github.com/jupyterhub/repo2docker) to generate our images.

There are two approaches to creating a repo2docker image:

1. Use a repo2docker-style image [template](https://github.com/cal-icor/base-user-image) (environment.yaml, etc)
2. Use a [Dockerfile](https://github.com/berkeley-dsep-infra/datahub-user-image/tree/main) (useful for larger/more complex images)

Generally, we prefer to use the first (`repo2docker`) approach.

If we need to install software as `root`, you can add a
[Dockerfile.appendix](https://repo2docker.readthedocs.io/en/latest/usage.html#cmdoption-jupyter-repo2docker-appendix)
to the repo.

There are two approaches to pre-populate the image's assets:

1. Fork our [base-user-image](https://github.com/cal-icor/base-user-image).
Click "Use this template" > "Create a new repository". Be sure to follow
convention and name the repo `<hubname>-user-image`, and the owner needs to be
`cal-icor`. When that is done, create your own fork of the new repo.

2. Use an existing image as a template. Browse through the
[berkeley-dsep-infra image repos](https://github.com/orgs/berkeley-dsep-infra/repositories?language=&q=image&sort=&type=all)
to find a hub that is similar to the one you are trying to create. This will
give you a good starting point.

## Image Repository Settings

There are now a few steps to set up the CI/CD for the new image repo.  In the
`cal-icor` image repo, click on `Settings`, and under `General`,
scroll down to `Pull Requests` and check the box labeled `Automatically delete
head branches`.

Scroll back up to the top of the settings, and in the left menu bar, click on
`Secrets and variables`, and then `Actions`.

From there, click on the `Variables` tab and then `New repository variable`. We
will be adding two new variables:

1. `HUB`:  the name of the hub (eg: jupyter)

1. `IMAGE`:  the Google Artifact Registry path and image name.  The path will
always be `us-central1-docker.pkg.dev/cal-icor-hubs/user-images/<image-name>` and the
image name will always be the same as the repo:  `<hubname>-user-image`.

## Disable Your Fork's Repository Settings

Now you will want to disable Github Actions for your fork of the image repo.
If you don't, whenever you push PRs to the root repo the workflows *in your
fork* will attempt to run, but don't have the proper permissions to
successfully complete.  This will then send you a nag email about a workflow
failure.

To disable this for your fork, click on `Settings`, `Actions` and `General`.
Check the `Disable actions` box and click save.

### Enable Artifact Registry Pushing

The image repository needs to be added to the list of allowed repositories in
the `cal-icor` secrets. Go to the `cal-icor` [Secrets and
Variables](https://github.com/organizations/cal-icor/settings/secrets/actions).
Give your repository permissions to push to the Artifact Registry,
as well as to push a branch to the [cal-icor-hubs repo](https://github.com/cal-icor/cal-icor-hubs).

Edit both `IMAGE_BUILDER_CREATE_PR` and `GAR_SECRET_KEY`, and click on the gear icon,
search for your repo name, check the box and save.

### Configure `common.yaml`

You need to let `helm` (via running `hubploy`) know the specifics of the image
by updating your deployment's chart in `common.yaml`. Change the `name` of the
image in `deployments/<hubname>/config/common.yaml` to point to your new image
name, and after the name add `PLACEHOLDER` in place of the image sha.  This
will be automatically updated after your new image is built and pushed to the
Artifact Registry.

Example:

```yaml
jupyterhub:
  # a bunch of hub config here...
  # ...
  singleuser:
    image:
      name: us-central1-docker.pkg.dev/cal-icor-hubs/user-images/<hubname>-user-image
      tag: PLACEHOLDER
```

Create a PR and merge to staging.  You can cancel the
[`Deploy staging and prod hubs` job in Actions](https://github.com/cal-icor/cal-icor-hubs/actions/workflows/deploy-hubs.yaml),
or just let it fail.

## Subscribe to GitHub Repo in Slack

Go to the #cal-icor-bots channel, and run the following command:

``` bash
/github subscribe cal-icor/<your repo name>
```

## Modify the Image

This step is straightforward: create a feature branch, and edit, delete, or add
any files to configure the image as needed.

We also strongly recommend copying `README-template.md` over the default
`README.md`, and modifying it to replace all occurrences of `<HUBNAME>` with
the name of your image.

## Submit Pull Requests

Familiarize yourself with [pull
requests](https://help.github.com/en/github/collaborating-with-issues-and-pull-requests/about-pull-requests)
and [repo2docker](https://github.com/jupyter/repo2docker), and create a fork of
the [cal-icor staging branch](https://github.com/cal-icor/cal-icor-hubs).

1. Set up your git/dev environment by following the [image templat's
contributing
    guide](https://github.com/cal-icor/base-user-image/blob/main/CONTRIBUTING.md).

2. [Test the image locally](repo2docker_local.md) using `repo2docker`.
3. Submit a PR to `staging`.
4. Commit and push your changes to your fork of the image repo, and
    create a new pull request at
    `https://github.com/cal-icor/<hubname-user-image>`.
5. After the build passes, merge your PR in to `main` and the image will
    be built again and pushed to the Artifact Registry.  If that succeeds,
    then a commit will be crafted that will update the `PLACEHOLDER` field in
    `common.yaml` with the image's SHA and pushed to the `cal-icor-hubs` repo.
    You can check on the progress of this workflow in your root image repo's
    `Actions` tab.
6. After the previous step is completed successfully, go to the
    `cal-icor-hubs` repo and click on the [New pull
    request](https://github.com/cal-icor/cal-icor-hubs/compare)
    button.  Next, click on the `compare: staging` drop down, and you should see
    a branch named something like `update-<hubname>-image-tag-<SHA>`.  Select
    that, and create a new pull request.

7. Once the checks has passed, merge to `staging` and your new image will be
    deployed!  You can watch the progress in the [deploy-hubs workflow](https://github.com/cal-icor/cal-icor-hubs/actions/workflows/deploy-hubs.yaml).
