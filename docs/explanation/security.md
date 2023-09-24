# Security

## Chiseled container images

Each charm in Charmed 5G is distributed by Canonical as a ROCK container image. These images are built using [Rockcraft](https://canonical-rockcraft.readthedocs-hosted.com/en/latest/), a tool to build secure, stable and OCI-compliant container images.

Each image is chiseled to contain only the bare minimum to run the application. This means that the images are small, and contain only the necessary dependencies to run the application. This reduces the attack surface of the application, and makes it easier to maintain.

Take the UPF image as an example:

```console
ubuntu@server:~$ docker images
REPOSITORY                          TAG             IMAGE ID       CREATED          SIZE
ghcr.io/canonical/sdcore-upf-bess   1.3             217c37a38b16   7 weeks ago      540MB  # Charmed 5G image
omecproject/upf-epc-bess            master-latest   ad4a5af9e8b5   3 months ago     1.29GB  # upstream image
```

As you can see, the Charmed 5G image is less than half the size of the upstream image.

## TLS everywhere

Charmed 5G uses TLS everywhere. This means that all communication between 5G network functions is encrypted. This is done by default using the `tls-certificates` charm relation interface, and is not optional.

Each charm generates its own private key and the corresponding certificate signing request (CSR). The CSR is then sent to the TLS certificates provider, which signs the certificate and returns it to the charm. The charm then uses the certificate to encrypt all communication with other charms.

By default, the TLS certificates provider is the [self-signed-certificates operator](https://charmhub.io/self-signed-certificates).
