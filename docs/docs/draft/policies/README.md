# Policies

::: tip
**Need help?** Installing and using Kuma should be as easy as possible. [Contact and chat](/community) with the community in real-time if you get stuck or need clarifications. We are here to help.
:::

Here you can find the list of Policies that Kuma supports, that will allow you to build a modern and reliable Service Mesh.

## Applying Policies

Once installed, Kuma can be configured via its policies. You can apply policies with [`kumactl`](/docs/DRAFT/documentation/#kumactl) on Universal, and with `kubectl` on Kubernetes. Regardless of what environment you use, you can always read the latest Kuma state with [`kumactl`](/docs/DRAFT/documentation/#kumactl) on both environments.

::: tip
We follow the best practices. You should always change your Kubernetes state with CRDs, that's why Kuma disables `kumactl apply [..]` when running in K8s environments.
:::

These policies can be applied either by file via the `kumactl apply -f [path]` or `kubectl apply -f [path]` syntax, or by using the following command:

```sh
echo "
  type: ..
  spec: ..
" | kumactl apply -f -
```

or - on Kubernetes - by using the equivalent:

```sh
echo "
  apiVersion: kuma.io/v1alpha1
  kind: ..
  spec: ..
" | kubectl apply -f -
```

Below you can find the policies that Kuma supports. In addition to [`kumactl`](/docs/DRAFT/documentation/#kumactl), you can also retrive the state via the Kuma [HTTP API](/docs/DRAFT/documentation/#http-api) as well.

## Mesh

This policy allows to create multiple Service Meshes on top of the same Kuma cluster.

On Universal:

```yaml
type: Mesh
name: default
```

On Kuberentes:

```yaml
apiVersion: kuma.io/v1alpha1
kind: Mesh
metadata:
  name: default
```

## Mutual TLS

This policy enables automatic encrypted mTLS traffic for all the services in a [`Mesh`](#mesh).

Kuma ships with a `builtin` CA (Certificate Authority) which is initialized with an auto-generated root certificate. The root certificate is unique for every [`Mesh`](#mesh) and it used to sign identity certificates for every data-plane. Kuma also supports third-party CA.

The mTLS feature is used for AuthN/Z as well: each data-plane is being assigned with a workload identity certificate, which is SPIFFE compatible. This certificate has a SAN set to `spiffe://<mesh name>/<service name>`. When Kuma enforces policies that require an identity, like [`TrafficPermission`](#traffic-permission), it will extract the SAN from the client certificate and use it for every identity matching operation.

By default, mTLS is **not** enabled. You can enable Mutual TLS by updating the [`Mesh`](#mesh) policy with the `mtls` setting.

On Universal with `builtin` CA:

```yaml
type: Mesh
name: default
mtls:
  enabled: true
  ca:
    builtin: {}
```

You can apply this configuration with `kumactl apply -f [file-path]`.

On Kubernetes with `builtin` CA:

```yaml
apiVersion: kuma.io/v1alpha1
kind: Mesh
metadata:
  name: default
spec:
  mtls:
    enabled: true
    ca:
      builtin: {}
```

You can apply this configuration with `kubectl apply -f [file-path]`.

Along with the self-signed certificates (`builtin`), Kuma also supports third-party certificates (`provided`). To use a third-party CA, change the mesh resource to use the `provided` CA. And then you can utilize `kumactl manage ca` to add or delete your certificates

On Universal with `provided` CA:

```yaml
type: Mesh
name: default
mtls:
  enabled: true
  ca:
    provided: {}
```

On Kubernetes with `provided` CA:

```yaml
apiVersion: kuma.io/v1alpha1
kind: Mesh
metadata:
  name: default
spec:
  mtls:
    enabled: true
    ca:
      provided: {}
```

::: tip
With mTLS enabled, traffic is restricted by default. Remember to apply a `TrafficPermission` policy to permit connections
between Dataplanes.
:::

## Traffic Permissions

Traffic Permissions allow you to determine security rules for services that consume other services via their [Tags](/docs/DRAFT/documentation/#tags). It is a very useful policy to increase security in the Mesh and compliance in the organization.

You can determine what source services are **allowed** to consume specific destination services. The `service` field is mandatory in both `sources` and `destinations`.

::: warning
In Kuma DRAFT the `sources` field only allows for `service` and only `service` will be enforced. This limitation will disappear in the next version of Kuma.
:::

In the example below, the `destinations` includes not only the `service` property, but also an additional `version` tag. You can include any arbitrary tags to any [`Dataplane`](/docs/DRAFT/documentation/dataplane-specification)

On Universal:

```yaml
type: TrafficPermission
name: permission-1
mesh: default
sources:
  - match:
      service: backend
destinations:
  - match:
      service: redis
      version: '5.0'
```

On Kubernetes:

```yaml
apiVersion: kuma.io/v1alpha1
kind: TrafficPermission
mesh: default
metadata:
  namespace: default
  name: permission-1
spec:
  sources:
    - match:
        service: backend
  destinations:
    - match:
        service: redis
        version: '5.0'
```

::: tip
**Match-All**: You can match any value of a tag by using `*`, like `version: '*'`.
:::

## Traffic Route

`TrafficRoute` policy allows you to configure routing rules for L4 traffic, i.e. blue/green deployments and canary releases. 

To route traffic, Kuma matches via tags that we can designate to `Dataplane` resources. In the example below, the redis destination services have been assigned the `version` tag to help with canary deployment. Another common use-case is to add a `env` tag as you separate testing, staging, and production environments' services. For the redis service, this `TrafficRoute` policy assigns a positive weight of 90 to the v1 redis service and a positive weight of 10 to the v2 redis service. Kuma utilizes positive weights in the traffic routing policy, and not percentages. Therefore, Kuma does not check if it adds up to 100. If you want to stop sending traffic to a destination service, change the weight for that service to 0.

On Universal:

```yaml
type: TrafficRoute
name: route-1
mesh: default
sources:
  - match:
      service: backend
destinations:
  - match:
      service: redis
conf:
  - weight: 90
    destination:
      service: redis
      version: '1.0'
  - weight: 10
    destination:
      service: redis
      version: '2.0'
```

On Kubernetes:

```yaml
apiVersion: kuma.io/v1alpha1
kind: TrafficRoute
mesh: default
metadata:
  namespace: default
  name: route-1
spec:
  sources:
    - match:
        service: backend
  destinations:
    - match:
        service: redis
  conf:
    - weight: 90
      destination:
        service: redis
        version: '1.0'
    - weight: 10
      destination:
        service: redis
        version: '2.0'
```

## Traffic Tracing

::: warning
This is a proposed policy not in GA yet. You can setup tracing manually by leveraging the [`ProxyTemplate`](#proxy-template) policy and the low-level Envoy configuration. Join us on [Slack](/community) to share your tracing requirements.
:::

The proposed policy will enable `tracing` on the [`Mesh`](#mesh) level by adding a `tracing` field.

On Universal:

```yaml
type: Mesh
name: default
tracing:
  enabled: true
  type: zipkin
  address: zipkin.srv:9000
```

On Kubernetes:

```yaml
apiVersion: kuma.io/v1alpha1
kind: Mesh
metadata:
  name: default
spec:
  tracing:
    enabled: true
    type: zipkin
    address: zipkin.srv:9000
```

## Traffic Log

With the `TrafficLog` policy you can configure access logging on every Envoy data-plane belonging to the [`Mesh`](#mesh). These logs can then be collected by any agent to be inserted into systems like Splunk, ELK and Datadog.
The first step is to configure backends for the `Mesh`. A backend can be either a file or a TCP service (like Logstash). Second step is to create a `TrafficLog` entity to select connections to log.

On Universal:

```yaml
type: Mesh
name: default
mtls:
  ca:
    builtin: {}
  enabled: true
logging:
  defaultBackend: file
  backends:
    - name: logstash
      format: |
        {
            "destination": "%UPSTREAM_CLUSTER%",
            "destinationAddress": "%UPSTREAM_LOCAL_ADDRESS%",
            "source": "%KUMA_DOWNSTREAM_CLUSTER%",
            "sourceAddress": "%DOWNSTREAM_REMOTE_ADDRESS%",
            "bytesReceived": "%BYTES_RECEIVED%",
            "bytesSent": "%BYTES_SENT%"
        }
      tcp:
        address: 127.0.0.1:5000
    - name: file
      file:
        path: /tmp/access.log
```

```yaml
type: TrafficLog
name: all-traffic
mesh: default
sources:
  - match:
      service: '*'
destinations:
  - match:
      service: '*'
# if omitted, the default logging backend of that mesh will be used
```

```yaml
type: TrafficLog
name: backend-to-database-traffic
mesh: default
sources:
  - match:
      service: backend
destinations:
  - match:
      service: database
conf:
  backend: logstash
```

On Kubernetes:

```yaml
apiVersion: kuma.io/v1alpha1
kind: Mesh
metadata:
  name: default
spec:
  mtls:
    ca:
      builtin: {}
    enabled: true
  logging:
    defaultBackend: file
    backends:
      - name: logstash
        format: |
          {
              "destination": "%UPSTREAM_CLUSTER%",
              "destinationAddress": "%UPSTREAM_LOCAL_ADDRESS%",
              "source": "%KUMA_DOWNSTREAM_CLUSTER%",
              "sourceAddress": "%DOWNSTREAM_REMOTE_ADDRESS%",
              "bytesReceived": "%BYTES_RECEIVED%",
              "bytesSent": "%BYTES_SENT%"
          }
        tcp:
          address: 127.0.0.1:5000
      - name: file
        file:
          path: /tmp/access.log
```

```yaml
apiVersion: kuma.io/v1alpha1
kind: TrafficLog
metadata:
  namespace: kuma-system
  name: all-traffic
spec:
  sources:
    - match:
        service: '*'
  destinations:
    - match:
        service: '*'
  # if omitted, the default logging backend of that mesh will be used
```

```yaml
apiVersion: kuma.io/v1alpha1
kind: TrafficLog
metadata:
  namespace: kuma-system
  name: backend-to-database-traffic
spec:
  sources:
    - match:
        service: backend
  destinations:
    - match:
        service: database
  conf:
    backend: logstash
```

::: tip
If a backend in `TrafficLog` is not explicitly specified, the `defaultBackend` from `Mesh` will be used.
:::

In the `format` field, you can use [standard Envoy placeholders](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log) for TCP as well as a few additional placeholders:

- `%KUMA_SOURCE_ADDRESS%` - source address of the Dataplane
- `%KUMA_SOURCE_SERVICE%` - source service from which traffic is sent
- `%KUMA_DESTINATION_SERVICE%` - destination service to which traffic is sent

## Health Check

The goal of Health Checks is to minimize the number of failed requests due to temporary unavailability of a target endpoint.
By applying a Health Check policy you effectively instruct a dataplane to keep track of health statuses for target endpoints.
Dataplane will never send a request to an endpoint that is considered "unhealthy".

Since pro-active health checking might result in a tangible extra load on your applications,
Kuma also provides a zero-overhead alternative - ["passive" health checking](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/outlier).
In the latter case, a dataplane will be making decisions whether target endpoints are healthy based on "real" requests
initiated by your application rather than auxiliary requests initiated by the dataplanes itself.

As usual, `sources` and `destinations` selectors allow you to fine-tune to which `Dataplanes` the policy applies (`sources`)
and what endpoints require health checking (`destinations`).

At the moment, `HealthCheck` policy is implemented at L4 level. In practice, it means that a dataplane is looking at success of TCP connections
rather than individual HTTP requests.

On Universal:

```yaml
type: HealthCheck
name: web-to-backend
mesh: default
sources:
- match:
    service: web
destinations:
- match:
    service: backend
conf:
  activeChecks:
    interval: 10s
    timeout: 2s
    unhealthyThreshold: 3
    healthyThreshold: 1
  passiveChecks:
    unhealthyThreshold: 3
    penaltyInterval: 5s
```

On Kubernetes:

```yaml
apiVersion: kuma.io/v1alpha1
kind: HealthCheck
metadata:
  namespace: kuma-example
  name: web-to-backend
mesh: default
spec:
  sources:
  - match:
      service: web
  destinations:
  - match:
      service: backend
  conf:
    activeChecks:
      interval: 10s
      timeout: 2s
      unhealthyThreshold: 3
      healthyThreshold: 1
    passiveChecks:
      unhealthyThreshold: 3
      penaltyInterval: 5s
```

## Proxy Template

With the `ProxyTemplate` policy you can configure the low-level Envoy resources directly. The policy requires two elements in its configuration:

- `imports`: this field lets you import canned `ProxyTemplate`s provided by Kuma.
  - In the current release, the only available canned `ProxyTemplate` is `default-proxy`
  - In future releases, more of these will be available and it will also be possible for the user to define them to re-use across their infrastructure
- `resources`: the custom resources that will be applied to every [`Dataplane`]() that matches the `selectors`.

On Universal:

```yaml
type: ProxyTemplate
mesh: default
name: template-1
selectors:
  - match:
      service: backend
conf:
  imports:
    - default-proxy
  resources:
    - ..
    - ..
```

On Kubernetes:

```yaml
apiVersion: kuma.io/v1alpha1
kind: ProxyTemplate
mesh: default
metadata:
  namespace: default
  name: template-1
spec:
  selectors:
    - match:
        service: backend
  conf:
    imports:
      - default-proxy
    resources:
      - ..
      - ..
```

Below you can find an example of what a `ProxyTemplate` configuration could look like:

```yaml
  imports:
    - default-proxy
  resources:
    - name: localhost:9901
      version: v1
      resource: |
        '@type': type.googleapis.com/envoy.api.v2.Cluster
        connectTimeout: 5s
        name: localhost:9901
        loadAssignment:
          clusterName: localhost:9901
          endpoints:
          - lbEndpoints:
            - endpoint:
                address:
                  socketAddress:
                    address: 127.0.0.1
                    portValue: 9901
        type: STATIC
    - name: inbound:0.0.0.0:4040
      version: v1
      resource: |
        '@type': type.googleapis.com/envoy.api.v2.Listener
        name: inbound:0.0.0.0:4040
        address:
          socket_address:
            address: 0.0.0.0
            port_value: 4040
        filter_chains:
        - filters:
          - name: envoy.http_connection_manager
            config:
              route_config:
                virtual_hosts:
                - routes:
                  - match:
                      prefix: /stats/prometheus
                    route:
                      cluster: localhost:9901
                  domains:
                  - '*'
                  name: envoy_admin
              codec_type: AUTO
              http_filters:
                name: envoy.router
              stat_prefix: stats
```