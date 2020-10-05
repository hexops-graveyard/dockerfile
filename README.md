# Hexops' Dockerfile - the Dockerfile you should start with

This is a Dockerfile that describes a few important things often subtly missed by people writing Dockerfiles:

- Using a non-root user for security purposes.
- Using a static GID/UID for both security and easier `chmod`/`chown` on the host system.
- Avoiding low-range GID/UIDs which are a security risk.
- A `tini` `ENTRYPOINT` for proper signall handling, because your application most likely does not.
- Installing `bind-tools` for DNS resolution in some more obscure Docker environments.
