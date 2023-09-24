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

The charmed 5G image of the SMF is less than half the size of the upstream image. This level of size reduction is typical for all Charmed 5G images with the total size of the SD-Core workload being around 1GB in total instead of 2GB in the upstream project.

In addition to being small, the images built on a weekly schedule. This means that the images are always up-to-date with the latest security patches and bug fixes.

## TLS everywhere

Charmed 5G uses TLS everywhere. This means that all communication between 5G network functions is encrypted. This is done by default using the `tls-certificates` charm relation interface and is not optional.

Each Charmed 5G charm generates its own private key and a certificate signing request (CSR). The CSR is then sent to the TLS certificate provider, which signs the certificate and returns it to the charm. The 5G network function then uses the certificate to encrypt all communication with other network functions.

By default, the TLS certificate provider is the [self-signed-certificates operator](https://charmhub.io/self-signed-certificates).
