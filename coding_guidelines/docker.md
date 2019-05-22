# Best practices for writing Dockerfiles

This document covers recommended best practices and methods for building efficient images.

You can find more details in the (official Docker documentation)[https://docs.docker.com/develop/develop-images/dockerfile_best-practices/]
Docker builds images automatically by reading the instructions from a Dockerfile -- a text file that contains all commands, in order, needed to build a given image. A Dockerfile adheres to a specific format and set of instructions which you can find at Dockerfile reference.

A Docker image consists of read-only layers each of which represents a Dockerfile instruction. The layers are stacked and each one is a delta of the changes from the previous layer. Consider this Dockerfile:

```
FROM ubuntu:18.04
COPY . /app
RUN make /app
CMD python /app/app.py
```

Each instruction creates one layer:

FROM creates a layer from the ubuntu:18.04 Docker image.
COPY adds files from your Docker client’s current directory.
RUN builds your application with make.
CMD specifies what command to run within the container.
When you run an image and generate a container, you add a new writable layer (the “container layer”) on top of the underlying layers. All changes made to the running container, such as writing new files, modifying existing files, and deleting files, are written to this thin writable container layer.

For more on image layers (and how Docker builds and stores images), see About storage drivers.

## General guidelines and recommendations
### Create ephemeral containers
The image defined by your Dockerfile should generate containers that are as ephemeral as possible. By “ephemeral”, we mean that the container can be stopped and destroyed, then rebuilt and replaced with an absolute minimum set up and configuration.

Refer to Processes under The Twelve-factor App methodology to get a feel for the motivations of running containers in such a stateless fashion.

### Use multi-stage builds
Multi-stage builds allow you to drastically reduce the size of your final image, without struggling to reduce the number of intermediate layers and files.

Because an image is built during the final stage of the build process, you can minimize image layers by leveraging build cache.

For example, if your build contains several layers, you can order them from the less frequently changed (to ensure the build cache is reusable) to the more frequently changed:

Install tools you need to build your application

Install or update library dependencies

Generate your application

A Dockerfile for a Go application could look like:

FROM golang:1.11-alpine AS build

# Install tools required for project
# Run `docker build --no-cache .` to update dependencies
RUN apk add --no-cache git
RUN go get github.com/golang/dep/cmd/dep

# List project dependencies with Gopkg.toml and Gopkg.lock
# These layers are only re-built when Gopkg files are updated
COPY Gopkg.lock Gopkg.toml /go/src/project/
WORKDIR /go/src/project/
# Install library dependencies
RUN dep ensure -vendor-only

# Copy the entire project and build it
# This layer is rebuilt when a file changes in the project directory
COPY . /go/src/project/
RUN go build -o /bin/project

# This results in a single layer image
FROM scratch
COPY --from=build /bin/project /bin/project
ENTRYPOINT ["/bin/project"]
CMD ["--help"]
Don’t install unnecessary packages
To reduce complexity, dependencies, file sizes, and build times, avoid installing extra or unnecessary packages just because they might be “nice to have.” For example, you don’t need to include a text editor in a database image.

### Decouple applications
Each container should have only one concern. Decoupling applications into multiple containers makes it easier to scale horizontally and reuse containers. For instance, a web application stack might consist of three separate containers, each with its own unique image, to manage the web application, database, and an in-memory cache in a decoupled manner.

Limiting each container to one process is a good rule of thumb, but it is not a hard and fast rule. For example, not only can containers be spawned with an init process, some programs might spawn additional processes of their own accord. For instance, Celery can spawn multiple worker processes, and Apache can create one process per request.


### Minimize the number of layers
In older versions of Docker, it was important that you minimized the number of layers in your images to ensure they were performant. The following features were added to reduce this limitation:

Only the instructions RUN, COPY, ADD create layers. Other instructions create temporary intermediate images, and do not increase the size of the build.

Where possible, use multi-stage builds, and only copy the artifacts you need into the final image. This allows you to include tools and debug information in your intermediate build stages without increasing the size of the final image.

### Sort multi-line arguments
Whenever possible, ease later changes by sorting multi-line arguments alphanumerically. This helps to avoid duplication of packages and make the list much easier to update. This also makes PRs a lot easier to read and review. Adding a space before a backslash (\) helps as well.

Here’s an example from the buildpack-deps image:

```
RUN apt-get update && apt-get install -y \
  bzr \
  cvs \
  git \
  mercurial \
  subversion
```

### Dockerfile instructions
These recommendations are designed to help you create an efficient and maintainable Dockerfile.

#### FROM

Whenever possible, use current official images as the basis for your images. We recommend the Alpine image as it is tightly controlled and small in size (currently under 5 MB), while still being a full Linux distribution.

#### RUN

Split long or complex RUN statements on multiple lines separated with backslashes to make your Dockerfile more readable, understandable, and maintainable.

#### APT-GET
Probably the most common use-case for RUN is an application of apt-get. Because it installs packages, the RUN apt-get command has several gotchas to look out for.

Avoid RUN apt-get upgrade and dist-upgrade, as many of the “essential” packages from the parent images cannot upgrade inside an unprivileged container. If a package contained in the parent image is out-of-date, contact its maintainers. If you know there is a particular package, foo, that needs to be updated, use apt-get install -y foo to update automatically.

Always combine RUN apt-get update with apt-get install in the same RUN statement. For example:

```
RUN apt-get update && apt-get install -y \
    package-bar \
    package-baz \
    package-foo
```

Using apt-get update alone in a RUN statement causes caching issues and subsequent apt-get install instructions fail.

Version pinning forces the build to retrieve a particular version regardless of what’s in the cache. This technique can also reduce failures due to unanticipated changes in required packages.

Below is a well-formed RUN instruction that demonstrates all the apt-get recommendations.
```
RUN apt-get update && apt-get install -y \
    aufs-tools \
    automake \
    build-essential \
    curl \
    dpkg-sig \
    libcap-dev \
    libsqlite3-dev \
    mercurial \
    reprepro \
    ruby1.9.1 \
    ruby1.9.1-dev \
    s3cmd=1.1.* \
 && rm -rf /var/lib/apt/lists/*
```
The s3cmd argument specifies a version 1.1.*. If the image previously used an older version, specifying the new one causes a cache bust of apt-get update and ensures the installation of the new version. Listing packages on each line can also prevent mistakes in package duplication.

In addition, when you clean up the apt cache by removing /var/lib/apt/lists it reduces the image size, since the apt cache is not stored in a layer. Since the RUN statement starts with apt-get update, the package cache is always refreshed prior to apt-get install.

Official Debian and Ubuntu images automatically run apt-get clean, so explicit invocation is not required.

#### ADD or COPY
Although ADD and COPY are functionally similar, generally speaking, COPY is preferred. That’s because it’s more transparent than ADD. COPY only supports the basic copying of local files into the container, while ADD has some features (like local-only tar extraction and remote URL support) that are not immediately obvious. Consequently, the best use for ADD is local tar file auto-extraction into the image, as in ADD rootfs.tar.xz /.

If you have multiple Dockerfile steps that use different files from your context, COPY them individually, rather than all at once. This ensures that each step’s build cache is only invalidated (forcing the step to be re-run) if the specifically required files change.

For example:
```
COPY requirements.txt /tmp/
RUN pip install --requirement /tmp/requirements.txt
COPY . /tmp/
```

Results in fewer cache invalidations for the RUN step, than if you put the COPY . /tmp/ before it.

Because image size matters, using ADD to fetch packages from remote URLs is strongly discouraged; you should use curl or wget instead. That way you can delete the files you no longer need after they’ve been extracted and you don’t have to add another layer in your image.

#### USER

If a service can run without privileges, use USER to change to a non-root user. Start by creating the user and group in the Dockerfile with something like RUN groupadd -r postgres && useradd --no-log-init -r -g postgres postgres.

Due to an unresolved bug in the Go archive/tar package’s handling of sparse files, attempting to create a user with a significantly large UID inside a Docker container can lead to disk exhaustion because /var/log/faillog in the container layer is filled with NULL (\0) characters. A workaround is to pass the --no-log-init flag to useradd. The Debian/Ubuntu adduser wrapper does not support this flag.

Avoid installing or using sudo as it has unpredictable TTY and signal-forwarding behavior that can cause problems. If you absolutely need functionality similar to sudo, such as initializing the daemon as root but running it as non-root), consider using “gosu”.

Lastly, to reduce layers and complexity, avoid switching USER back and forth frequently.

#### WORKDIR

For clarity and reliability, you should always use absolute paths for your WORKDIR. Also, you should use WORKDIR instead of proliferating instructions like RUN cd … && do-something, which are hard to read, troubleshoot, and maintain.
