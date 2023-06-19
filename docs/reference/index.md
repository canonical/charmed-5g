# Reference

## SD-Core deployment options

`````{tab-set}
    
````{tab-item} Single site deployment

Use the `sdcore` bundle to deploy a standalone 5G core network.
This bundle contains the 5G control plane functions, the UPF, Webui, Grafana Agent, Self Signed 
Certificates and MongoDB.

```{image} ../images/sdcore_single_site.png
:alt: Single site deployment
:height: 600
:align: center
```

````

````{tab-item} Edge deployment

Use the `sdcore-control-plane` to deploy the 5G control plane in a central place and the 
`sdcore-user-plane` bundle to deploy the 5G user plane in edge sites.

```{image} ../images/sdcore_edge.png
:alt: Edge deployment
:height: 600
:align: center
```

````

`````
