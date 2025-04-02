# unbind-charts

Meta [helmfile](https://github.com/roboll/helmfile/tree/master) to manage unbind and all of its dependencies.

## Namespace

By default, and by the [unbind-installer](https://github.com/unbindapp/unbind-installer) all resources are tied to the same namespace.

This gives the unbind service account access to all of its resources (as it has superuser permissions to its own namespace)

## Bringing your own cluster

**Wild card domain**

Unbind expects and requires to take over a domain, this means a wildcard DNS record like `A * 50.51.52.53 mydomain.com`

`mydomain.com` is the value you must set for `global.baseDomain`

You can take over a subdomain as well, `*.sub.mydomain.com`, then your base domain becomes `sub.mydomain.com`

If you bring your own cluster, you may disable certain services. That's fine, but any services that you reference externally may not be configurable from Unbind UI anymore (you will have to manage them yourself, for example changing retention policy on metrics)

## Special cases (for bringing your own cluster)

### kube-state-metrics

If you have a pre-existing monitoring stack, Unbind metrics **will not work** _unless_ you configure pod label exporting.

This can be done in the [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) chart by setting the following:

```
kube-state-metrics:
  extraArgs:
  - --metric-labels-allowlist=pods=[*]
```

### buildkitd

Buildkitd will only schedule once per node, so you should not set replicas > the number of nodes in your cluster or else you'll have scheduling errors.

More replicas = higher build concurrency.
