# Run the 5G simulation

## Deploy the simulator

Create a new juju model named `sim`:

```console
juju add-model sim
```

Deploy the `sdcore-gnbsim` operator

```console
juju deploy sdcore-gnbsim --trust
```

## Run the simulation

```console
juju run sdcore-gnbsim/leader start-simulation
```

## Monitor the network

Now go to your Prometheus dashboard
