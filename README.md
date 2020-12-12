# Hexops' Dockerfile: the Dockerfile you should start with <a href="https://hexops.com"><img align="right" height="36px" alt="Hexops logo" src="https://raw.githubusercontent.com/hexops/media/master/logo_whitebg.svg"></img></a>

Writing production-worthy Dockerfiles is, unfortunately, not as simple as you would imagine. Even professionals often[1](https://github.com/docker-library/postgres/issues/175) get[2](https://github.com/prometheus/prometheus/issues/3441) this[2](https://github.com/docker-library/postgres/pull/791) wrong[3](https://github.com/docker-library/postgres/issues/796).

These are best-practices for writing Dockerfiles I (@slimsag) have, often quite painfully, learned over the years both from my personal projects and from my work @sourcegraph. This is all guidance, not a mandate - there may sometimes be reasons to not do what is described here, but if you _don't know_ then this is probably what you should be doing.

## How can I use this?

Copy https://github.com/hexops/dockerfile/blob/master/Dockerfile into your own project and follow the comments.

## Dockerfile best practices

### Pin image tags using a version instead of `latest`

We suggest pinning image tags using a specific image `version` (`major.minor`, not `major.minor.patch`) so as to ensure you are always:

1. Keeping your builds working (`latest` means your build can arbitrarily break in the future, whereas `major.minor` _should_ mean this doesn't happen.)
2. Getting the latest security updates included in new images you build. By using `major.minor.patch` or SHA pinning, you can guarantee the image doesn't change but at the cost of having to ensure you always update the SHA to include new security fixes (this is a tradeoff, see e.g. [this blog post](https://rockbag.medium.com/why-you-should-pin-your-docker-images-with-sha-instead-of-tags-fd132443b8a6).)

#### Run as a non-root user

Running containers as a non-root user substantially decreases the risk that container -> host priviledge escalation could occur. This is an added security benefit. ([Docker docs](https://docs.docker.com/engine/security/#linux-kernel-capabilities), [Bitnami blog post](https://engineering.bitnami.com/articles/why-non-root-containers-are-important-for-security.html))

#### Do not use a UID below 10,000

UIDs below 10,000 are a security risk on several systems, because if someone does manage to escalate priviledges outside the Docker container their Docker container UID may overlap with a more priviledged system user's UID granting them additional permissions. For best security, always run your processes as a UID above 10,000.

#### Use a static UID and GID

Eventually someone dealing with your container will need to manipulate file permissions for files owned by your container. If your container does not have a static UID/GID, then one must extract this information from the running container before they can assign correct file permissions on the host machine. It is best that you use a single static UID/GID for all of your containers that never changes. We suggest `10000:10001` such that `chown 10000:10001 files/` always works for containers following these best practices.

#### Use `tini` as your ENTRYPOINT

We suggest using [tini](https://github.com/krallin/tini) as the ENTRYPOINT in your Dockerfile, even if you think your application handles signals correctly. This can alter the stability of the host system and other containers running on it, if you get it wrong in your application. See the [tini docs](https://github.com/krallin/tini) for details and benefits:

> Using Tini has several benefits:
>
> * It protects you from software that accidentally creates zombie processes, which can (over time!) starve your entire system for PIDs (and make it unusable).
> * It ensures that the default signal handlers work for the software you run in your Docker image. For example, with Tini, SIGTERM properly terminates your process even if you didn't explicitly install a signal handler for it.
> * It does so completely transparently! Docker images that work without Tini will work with Tini without any changes.

#### Install bind-tools if you care about DNS resolution on some older Docker versions

If you want your Dockerfile to run on old/legacy Linux systems and Docker for Mac versions and wish to avoid DNS resolution issues, install bind-tools.

For additional details [see here](https://github.com/sourcegraph/godockerize/commit/5cf4e6d81720f2551e6a7b2b18c63d1460bbbe4e#commitcomment-45061472).

#### Make `CMD` your arguments only

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
