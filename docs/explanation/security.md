# Security

## Chiseled container images built frequently

Each charm in Charmed 5G is distributed by Canonical as a ROCK container image. These images are built using [Rockcraft](https://canonical-rockcraft.readthedocs-hosted.com/en/latest/), a tool to build secure, stable, and OCI-compliant container images.

Each image is chiseled to contain only the bare minimum to run the application. This means that the images are small, and contain only the necessary dependencies to run the application. This reduces the attack surface of the application and makes it easier to maintain.

Take the SMF image as an example:

```console
ubuntu@server:~$ docker images
REPOSITORY                          TAG             IMAGE ID       CREATED        SIZE
ghcr.io/canonical/sdcore-smf        1.3             e15917b9cce9   18 hours ago   51MB  # Charmed 5G image
omecproject/5gc-smf                 master-latest   d4b790e2c492   4 months ago   104MB  # Upstream image
```

The charmed 5G image of the SMF is less than half the size of the upstream image. This level of size reduction is typical for all Charmed 5G images with the total size of the SD-Core workloads being around 1GB, contrasting with 2GB in the upstream project.

In addition to being small, the images are built on a weekly schedule. This means that the images are always up-to-date with the latest security patches and bug fixes.

## TLS everywhere

Charmed 5G enforces TLS encryption across all communication within the 5G network functions.

Each Charmed 5G charm generates its private key and a certificate signing request (CSR). The CSR is then transmitted to the TLS certificate provider, which in turn signs the certificate and sends it back to the charm. Subsequently, the 5G network function utilizes this certificate to encrypt its communications with other network functions.

By default, the TLS certificate provider employed is the [self-signed-certificates operator](https://charmhub.io/self-signed-certificates). However, users have the flexibility to utilize any TLS certificates provider as long as it supports the `tls-certificates` integration.
