+++
title = "Using Volumes With Docker in Docker"
date = 2019-12-27T20:26:22+11:00
draft = false
tags = []
categories = []
+++

Docker in Docker (DIND) is a way of accessing a docker container within another docker container. It is commonly used in CI/CD pipelines as it allows the build server to be run in a docker container and allows the build pipeline that is run on the build server to also be run in a docker container.

The usual way this is done is to link the docker socket from the host to the build server container.

For example:

```bash
docker run -it -d -p 8080:8080 -v /var/run/docker.sock:/var/run/docker.sock jenkins-docker
```

Where `/var/run/docker.sock` is the socket on the host.

A major downside of this approach is that the socket used inside the container does not have access to the same filesystem that the host has. This limits the volumes that can be mounted inside the docker container for the build process to only those from the host. The result is that any volumes that need to be mounted to the inner container need to be mounted to the build server container (e.g. jenkins-docker) first.

This can be quite cumbersome in a build pipeline when the user creating and running the build does not have control over mounts of the build server.

The workaround I have found is to create a volume inside the build server container as part of the build pipeline, copy the required files into the volume, then attach the volume to the container that is doing the build.

I will illustrate with an example from https://github.com/tide-org/tide that runs on Shippable inside a build server container using DIND:

### The steps to implement this are:

There was a requirement for another repository in the build process, so this was cloned into the workspace.

```bash
git clone https://github.com/tide-org/tide-plugins $SHIPPABLE_BUILD_DIR/plugins
```

A temporary volume `temp-shippable-volume` is created to hold the application inside the DIND container:

```bash
docker volume create temp-shippable-volume
```

A temporary container is then created to mount the `temp-shippable-volume` volume to:

```bash
docker container create --name temp-shippable-container -v temp-shippable-volume:/app busybox
```

The required files from the workspace are then copied into the `temp-shippable-volume` volume:

```bash
docker cp $SHIPPABLE_BUILD_DIR temp-shippable-container:/app
```

The temp container is then removed:

```bash
docker rm temp-shippable-container
```

A listing of volumes in the container verifies the `temp-shippable-volume` is present:

```bash
docker volume ls
```

An image is created and tagged from the Docker file:

```bash
docker build -t shippable-python - < tests/docker/Dockerfile
```

The docker image build in the previous step is run with the `temp-shippable-volume` mounted to the correct location `(-v)`.
An environment variable is set for the tests `(-e)`.
The working directory is set inside the build container `(-w)` and;
A command line is run inside the DIND container `(sh -c "...")`:

```bash
docker run --rm \
  -v temp-shippable-volume:/work \
  -e PYTHONPATH=/work/tide \
  -w /work/tide \
  shippable-python sh -c "cd /work/tide/tests/scripts && ./run-python-tests"
```

The temp-shippable-volume is then removed from the build server to prevent it being used again:

```bash
docker volume rm temp-shippable-volume
```

The full steps of the shippable.yaml were:

```yaml
build:
  ci:
    - git clone https://github.com/tide-org/tide-plugins $SHIPPABLE_BUILD_DIR/plugins
    - docker volume create temp-shippable-volume
    - docker container create --name temp-shippable-container -v temp-shippable-volume:/app busybox
    - docker cp $SHIPPABLE_BUILD_DIR temp-shippable-container:/app
    - docker rm temp-shippable-container
    - docker volume ls
    - docker build -t shippable-python - < tests/docker/Dockerfile
    - >-
        docker run --rm \
          -v temp-shippable-volume:/work \
          -e PYTHONPATH=/work/tide \
          -w /work/tide \
          shippable-python sh -c "cd /work/tide/tests/scripts && ./run-python-tests" \
    - docker volume rm temp-shippable-volume
```

This took me a few hours to figure out so I decided to write it up in the hope it helps someone else avoid this time-sink.
