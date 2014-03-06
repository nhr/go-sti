go-sti
======

source-to-images (`sti`) is a tool for building reproducable Docker images.  source-to-images
produces ready-to-run images by injecting a user source into a docker image and /preparing/
a new Docker image which incorporates the base image and built source, and is ready to use
with `docker run`.  source-to-images supports:

1. Incremental builds which re-use previously downloaded dependencies, previously built
   artifacts, etc
1. Build on one image, deploy on another with extended builds

Interested in learning more?  Read on!

### Philosophy

1. Simplify the process of application source + base image -> usable image for most use cases (the 80%)
2. Define and implement a workflow for incremental build that eventually uses only docker primitives
3. Develop tooling that can assist in verifying that two different base images result in the same
   "docker run" outcome for the same input
4. Use native docker primitives to accomplish this - map out useful improvements to docker build that
   benefit all image builders

### Anatomy of a source image

Building source images is as easy as implementing two scripts.  source-to-images expects the
following scripts in `/usr/bin`:

1. `prepare` : This script is responsible for building and/or deploying the source
1. `run`: This script is responsible for running the deployed source

### Basic (`--clean`) builds

source-to-images accepts the following inputs to do a build:

1. Application source: this can be source code, zipped source, a binary, etc
1. Build image: the basis for the new image to build
1. Application image tag: the tag to use for the newly created application image

The basic build process is as follows:

1. source-to-images pulls the build image if it is not already present on the system
1. source-to-images generates a `Dockerfile` to describe the output image:
    1. Based on the build image
    1. Adds the application source at `/usr/src` in the container
    1. Calls `/usr/bin/prepare` in the container
    1. Sets the image's default command to `/usr/bin/run`
1. source-to-images builds the new image from the `Dockerfile` using `docker build`, using the
   supplied tag for a successful build

### Incremental builds

source-to-images automatically detects:

1. Whether a source image is compatible with incremental building
1. Whether an incremental build can be formed when an image is compatible

If the source image is compatible, a prior build already exists, and the `--clean` option is not used,
the workflow is as follows:

1. source-to-images creates a new docker container from the prior build image, with a volume in `/usr/artifacts`
1. source-to-images runs `/usr/bin/save-artifacts` in this container
1. source-to-images creates a new docker container from the build image, mounting the volumes from the
   incremental build container
1. source-to-images generates a `Dockerfile` to describe the output image:
    1. Based on the build image
    1. Adds the application source at `/usr/src` in the container
    1. Adds the build artifacts at `/usr/artifacts` in the container
    1. Runs `/usr/bin/prepare` in the container - this script is responsible for detecting the artifacts
       of the previous build and restoring them
    1. Sets the image's default command to `/usr/bin/run`
1. source-to-images builds the new image from the `Dockerfile` using `docker build`, using the
   supplied tag for a successful build

Note the invocation of the `save-artifacts` script; this script is responsible for moving build
dependencies to `/usr/artifacts`

### Extended builds

Extended builds allow you to execute your build on a build image, then deploy it on a different
runtime image. The workflow for extended builds is as follows:

1. source-to-images looks for the previous build image for the tag, `<tag>-build`.
1. If that image exists:
    1. source-to-images creates a container from this image and runs `/usr/bin/save-artifacts` in it
1. source-to-images creates a build container from the build image with a volume at `/usr/build`
   and bind-mounts in the artifacts from the prior build, if applicable
1. source-to-images runs `/usr/bin/prepare` in the build container - this script is responsible for
   populating `/usr/build` with the result of the build
1. source-to-images generates a `Dockerfile` to describe the output image:
    1. Based on the build image
    1. Adds the result of the build in the build container (ie, build container's `/usr/build` at
       `/usr/src` in the container
    1. Adds the build artifacts at `/usr/artifacts` in the container
    1. Runs `/usr/bin/prepare` in the container - this script is responsible for detecting the artifacts
       of the previous build and restoring them
    1. Sets the image's default command to `/usr/bin/run`
1. source-to-images builds the new image from the `Dockerfile` using `docker build`, using the
   supplied tag for a successful build
1. If the docker build succeeds, the build container is tagged as `<tag>-build`

### Getting started

#### Installation

	go get github.com/pmorie/go-sti/sti

#### Example

You can start using sti right away by using the sample image and application sources in the
`test_sources` directory.  Here's an example that builds a simple HTML app:

	docker build -rm -t fedora-mock test_sources/images/fedora-mock
	sti build test_sources/applications/html fedora-mock sti_app
	docker run -rm -i -p PORT:8080 -t sti_app

### Validating a source image

    sti validate BUILD_IMAGE_TAG [--runtime-image=RUNTIME_IMAGE_TAG] [--incremental] [--url=URL]
        [--timeout=TIMEOUT] [-l LOG_LEVEL]

    Arguments:
        BUILD_IMAGE_TAG        Tag for the Docker image which provides the build and runtime for the application.
        SOURCE_DIR             Directory or GIT repository containing your application sources.
        APP_IMAGE_TAG          Tag for the Docker image which is created by STI. In the case of incremental
                               builds, this tag is also used to identify the previous build of the application.

    Options:
        --runtime-image=RUNTIME_IMAGE_TAG   Tag which identifies an optional Docker image with runtime components but
                                            none of the build dependencies. If provided, the application will be built
                                            with BUILD_IMAGE_TAG and the binaries will be extracted and installed on
                                            the runtime image.
        -l LOG_LEVEL                        Logging level. Default: INFO
        --timeout=TIMEOUT                   Timeout commands if they take too long. Default: 120 seconds.
        --url=URL                           Connect to docker at the specified url [default: unix://var/run/docker.sock]

You can validate that an image is usable as a sti source image as follows:

	sti validate BUILD_IMAGE_TAG

The `--incremental` option to enables validation for incremental builds:

    sti validate BUILD_IMAGE_TAG --incremental

Add the `--runtime-image` option to additionally validate a runtime image for extended builds:

    sti validate BUILD_IMAGE_TAG --runtime-image RUNTIME_IMAGE_TAG

When specifying a runtime image with `sti validate`, the build image is automatically validated for
incremental builds.

### Building a deployable image with sti

    sti build SOURCE_DIR BUILD_IMAGE_TAG APP_IMAGE_TAG [--runtime-image=RUNTIME_IMAGE_TAG] [--clean]
        [--user=USERID] [--url=URL] [--timeout=TIMEOUT] [-e ENV_NAME=VALUE]... [-l LOG_LEVEL]
        [--dir=WORKING_DIR] [--push]

    Arguments:
        BUILD_IMAGE_TAG        Tag for the Docker image which provides the build and runtime for the application.
        SOURCE_DIR             Directory or GIT repository containing your application sources.
        APP_IMAGE_TAG          Tag for the Docker image which is created by STI. In the case of incremental
                               builds, this tag is also used to identify the previous build of the application.

    Options:
        --runtime-image=RUNTIME_IMAGE_TAG   Tag which identifies an optional Docker image with runtime components but
                                            none of the build dependencies. If provided, the application will be built
                                            with BUILD_IMAGE_TAG and the binaries will be extracted and installed on
                                            the runtime image.
        --clean                             Do a clean build, ie. do not perform an incremental build.
        --dir=WORKING_DIR                   Directory where Dockerfiles and other support scripts are created.
                                            (Default: temp dir)
        -l LOG_LEVEL                        Logging level. Default: INFO
        --timeout=TIMEOUT                   Timeout commands if they take too long. Default: 120 seconds.
        --user=USERID                       Perform the build as specified user.
        --url=URL                           Connect to docker at the specified url [default: unix://var/run/docker.sock]

The most basic `sti build` uses a single build image:

    sti build SOURCE_DIR BUILD_IMAGE_TAG APP_IMAGE_TAG

If the build is successful, the built image will be tagged with `APP_IMAGE_TAG`.

If the build image is compatible with incremental builds, `sti build` will look for an image tagged
with `APP_IMAGE_TAG`.  If an image is present with that tag, `sti build` will save the build
artifacts from that image and add them to the build container at `/usr/artifacts` so an image's
`/usr/bin/prepare` script can restore them before building the source.

When using an image that supports incremental builds, you can do a clean build with `--clean`:

    sti build SOURCE_DIR BUILD_IMAGE_TAG APP_IMAGE_TAG --clean

Extended builds allow you to use distinct images for building your sources and deploying them. Use
the `--runtime-image` option perform an extended build targeting a runtime image:

    sti build SOURCE_DIR BUILD_IMAGE_TAG APP_IMAGE_TAG --runtime-image RUNTIME_IMAGE_TAG

When specifying a runtime image, the build image must be compatible with incremental builds.
`sti build` will look for an image tagged with `<APP_IMAGE_TAG>-build`.  If an image is present with
that tag, `sti build` will save the build artifacts from that image and add them to the build
container at `/usr/artifacts` so the build image's `/usr/bin/prepare` script can restore them before
building the source.  The build image's `/usr/bin/prepare` script is responsible for populating
`/usr/build` with an artifact to be deployed into the runtime container.

After performing the build, a new runtime image is created based on the image tagged with
`RUNTIME_IMAGE_TAG` with the output of the build in `/usr/src`.  The runtime image's
`/usr/bin/prepare` script is responsible for detecting and deploying the artifact.  If the build is
successful, two images are tagged:

1. The build image is tagged with `<APP_IMAGE_TAG>-build`
1. The prepared image incorporating the deployed build is tagged with `APP_IMAGE_TAG`

You can do a clean extended build with `--clean`:

    sti build SOURCE_DIR BUILD_IMAGE_TAG APP_IMAGE_TAG --runtime-image RUNTIME_IMAGE_TAG --clean
