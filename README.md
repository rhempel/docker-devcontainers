## Docker Base Image Builds

This builds a Docker image (not a devcontainer) that can be used to
build a devcontainer.

Why do it like this? Mainly to speed up the creation of multiple
devcontiners for embedded systems development.

The idea is to do the hard work of creating an image suitable
for a variety of ARM related embedded development, including
compiling, debugging, unit testing, and documenting your project.

It also has a suite of Python tools.

Containers are awesome tools to simplify your development process. To
learn more about how they work, I can recommend [this 'zine by Julia
Evans][how-containers-work].

If you have ever wondered how containers get their default names, then
[this][container-names] file will be interesting.

## Starting a Build

The version of Docker Desktop used for these instructions is 4.32.0 or
higher. The traditional `build` command is now legacy, so we will
be using `buildx` instead.

If you need more control over the process, refer to the
[Docker CLI reference for `buildx`][buildx].

Resist the urge to customize this base image, we are going to
make those changes when we build the devcontainer from this
image. 

In the examples below, we refer to the Ubuntu version as `xx.yy`.
Use the actual version in the commands below.

On Windows, you will need to use the `\` as a file separator.

The simplest way to kick off the build of a new image is:

```
cd path/to/ubuntu-xx.yy-embedded-docker
docker buildx build -f Dockerfile -t ubuntu-embedded:xx.yy .

NOTE: Don't forget the trailing "." 
```

If you want to avoid changing to the directory where the `Dockerfile`
lives, you can just do this:

```
docker buildx build -t ubuntu-embedded:xx.yy path/to/ubuntu-xx.yy-embedded-docker
```

Depending on network and computer speed, the build time will vary. On my
laptop it's about 5 minutes without the cache - and not even 5 seconds with
the cache.

## Renaming Docker Images

Building a Docker image from the instructions above will result in
an image with the name `ubuntu-embedded:xx.yy`, but what if you want
to change that name?

Docker Desktop doesn't provide a rename function, but you can create as
many names as you want for an image.

If you need more control over the tag process, refer to the
[Docker CLI reference for `tag`][tag].

You can rename an image like this:

```
docker image tag ubuntu-embedded:22.04 some-new-name:some-new-tag
```

Sometimes the orignal tag is a really long string, for example a VSCode
devcontainer has a hash after the name.

`vsc-adaptabuild-example-614c6be1beb9a2a0db595c65c0d81774b6cd195b6092ff5ca691bd219d5caaa6:latest`

That's not going to be fun, even with copy/paste. So we can substitute
the Image ID for the name, which can be copied from the Docker Desktop
Image tab, or from the command line:

```
docker image list

REPOSITORY                           TAG    IMAGE ID     CREATED        SIZE
ubuntu-foo                           22.04  3b63e27dcd9f 31 minutes ago 2.24GB
vsc-adaptabuild-example-614c6be1b... latest 047565335b81 16 hours ago   2.25GB
plantuml/plantuml-server             jetty  cba36f940161 6 weeks ago    483MB
...
```

So making that VSCode Image more friendly looks like this:

```
docker image tag 047565335b81 some-new-name:some-new-tag
```

## `apt` Pinning - Ensure Secure and Repeatable Images

One of the problems we are trying to solve with containers is a repeatable
build environment. This can avoid "it works on my machine" problems and
keeps the development team happy. It also keeps the legacy support team
happy because they can recreate the build environment.

This is true for as long as DockerHub is around, and for as long as
the the Linux distro provides a package manager, and for as long
as package maintainers keep the packages available.

For this reason, it is HIGHLY recommended that your team keeps an
archive of mission critical Docker images and containers somewhere
safe. For more details on saving images read the [Docker CLI reference
for `image save`][image-save].

The `Dockerfile` is where we specify which packages are to be added
the base Ubuntu image - but how do we ensure that the package versions
stay stable, and still receive security updates? The secret lies
in [`apt` pinning][apt-pinning]

We want to make sure that specific versions of the essential packages
are retreived, but the Ubuntu LTS releases do provide security updates
from time to time. In the past this meant updating the `Dockerfile` with
more specific version strings, but now we encapsulate this in the
`apt.preferences` file that is copied to `/etc/apt` in the image creation
step.

A good example is the `ssh` package, it's at the heart of your container
network access security management. Here is the section from
`apt.preferences` for that package:

```
Package: ssh
Pin: version 1:8.9*
Pin-Priority: 1000
```

This says that version `1:8.9` followed by any additional characters
will be installed. The whole point of the `apt` package manager is
to keep your system up to date. Currently the preferred version of the
`ssh` package for Ubuntu 22.04 (jammy) is:

```
jammy (22.04LTS) (net): secure shell client and server (metapackage)
1:8.9p1-3ubuntu0.10 [security]: all
```

When a security update is made, any old package versions are *deleted* from
the Ubuntu package manager - this makes sense because you don't want to
install a packe version with a known security issue.

As long as the Ubuntu Jammy package maintainers keep the version of
`ssh` at `8.9` we are good. Ubuntu 22.04 LTS is by definition the Long Term
Support version, so we can be confident that the version number of `ssh`
is unlikely to change, except for security patches.

## Troubleshooting Build Issues

Sometimes the image won't build. Here are a few common failures:

### Failure to Download Packages

We are building on the official Ubuntu images from DockerHub, and in
turn those images will update packages from other servers. Sometimes
those servers are down, in which case trying later helps.

If you are using an externally managed computer, then VPN or other
network settings may be preventing access to certain IP addresses. Consult
your provider for instructions on how to proceed to access external
IP addresses.

For example, you might run into trouble downloading the `plantuml` file
from `github.com`:

```
0.570 Resolving objects.githubusercontent.com ...
0.630 Connecting to objects.githubusercontent.com ...
0.795 ERROR: cannot verify objects.githubusercontent.com's certificate ...
0.795   Unable to locally verify the issuer's authority.
0.795 To connect to objects.githubusercontent.com insecurely, use `--no-check-certificate'.                             -
```

Yeah, don't start modifying the `Dockerfile` - the problem is most likely
your network settings. In my case I just had to (temporarliy) disable the
`ZScaler` Internet Security setting on my managed laptop. Note that it
was still on the VPN and that our organization sets a timeout so that
if you forget to turn it back on you are still safe.

### Cache Confusion

Having a cache of previously created container layers really helps to
speed up container construction - but sometimes the cache works against
you. One way around this is to force a build without the cache:

```
docker buildx build --no-cache -t ubuntu-embedded:xx.yy path/to/ubuntu-xx.yy-embedded-docker
```

As an absolute last resort, you can consider clearing out your entire
Docker cache - this can actually free up quite a bit of disk space, but you
will slow down any subsequent container builds until the cache is refilled.

```
docker buildx prune -f 
```

[buildx]: https://docs.docker.com/reference/cli/docker/buildx/build/
[tag]: https://docs.docker.com/reference/cli/docker/image/tag/
[image-save]: https://docs.docker.com/reference/cli/docker/image/save/
[apt-pinning]:  https://help.ubuntu.com/community/PinningHowto
[how-containers-work]: https://jvns.ca/blog/2020/04/27/new-zine-how-containers-work/
[container-names]: https://github.com/moby/moby/blob/master/pkg/namesgenerator/names-generator.go