# Hexops' Dockerfile best practices <a href="https://hexops.com"><img align="right" height="36px" alt="Hexops logo" src="https://raw.githubusercontent.com/hexops/media/master/logo_whitebg.svg"></img></a>

Writing production-worthy Dockerfiles is, unfortunately, not as simple as you would imagine. Most Docker images in the wild fail here, and even professionals often[[1]](https://github.com/docker-library/postgres/issues/175) get[[2]](https://github.com/prometheus/prometheus/issues/3441) this[[2]](https://github.com/caddyserver/caddy-docker/issues/104) wrong[[3]](https://github.com/docker-library/postgres/issues/796).

This repository has best-practices for writing Dockerfiles that I (@slimsag) have quite painfully learned over the years both from my personal projects and from my work @sourcegraph. This is all guidance, not a mandate - there may sometimes be reasons to not do what is described here, but if you _don't know_ then this is probably what you should be doing.

## How to use this

Copy https://github.com/hexops/dockerfile/blob/master/Dockerfile into your own project and follow the comments to create _your_ Dockerfile.

## Best practices included in the Dockerfile

The following are included in the Dockerfile in this repository:

- [Run as a non-root user](#run-as-a-non-root-user)
- [Do not use a UID below 10,000](#do-not-use-a-uid-below-10-000)
- [Use a static UID and GID](#use-a-static-uid-and-gid)
- [Do not use `latest`, pin your image tags](#do-not-use-latest-pin-your-image-tags)
- [Use `tini` as your ENTRYPOINT](#use-tini-as-your-entrypoint)
- [Only store arguments in `CMD`](#only-store-arguments-in-cmd)
- [Install bind-tools if you care about DNS resolution on some older Docker versions](#install-bind-tools-if-you-care-about-dns-resolution-on-some-older-docker-versions)

## Run as a non-root user

Running containers as a non-root user substantially decreases the risk that container -> host priviledge escalation could occur. This is an added security benefit. ([Docker docs](https://docs.docker.com/engine/security/#linux-kernel-capabilities), [Bitnami blog post](https://engineering.bitnami.com/articles/why-non-root-containers-are-important-for-security.html))

## Do not use a UID below 10,000

UIDs below 10,000 are a security risk on several systems, because if someone does manage to escalate priviledges outside the Docker container their Docker container UID may overlap with a more priviledged system user's UID granting them additional permissions. For best security, always run your processes as a UID above 10,000.

## Use a static UID and GID

Eventually someone dealing with your container will need to manipulate file permissions for files owned by your container. If your container does not have a static UID/GID, then one must extract this information from the running container before they can assign correct file permissions on the host machine. It is best that you use a single static UID/GID for all of your containers that never changes. We suggest `10000:10001` such that `chown 10000:10001 files/` always works for containers following these best practices.

## Do not use `latest`, pin your image tags

We suggest pinning image tags using a specific image `version` using `major.minor`, not `major.minor.patch` so as to ensure you are always:

1. Keeping your builds working (`latest` means your build can arbitrarily break in the future, whereas `major.minor` _should_ mean this doesn't happen)
2. Getting the latest security updates included in new images you build.

You may also consider using `major.minor.patch` or SHA pinning, which can guarantee the image doesn't change _but_ at the cost of having to ensure you always update the SHA to include new security fixes when you build new versions of your images. This is a tradeoff, see e.g. [this blog post](https://rockbag.medium.com/why-you-should-pin-your-docker-images-with-sha-instead-of-tags-fd132443b8a6) - others may advise you use SHA pinning, but if you do this you _really_ should look at how you are going to handle security fixes in your third-party Docker images because you need to ensure you release a new version of your image with the insecure dependency images updated (you should do this anyway using tooling like container registry security scanners, but using `major.minor` tags helps make this less likely if you are NOT going to do that.)

## Use `tini` as your ENTRYPOINT

We suggest using [tini](https://github.com/krallin/tini) as the ENTRYPOINT in your Dockerfile, even if you think your application handles signals correctly. This can alter the stability of the host system and other containers running on it, if you get it wrong in your application. See the [tini docs](https://github.com/krallin/tini) for details and benefits:

> Using Tini has several benefits:
>
> * It protects you from software that accidentally creates zombie processes, which can (over time!) starve your entire system for PIDs (and make it unusable).
> * It ensures that the default signal handlers work for the software you run in your Docker image. For example, with Tini, SIGTERM properly terminates your process even if you didn't explicitly install a signal handler for it.
> * It does so completely transparently! Docker images that work without Tini will work with Tini without any changes.

## Only store arguments in `CMD`

By having your `ENTRYPOINT` be your command name:

```Dockerfile
ENTRYPOINT ["/sbin/tini", "--", "myapp"]
```

And `CMD` be only arguments for your command:

```Dockerfile
CMD ["--foo", "1", "--bar=2"]
```

It allows people to eronomically pass arguments to your binary without having to guess its name, e.g. they can write:

```sh
docker run yourimage --help
```

If `CMD` includes the binary name, then they must guess what your binary name is in order to pass arguments etc.

## Install bind-tools if you care about DNS resolution on some older Docker versions

If you want your Dockerfile to run on old/legacy Linux systems and Docker for Mac versions and wish to avoid DNS resolution issues, install bind-tools.

For additional details [see here](https://github.com/sourcegraph/godockerize/commit/5cf4e6d81720f2551e6a7b2b18c63d1460bbbe4e#commitcomment-45061472).
