---
title: Kubernetes Ingress Controller
sidebar_label: Ingress Controller
lang: en-US
keywords: [pomerium, identity access proxy, oidc, kubernetes, ingress, ingress controller, reverse proxy]
pagination_next: null
---

Use Pomerium as a first-class secure-by-default Ingress Controller. The Pomerium Ingress Controller enables workflows more native to Kubernetes environments, such as Git-Ops style actions based on pull requests. Dynamically provision routes from Ingress resources and set policy based on annotations. By defining routes as Ingress resources you can independently create and remove them from Pomerium's configuration.

## Prerequisites

- A certificate management solution. If you do not already have one in place, this article covers using [cert-manager](https://cert-manager.io/).
- A [Redis](https://redis.io/) backend with high [persistence](https://redis.io/topics/persistence) is highly recommended.

### System Requirements

- Kubernetes v1.19.0+
- Pomerium [Helm Chart](https://github.com/pomerium/pomerium-helm/tree/main/charts/pomerium) v25.0.0+

### Limitations

:::caution

Only one Ingress Controller instance is supported per Pomerium cluster.

:::

## Installation

### Helm

Our instructions for [Installing Pomerium Using Helm](/docs/k8s/helm) includes the Ingress Controller as part of the documented configuration. You can confirm by looking for this line in `pomerium-values.yaml`:


```yaml
ingressController:
  enabled: true
```

### Docker Image

You may deploy the Ingress controller from your own manifests by using the `pomerium/ingress-controller` docker image.

## Configuration

| Flag                           | Description                                                             |
| ------------------------------ | ----------------------------------------------------------------------- |
| `--databroker-service-url`     | The databroker service url                                              |
| `--databroker-tls-ca`          | `base64` encoded TLS CA                                                 |
| `--databroker-tls-ca-file`     | TLS CA file path for the databroker connection connection               |
| `--health-probe-bind-address`  | The address the probe endpoint binds to. (default ":8081")              |
| `--metrics-bind-address`       | The address the metric endpoint binds to. (default ":8080")             |
| `--name`                       | IngressClass controller name (default "pomerium.io/ingress-controller") |
| `--namespaces`                 | Namespaces to watch, omit to watch all namespaces                       |
| `--prefix`                     | Ingress annotation prefix (default "ingress.pomerium.io")               |
| `--shared-secret`              | `base64` encoded shared secret for communicating with databroker        |
| `--update-status-from-service` | Update ingress status from given service status (pomerium-proxy)        |

The helm chart exposes a subset of these flags for appropriate customization.

## Usage

### Defining Routes

If you've tested Pomerium using the [all-in-one binary](/docs/install/binary), you're probably familiar with configuring routes in Pomerium's [`config.yaml`](/docs/install/binary#configuration-file). When using the Pomerium Ingress Controller, each route is defined as an Ingress resource in the Kubernetes API.

The Ingress Controller will monitor Ingress resources in the cluster, creating a Pomerium route definition for each one. Policy and other configuration options for the route are set by using annotations starting with `ingress.pomerium.io/`.

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    ingress.pomerium.io/policy: '[{"allow":{"and":[{"email":{"is":"user@yourdomain.com"}}]}}]' # This can also be a yaml block quote
spec:
  rules:
  - host: hello.localhost.pomerium.io
    http:
      paths:
      - backend:
          service:
            name: nginx-hello
            port:
              name: http
        path: /
        pathType: Prefix
```

Becomes:

```yaml
routes:
  - from: https://hello.localhost.pomerium.io
    to: http://nginx-hello.default.svc.cluster.local
    policy:
    - allow:
        and:
          - email:
              is: user@yourdomain.com
```

<details>
  <summary>Write Policies in YAML</summary>
  <div>
  You can also define a route's policies using YAML:

  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: name
    annotations:
      ingress.pomerium.io/policy: |
        - allow:
            or:
              - domain:
                  is: pomerium.com
  ```

  </div>
</details>

:::tip
Routes are sorted and applied in the following order.

1. Ascending by `from`.
1. Descending by `path`.
1. Descending by `regex`.
1. Descending by `prefix`.
1. Ascending by `id`.

This sorting order helps ensure that more restrictive routes for specific paths and regexes are applied correctly.
:::

### Supported Annotations

Most configuration keys in non-Kubernetes deployments can be specified as annotation in an Ingress Resource definition. The format is `ingress.pomerium.io/${OPTION_NAME}`. The expandable list below contains the annotations available, which behave as described in our reference documentation (with links to the appropriate reference documentation).

<details>
<summary>Pomerium-Standard Annotations</summary>
<div>

- [`ingress.pomerium.io/allow_any_authenticated_user`]
- [`ingress.pomerium.io/allow_public_unauthenticated_access`]
- [`ingress.pomerium.io/allow_spdy`]
- [`ingress.pomerium.io/allow_websockets`]
- [`ingress.pomerium.io/allowed_domains`]
- [`ingress.pomerium.io/allowed_groups`]
- [`ingress.pomerium.io/allowed_idp_claims`]
- [`ingress.pomerium.io/allowed_users`]
- [`ingress.pomerium.io/cors_allow_preflight`]
- [`ingress.pomerium.io/host_path_regex_rewrite_pattern`]
- [`ingress.pomerium.io/host_path_regex_rewrite_substitution`]
- [`ingress.pomerium.io/host_rewrite`]
- [`ingress.pomerium.io/host_rewrite_header`]
- [`ingress.pomerium.io/idle_timeout`]
- [`ingress.pomerium.io/outlier_detection`]
- [`ingress.pomerium.io/pass_identity_headers`]
- [`ingress.pomerium.io/policy`]
- [`ingress.pomerium.io/prefix_rewrite`]
- [`ingress.pomerium.io/preserve_host_header`]
- [`ingress.pomerium.io/regex_rewrite_pattern`]
- [`ingress.pomerium.io/regex_rewrite_substitution`]
- [`ingress.pomerium.io/remove_request_headers`]
- [`ingress.pomerium.io/rewrite_response_headers`]
- [`ingress.pomerium.io/set_request_headers`]
- [`ingress.pomerium.io/set_response_headers`]
- [`ingress.pomerium.io/timeout`]
- [`ingress.pomerium.io/tls_server_name`]
- [`ingress.pomerium.io/tls_skip_verify`]

</div>
</details>

The remaining annotations are specific to or behave differently than they do when using Pomerium without the Ingress Controller:

| Annotation                                             | Description                                                                                                                                                                                   |
| ------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ingress.pomerium.io/kubernetes_service_account_token` | Name of a Kubernetes `token` Secret containing a [Kubernetes Service Account Token](/docs/reference/routes/kubernetes-service-account-token).                                                   |
| `ingress.pomerium.io/path_regex`                       | When set to `"true"` enables path regex matching. See the [Regular Expressions Path Matching](#regular-expressions-path-matching) section for more information.                               |
| `ingress.pomerium.io/secure_upstream`                  | When set to `"true"`, use `https` when connecting to the upstream endpoint.                                                                                                                   |
| `ingress.pomerium.io/set_request_headers_secret`       | Name of Kubernetes Secret containing the contents of the request header to send upstream. When used, `ingress.pomerium.io/set_request_headers` should not contain overlapping keys.           |
| `ingress.pomerium.io/set_response_headers_secret`      | Name of Kubernetes Secret containing the contents of the response header to send downstream. When used, `ingress.pomerium.io/set_response_headers` should not contain overlapping keys.       |
| `ingress.pomerium.io/service_proxy_upstream`           | When set to `"true"` forces Pomerium to connect to upstreams through the k8s service proxy, and not individual endpoints. <br/> This is useful when deploying Pomerium inside a service mesh. |
| `ingress.pomerium.io/tcp_upstream`                     | When set to `"true"`, defines the route as supporting a TCP tunnel. See the [example below](#tcp-endpoints) for more information.                                                             |
| `ingress.pomerium.io/tls_client_secret`                | Name of Kubernetes `tls` Secret containing a [client certificate][tls_client_certificate] for connecting to the upstream.                                                                     |
| `ingress.pomerium.io/tls_custom_ca_secret`             | Name of Kubernetes `tls` Secret containing a custom [CA certificate][`tls_custom_ca_secret`] for the upstream.                                                                                |
| `ingress.pomerium.io/tls_downstream_client_ca_secret`  | Name of Kubernetes `tls` Secret containing a [Client CA][client-certificate-authority] for validating downstream clients.                                                                     |

:::tip

Every value for the annotations above must be in `string` format.

:::

### cert-manager Integration

Pomerium Ingress Controller can use [cert-manager](https://cert-manager.io/) to automatically provision certificates. These may come from the [ingress-shim](https://cert-manager.io/docs/usage/ingress/) or explicitly configured [`Certificate` resources](https://cert-manager.io/docs/usage/certificate/).

To use [HTTP01 Challenges](https://cert-manager.io/docs/configuration/acme/http01/) with your [Issuer](https://cert-manager.io/docs/concepts/issuer/), configure the solver class to match the Ingress Controller. The Ingress Controller will automatically configure policy to facilitate the HTTP01 challenge:

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: example-issuer
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: example-issuer-account-key
    solvers:
    - http01:
       ingress:
         class: pomerium
```

An example of using the [ingress-shim](https://cert-manager.io/docs/usage/ingress/) with an Ingress resource managed by Pomerium:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/issuer: example-issuer
    ingress.pomerium.io/policy: '[{"allow":{"and":[{"email":{"is":"user@exampledomain.com"}}]}}]'
  name: example
spec:
  ingressClassName: pomerium
  rules:
  - host: example.localhost.pomerium.io
    http:
      paths:
      - backend:
          service:
            name: example
            port:
              name: http
        path: /
        pathType: Prefix
  tls:
  - hosts:
    - example.localhost.pomerium.io
    secretName: example-tls
```

## HTTPS Endpoints

The `Ingress` spec assumes that all communications to the upstream service is sent in plaintext. For more information, see the [TLS](https://kubernetes.io/docs/concepts/services-networking/ingress/#tls) section of the Ingress API documentation. Pomerium supports HTTPS communication with upstream endpoints, including mTLS.

Annotate your `Ingress` with

```yaml
ingress.pomerium.io/secure_upstream: true
```

Additional TLS certificates may be supplied by creating a Kubernetes secret(s) in the same namespaces as the `Ingress` resource. Please note that we do not support file paths or embedded secret references.

- [`ingress.pomerium.io/tls_client_secret`](/docs/reference/routes/tls-client-certificate)
- [`ingress.pomerium.io/tls_custom_ca_secret`](/docs/reference/routes/tls-custom-certificate-authority)
- [`ingress.pomerium.io/tls_downstream_client_ca_secret`](#supported-annotations)

Please note that the referenced `tls_client_secret` must be a [TLS Kubernetes secret](https://kubernetes.io/docs/concepts/configuration/secret/#tls-secrets). `tls_custom_ca_secret` and `tls_downstream_client_ca_secret` must contain `ca.crt` containing a .PEM encoded (base64-encoded DER format) public certificate.

### External Services

You may refer to external services by defining a [Service](https://kubernetes.io/docs/concepts/services-networking/service/) with `externalName`.

I.e. if you have `https://my-existing-service.corp.com`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external
spec:
  type: ExternalName
  externalName: "my-existing-service.corp.com"
  ports:
    - protocol: TCP
      name: https
      port: 443
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: external
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod-http
    ingress.pomerium.io/secure_upstream: "true"
    ingress.pomerium.io/policy: |
      - allow:
          and:
            - domain:
                is: pomerium.com
spec:
  ingressClassName: pomerium
  tls:
    - hosts:
        - "external.localhost.pomerium.io"
      secretName: external-localhost-pomerium.io
  rules:
    - host: "external.localhost.pomerium.io"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: external
                port:
                  name: https
```

### Regular Expressions Path Matching

You can use a [re2 regular expression] To create an Ingress that matches multiple paths.

1. Set the `path_regex` annotation to `"true"`
1. Set `pathType` to `ImplementationSpecific`
1. Set `path` to an re2 expression matching the full path. It must include the `^/` prefix and `$` suffix. Any query strings should be removed.

:::tip
Check out [this example expression](https://regex101.com/r/IBVUKT/1/) at [regex101.com] for a more detailed explanation and example paths, both matching and not.
:::

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/issuer: example-issuer
    ingress.pomerium.io/allowed_domains: '["exampledomain.com"]'
    ingress.pomerium.io/path_regex: "true"
  name: example
spec:
  ingressClassName: pomerium
  rules:
  - host: example.localhost.pomerium.io
    http:
      paths:
      - backend:
          service:
            name: example
            port:
              name: http
        path: ^/(admin|superuser)/.*$
        pathType: ImplementationSpecific
  tls:
  - hosts:
    - example.localhost.pomerium.io
    secretName: example-tls
```

## TCP Endpoints

The example route below defines a route providing a [tunneled TCP connection](/docs/tcp/) to an upstream service listening for non-web traffic:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tcp-example
  annotations:
    ingress.pomerium.io/tcp_upstream: "true"
spec:
  ingressClassName: pomerium
  rules:
    - host: "tcp.localhost.pomerium.io"
      http:
        paths:
          - pathType: ImplementationSpecific
            backend:
              service:
                name: tcp-service
                port:
                  name: app
```

The important points to note in this example:

- The annotation `ingress.pomerium.io/tcp_upstream:` is set to `"true"`,
- `spec.rules.[].http.paths.[].path` is omitted,
- `spec.rules.[].http.paths.[].pathType` is set to `ImplementationSpecific`,
- `spec.rules.[].host` and `spec.rules.[].paths.[].backend.service.port.name/number` together define the address used when connecting to the route using the [Pomerium Desktop or CLI clients](/docs/tcp/client),
- You may apply standard access control annotations to define access restrictions to your port.

Unlike a standalone Pomerium configuration, you may not create multiple TCP routes using the same hostname with different ports. This limitation was made to avoid confusion, and because additional configuration parameters, such as the Ingress resource, do not allow passing port numbers in the `spec.rules.host` parameter.

## Troubleshooting

### View Event History

Pomerium Ingress Controller will add **events** to the Ingress objects as it processes them.

```bash
kubectl describe ingress/my-ingress
```

```log
Events:
  Type    Reason   Age   From              Message
  ----    ------   ----  ----              -------
  Normal  Updated  18s   pomerium-ingress  updated pomerium configuration
```

If an error occurs, it may be reflected in the events:

```log
Events:
  Type     Reason       Age                 From              Message
  ----     ------       ----                ----              -------
  Normal   Updated      5m53s               pomerium-ingress  updated pomerium configuration
  Warning  UpdateError  3s                  pomerium-ingress  upsert routes: parsing ingress: annotations: applying policy annotations: parsing policy: invalid rules in policy: unsupported conditional "maybe", only and, or, not, nor and action are allowed
```

### HSTS

If your domain has [HSTS] enabled and you visit an endpoint while Pomerium is using the self-signed bootstrap certificate or a LetsEncrypt staging certificate (before cert-manager has provisioned a production certificate), the untrusted certificate may be pinned in your browser and would need to be reset. See [this article](https://www.ssl2buy.com/wiki/how-to-clear-hsts-settings-on-chrome-firefox-and-ie-browsers) for more information.

## More Information

For more information on the Pomerium Ingress Controller or the Kubernetes concepts discussed, see:

- [Ingress (Kubernetes Docs)](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Pomerium Helm Chart README: Pomerium Ingress Controller](https://github.com/pomerium/pomerium-helm/tree/main/charts/pomerium#pomerium-ingress-controller)
- [Pomerium Kubernetes Ingress Controller (code repository)](https://github.com/pomerium/ingress-controller)

[`ingress.pomerium.io/allow_any_authenticated_user`]: /docs/reference/routes/allow-any-authenticated-user
[`ingress.pomerium.io/allow_public_unauthenticated_access`]: /docs/reference/routes/public-access
[`ingress.pomerium.io/allow_spdy`]: /docs/reference/routes/spdy
[`ingress.pomerium.io/allow_websockets`]: /docs/reference/routes/websocket-connections
[`ingress.pomerium.io/allowed_domains`]: /docs/reference/policy/allowed-domains
[`ingress.pomerium.io/allowed_groups`]: /docs/reference/policy/allowed-groups
[`ingress.pomerium.io/allowed_idp_claims`]: /docs/reference/policy/allowed-idp-claims
[`ingress.pomerium.io/allowed_users`]: /docs/reference/policy/allowed-users
[`ingress.pomerium.io/cors_allow_preflight`]: /docs/reference/routes/cors-preflight
[`ingress.pomerium.io/health_checks`]: /docs/reference/health-checks
[`ingress.pomerium.io/host_path_regex_rewrite_pattern`]: /docs/reference/routes/host-rewrite
[`ingress.pomerium.io/host_path_regex_rewrite_substitution`]: /docs/reference/routes/host-rewrite
[`ingress.pomerium.io/host_rewrite_header`]: /docs/reference/routes/host-rewrite
[`ingress.pomerium.io/host_rewrite`]: /docs/reference/routes/host-rewrite
[`ingress.pomerium.io/idle_timeout`]: /docs/reference/routes/idle-timeout
[`ingress.pomerium.io/lb_config`]: /docs/reference/load-balancing-policy-config
[`ingress.pomerium.io/outlier_detection`]: /docs/reference/routes/outlier-detection
[`ingress.pomerium.io/pass_identity_headers`]: /docs/reference/routes/pass-identity-headers
[`ingress.pomerium.io/policy`]: /docs/reference/policy/policy
[`ingress.pomerium.io/prefix_rewrite`]: /docs/reference/routes/prefix-rewrite
[`ingress.pomerium.io/preserve_host_header`]: /docs/reference/routes/host-rewrite
[`ingress.pomerium.io/regex_rewrite_pattern`]: /docs/reference/routes/regex-rewrite
[`ingress.pomerium.io/regex_rewrite_substitution`]: /docs/reference/routes/regex-rewrite
[`ingress.pomerium.io/remove_request_headers`]: /docs/reference/routes/remove-request-headers
[`ingress.pomerium.io/rewrite_response_headers`]: /docs/reference/routes/rewrite-response-headers
[`ingress.pomerium.io/set_request_headers`]: /docs/reference/routes/set-request-headers
[`ingress.pomerium.io/set_response_headers`]: /docs/reference/set-response-headers
[`ingress.pomerium.io/timeout`]: /docs/reference/routes/route-timeout
[`ingress.pomerium.io/tls_server_name`]: /docs/reference/routes/tls-server-name
[`ingress.pomerium.io/tls_skip_verify`]: /docs/reference/routes/tls-skip-verification
[`tls_custom_ca_secret`]: /docs/reference/routes/tls-custom-certificate-authority
[client-certificate-authority]: /docs/reference/client-certificate-authority
[HSTS]: https://en.wikipedia.org/wiki/HTTP_Strict_Transport_Security
[re2 regular expression]: https://github.com/google/re2/wiki/Syntax
[regex101.com]: https://regex101.com
[tls_client_certificate]: /docs/reference/routes/tls-client-certificate