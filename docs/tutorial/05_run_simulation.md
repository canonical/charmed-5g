# 5. Run the 5G simulation

## Deploy the simulator

Switch to the `core` Juju model:

```console
juju switch core
```

Deploy the `sdcore-gnbsim` operator

```console
juju deploy sdcore-gnbsim gnbsim --trust --channel=edge
```

Integrate it to the AMF:

```console
juju integrate gnbsim:fiveg-n2 amf:fiveg-n2
```

## Run the simulation

```console
juju run gnbsim/leader start-simulation
```

The simulation executed successfully if you see `success: "true"` as one of the
output messages:

```console
ubuntu@host:~$ juju run gnbsim/leader start-simulation
Running operation 1 with 1 task
  - task 2 on unit-gnbsim-0

Waiting for task 2...
info: run juju debug-log to get more information.
success: "true"
```

## Monitor the network

Now go back to the "5G Network Overview" Grafana dashboard, you should see that there was a spike 
in usage right when you ran the simulation:

```{image} ../images/grafana_5g_dashboard_sim_after.png
:alt: Grafana dashboard
:align: center
```
