# ibm-grafana-ocpthanos-proxy 

> A simple proxy between Grafana and Openshift Container Platform (OCP) thanos-querier service with multi-tenancy enabled

This is a fork of the IBM project [ibm-grafana-ocpthanos-proxy](https://github.com/IBM/ibm-grafana-ocpthanos-proxy).

OpenShift 4.6 enables [Monitoring for User-Defined Projects](https://docs.openshift.com/container-platform/4.6/monitoring/enabling-monitoring-for-user-defined-projects.html). This allows users to create ServiceMonitor and PrometheusRules resources in their own namespaces to monitor and send alerts for their own workloads. Using the OpenShift web consonle users are restricted to viewing metrics of workloads in their own namespaces.

When using Grafana to monitor these workloads a token of a Service Account or User with the "cluster-monitoring-view" role is needed to connect to the Thanos Querier at https://thanos-querier.openshift-monitoring.svc:9091. This has the unfortunate effect of exposing metrics of ALL namespaces. The ibm-grafana-ocpthanos-proxy is placed in between Grafana and the Thanos Querier and filters queries by enforcing or applying a namespace (or any other label) filter. This enables tenants to deploy their own Grafana and only see their own metrics.

## Command line options

- --listen-address
  The address ibm-grafana-ocpthanos-proxy should listen on. Default value: 127.0.0.1:9096
- --url-prefix
  url prefix of the proxy. Default value is "/"
- --thanos-address
  The address of thanos-querier service. Default value: `https://thanos-querier.openshift-monitoring.svc:9091`
- --ns-parser-conf
  NSParser configurate file location. Default value: "/etc/conf/ns-config.yaml"
- --thanos-token-file
  The token file passed to OCP thanos-querier service for authentication. Default value: "/var/run/secrets/kubernetes.io/serviceaccount/token"
- --ns-label-name
  The name of metrics' namespace label. Defalut value: namespace

## Getting Started

The example below should not be used in production environments. In production environments it should be used as sidecar of the Grafana Pod and listen to the loopback interface only.
1. Enable OCP Application monitoring according to [OCP official document](https://docs.openshift.com/container-platform/4.6/monitoring/enabling-monitoring-for-user-defined-projects.html).
2. Set up this proxy as a standalone OCP service. For reference, view "example/openshift.yaml".
3. Install Grafana into your cluster. [The Grafana Operator](https://operatorhub.io/operator/grafana-operator) works pretty well.
4. Configure thanos-proxy as datasource of Grafana
   - Use the `Prometheus` datasource type.
   - Use `http://thanos-proxy:9096` as HTTP URL
   - Use the "GET" method instead of "POST". This proxy only supports "GET".

The "example" directory contains an example Kubernetes manifest and ns-list.yaml.

## Configuration files

The proxy uses two important files:
1. ns-config.yaml - this is where the filtered namespaces (or labels) are configured.
2. token - the token for Thanos Querier with "cluster-monitoring-view" role.

### ns-config.yaml

The ns-config.yaml can be mounted onto the filesystem from a configmap. The default location is "/etc/conf/ns-config.yaml". The file contains a simple list with namespaces. Since the query sent to Thanos Querier by the proxy uses the "=~" operator in the query, regex expressions can be used to create "wildcard" namespaces.

Simple list of namespaces that are allowed:

```yaml
type: ns-list
paras:
  namespaces:
  - "namespace-1"
  - "namespace-2"
```

Wildcard format, any namespace starting with "namespace-" will be allowed:

```yaml
type: ns-list
paras:
  namespaces:
  - "namespace-.+"
```

Keep in mind that the default is to filter using the "namespace" label. Any other label can be used by using "--ns-label-name" command-line argument.

### token

The token can be mounted onto the filesystem from a secret. The default location is "/var/run/secrets/kubernetes.io/serviceaccount/token". Following can be used to retrieve the token:

```bash
#!/bin/bash
SECRET=`oc get secret -n openshift-user-workload-monitoring | grep  prometheus-user-workload-token | head -n 1 | awk '{print $1 }'`
TOKEN=`echo $(oc get secret $SECRET -n openshift-user-workload-monitoring -o json | jq -r '.data.token') | base64 -d`

# Print token in base64. Use -w0 to prevent multi-line output.
echo -n $TOKEN | base64 -w0
```

## Limitations

1. The proxy does not provide TLS encryption and any authentication/authorization. It is expected to be used as sidecar of Grafana pod and listen to loopback interface only.
2. The proxy uses the namespace label as matcher for multi-tenancy. So the query in Grafana should only use following format matcher for namespace label.
    - No namespace matcher at all. The query will be updated to `metric_name{namespace=~"namespace1|namespace2"}`
    - Use Equal operator only. `{namespace="namespace1"}` for example. If `namespace1` is allowed by NSParser, the query will be passed onto thanos service without change. Otherwise empty data will be returned.
    - Use simple `=~` operator. `{namespace=~"namespace1|namespace2"}` for example. The query will be passed onto thanos service only if both namespace1 and namespace2 are allowed by NSParser. Otherwise empty data will be returned.