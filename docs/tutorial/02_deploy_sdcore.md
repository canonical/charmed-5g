# Deploy SD-Core

Now we will deploy the `sdcore` charm bundle that contains a complete 5G core network.

Create a Juju model named `core`:

```console
juju add-model core
```

Deploy the `sdcore` charm bundle:

```console
juju deploy sdcore --trust
```

Deploying the core network can take up to 15 minutes. You can validate the status of the 
deployment by running `juju status`. The core network is ready when all the charms are in the
`Active/Idle` state.
