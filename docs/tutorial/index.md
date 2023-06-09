# Tutorial

In this tutorial, we will deploy and run the SD-Core 5G core network using Juju. We will also
deploy a radio and cellphone simulator to simulate usage of this network.

Charmed 5G is a complex piece of software.

This tutorial will introduce you to key concepts, tools, processes and
operations, starting from your first installation to a cloud deployment.
Along the way it will give examples of good practice, and pointers to much
more detailed information.

You can expect to spend one to two hours working through the complete
tutorial. It’s a strongly-recommended investment of time if you’re new to
Charmed 5G - it will save you many more hours later on. Follow the
tutorial steps in sequence; they take you on a learning journey through the
product.

To complete this tutorial, you will need a machine with the following
requirements:

- A recent `x86_64` CPU (Intel 4ᵗʰ generation or newer, or AMD Ryzen or newer)
- 8GB of RAM
- 50GB of free disk space

The tutorial has been tested with a variety of users. We make every effort to
keep it up-to-date and ensure that it’s reliable - but if you encounter any
problems, we want to help you, so please let us know.

## Table of content
```{toctree}
:maxdepth: 1

01_bootstrap_juju
02_deploy_sdcore
03_configure_sdcore
04_integrate_sdcore_with_observability
05_run_simulation
06_destroy_environment
```
