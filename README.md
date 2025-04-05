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

### alloy

[Alloy](https://grafana.com/docs/alloy/latest/) is required to export pod logs for unbind.

If you bring your own, you need to configure it to index unbind specific labels. IE:

```yaml
alloy:
  mounts:
    varlog: true
  configMap:
    content: |
      logging {
        level  = "info"
        format = "logfmt"
      }
      discovery.kubernetes "pods" {
        role = "pod"
        selectors {
          role = "pod"
          label = "unbind-team,unbind-project,unbind-environment,unbind-service"
        }
      }
      discovery.kubernetes "deployment_pods" {
        role = "pod"
        selectors {
          role = "pod"
          label = "unbind-deployment"
        }
      }
      discovery.relabel "pods" {
        targets = discovery.kubernetes.pods.targets
        rule {
          action        = "replace"
          source_labels = ["__meta_kubernetes_pod_label_unbind_team"]
          target_label  = "unbind_team"
        }
        rule {
          action        = "replace"
          source_labels = ["__meta_kubernetes_pod_label_unbind_project"]
          target_label  = "unbind_project"
        }
        rule {
          action        = "replace"
          source_labels = ["__meta_kubernetes_pod_label_unbind_environment"]
          target_label  = "unbind_environment"
        }
        rule {
          action        = "replace"
          source_labels = ["__meta_kubernetes_pod_label_unbind_service"]
          target_label  = "unbind_service"
        }
      }
      discovery.relabel "deployment_pods" {
        targets = discovery.kubernetes.deployment_pods.targets
        rule {
          action        = "replace"
          source_labels = ["__meta_kubernetes_pod_label_unbind_deployment"]
          target_label  = "unbind_deployment"
        }
      }
      loki.source.kubernetes "pods" {
        targets = discovery.relabel.pods.output
        forward_to = [loki.write.endpoint.receiver]
      }
      loki.source.kubernetes "deployment_pods" {
        targets = discovery.relabel.deployment_pods.output
        forward_to = [loki.write.endpoint.receiver]
      }
      loki.write "endpoint" {
        endpoint {
          url = "http://{{ .Environment.Values | get "overrides.alloy.loki.endpoint.host" }}.{{ .Environment.Values | get "overrides.alloy.loki.namespace" .Environment.Values.global.namespace }}.svc.cluster.local:{{ .Environment.Values | get "overrides.alloy.loki.endpoint.port" "80" }}/loki/api/v1/push"
          tenant_id = "local"
        }
      }
```

### Loki

[Loki](https://grafana.com/docs/loki/latest/) is used for querying unbind logs, if you bring your own you should update the alloy configuration to point to the host/namespace/port where it is deployed.
