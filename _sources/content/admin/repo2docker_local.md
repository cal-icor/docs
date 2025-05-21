# Test User Images Locally

You should use `repo2docker` to build and test the image on your own device before you push and create a PR. It is often faster to do this first before using CI/CD since you can take advantage of local caching and rapid iteration. There's no need to waste Github Action minutes to test build images when you can do this on your own device.

## Common Usage

One can simply run `repo2docker /path/to/image/assets`. For example if one has changed into the directory containing the `repo2docker` files (such as `environment.yml` and/or `Dockerfile`), the command would be:

```shell
repo2docker .
```

This works on Linux and Windows Subsystem for Linux (WSL). It will build the image, then launch jupyter server and display a localhost URL. Copy the URL and paste it into a local web browser.

If you just want to build the image without also running the server,
add the `--no-run` argument:

```shell
repo2docker --no-run .
```

## On Apple Silicon

Apple's ARM-based CPUs (the "M" chips) are different from those run on the virtual machines in our clusters. macOS is capable of emulating x86_64/amd64, but it is necessary to optimize docker for this emulation, and to explicitly tell your local docker runtime that the images should be built on the `linux/amd64` platform.

In Docker's settings:

- Under **General** > **Virtual Machine Options**, either enable both **Apple Virtualization framework** and **Use Rosetta for x86_64/amd64 emulation on Apple Silicon**, or enable **Docker VMM**.
- Under **Resources** it is also recommended to raise the memory limit to at least 4GB.

There are two methods for building `linux/amd64` images. The default uses `repo2docker`'s support for `docker-py`, while the second uses a `repo2docker` plugin that can invoke your local docker command-line interface.

### docker-py (default)

Run `jupyter-repo2docker` with the following arguments:

``` bash
repo2docker \
  --Repo2Docker.platform=linux/amd64 \
  -e PLAYWRIGHT_BROWSERS_PATH=/srv/conda \
  --user-id=1000 --user-name=jovyan \
  --target-repo-dir=/home/jovyan/.cache \
  .
```

where the final parameter is the path to the assets or `.` if they are in the current directory.

The `--user-id` and `--user-name` options are for non-Dockerfile based builds. Images with Dockerfiles do not need those options.

Note that you may see (possibly harmless) architecture mismatch warnings with this method.

### `docker` CLI

You can instruct `repo2docker` to use your machine's local `docker` executable directly rather than the default of `docker-py`. You will first need to install [repo2podman](https://github.com/manics/repo2podman), a plugin that lets you use any container runtime with a command-line user interface similar to that of `docker`. This is useful if you want to leverage [docker buildx](https://github.com/docker/buildx/) (for things like multi-stage builds) or if you want to use an alternative executable like `podman`. This also eliminates architecture mismatch warnings.

:::{admonition} WSL + repo2podman
:class: warning
repo2podman reportedly does not work yet on WSL.
:::

``` bash
repo2docker \
  --Repo2Docker.platform=linux/amd64 \
  -e PLAYWRIGHT_BROWSERS_PATH=/srv/conda \
  --engine podman --PodmanEngine.podman_executable=docker \
  .
```
