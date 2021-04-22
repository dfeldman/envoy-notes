
# Envoy+SPIFFE Configuration "Blocks"
This file is designed to have reusable blocks of Envoy configuration YAML that can be easily copied and pasted. 

Templated variables start with TEMPLATE_. This can easily be replaced manually, with sed, or swapped with Go template variables (for use with Helm) or Jinja2 template variables (for use with Python). 

If you use sed, be aware that SPIFFE IDs contain /, which will confuse sed when replacing.

# Versions
In version 1.17, Envoy upgraded from "V2" config format to "V3." The formats are subtly different and switching between them takes forever. There is a backward compatibility mode available, but it is buggy so I don't recommend using it. I've included config blocks for both formats here (although not all are in both formats). 

As of this writing, a lot of the easily-Googleable Envoy documentation is for the V2 format, and hasn't yet been updated for V3. 

# Note about SPIFFE Certificate Validator
Envoy is working on a "SPIFFE Certificate Validator" which will require different config options. It is not ready yet so I have not used it in this document. Eventually, it will be ready, and we should switch to it because it will support federated trust (which Envoy does not natively support). 

# Overall format
You'll want these parts to your config file:
 * Header
 * Admin port (optional but recommended)
 * Listeners (at least 1)
 * Clusters (at least 2: one for data, one for SPIFFE itself)

I recommend always running your config through a YAML validator before even trying it with Envoy. 

# Variables
 
| Variable | Description | 
--- | --- 
TEMPLATE_LISTEN_ADDRESS | Address to listen on. 0.0.0.0 for all local interfaces (most common choice). 
TEMPLATE_LISTEN_PORT | Port to listen on
TEMPLATE_CLIENT_SPIFFE_ID | Acceptable SPIFFE ID for clients connecting to the proxy
TEMPLATE_SERVER_SPIFFE_ID | SPIFFE ID to use on the server side, both for listening for connections and initiating connections upstream. 
TEMPLATE_TRUST_DOMAIN | Trust domain to use WITH the leading spiffe:// (eg spiffe://test.com). All incoming certs will be validated against this bundle.
TEMPLATE_UPSTREAM_SPIFFE_ID | SPIFFE ID to expect of the upstream server Envoy is connecting to (if TLS)
TEMPLATE_UPSTREAM_ADDRESS | Address to connect to upstream
TEMPLATE_UPSTREAM_PORT | Port to connect to upstream
TEMPLATE_ACCESS_LOG_PATH | Path to a file to log accesses (typically /var/log/envoy.log)
TEMPLATE_ENVOY_ID | ID of this envoy instance (eg "id_01"), *must be unique per host*
TEMPLATE_ENVOY_CLUSTER | Cluster of this envoy instance (eg "cluster_01")
TEMPLATE_WORKLOAD_API_SOCKET | The socket to use to talk to the SPIFFE Workload API (typically /run/spire/agent.sock on Kubernetes, or /var/run/spire/agent.sock outside Kubernetes)

# Deciding on HTTP vs TCP listeners
Non-HTTP TCP traffic should just always use a TCP listener. 

For HTTP, there's a decision to make. The HTTP listeners are generally superior since you'll get more logging data, and ability to reroute traffic based on URL, and more informative logging. However, there's additional complexity and more failure modes. 

For simplicity, I've always called the cluster for the workload API "spire-agent," and the cluster for the actual data "upstream."

# Header
Be sure to always change the TEMPLATE_ENVOY_ID so each instance on the same machine has a different ID. 
```
node:
  id: "TEMPLATE_ENVOY_ID"
  cluster: "TEMPLATE_ENVOY_CLUSTER"
```

# Admin port
For debugging/testing, I recommend always enabling the admin interface on port 9901.
```
admin:
  access_log_path: /var/log/envoy_admin.log
  address:
    socket_address: { address: 0.0.0.0, port_value: 9901 }
```
Connecting to this port will show an extremely useful admin interface that lets you see what Envoy is actually doing.

This should not be exposed in production.


# Listeners

Start the listener section with:
```
static_resources:
  listeners:
```

## Non-TLS HTTP listener (V2)
```
- name: envoy.http_connection_manager
  typed_config:
    "@type": type.googleapis.com/envoy.config.filter.network.http_connection_manager.v2.HttpConnectionManager
    common_http_protocol_options:
      idle_timeout: 1s
    codec_type: auto
    access_log:
    - name: envoy.file_access_log
      config:
        path: TEMPLATE_ACCESS_LOG_PATH
    stat_prefix: ingress_http
    route_config:
      name: service_route
      virtual_hosts:
      - name: outbound_proxy
        domains: ["*"]
        routes:
        - match:
            prefix: "/"
          route:
            cluster: upstream
    http_filters:
    - name: envoy.router
```

## TLS HTTP listener (V2)
```
- name: envoy.http_connection_manager
  typed_config:
    "@type": type.googleapis.com/envoy.config.filter.network.http_connection_manager.v2.HttpConnectionManager
    common_http_protocol_options:
      idle_timeout: 1s
    forward_client_cert_details: sanitize_set
    set_current_client_cert_details:
      uri: true
    codec_type: auto
    access_log:
    - name: envoy.file_access_log
      config:
        path: TEMPLATE_ACCESS_LOG_PATH
    stat_prefix: ingress_http
    route_config:
      name: local_route
      virtual_hosts:
      - name: local_service
        domains: ["*"]
        routes:
        - match:
            prefix: "/"
          route:
            cluster: upstream
    http_filters:
    - name: envoy.router
tls_context:
  common_tls_context:
    require_client_certificate: true
    tls_certificate_sds_secret_configs:
    - name: TEMPLATE_SERVER_SPIFFE_ID
      sds_config:
        api_config_source:
          api_type: GRPC
          grpc_services:
            envoy_grpc:
              cluster_name: spire_agent
    combined_validation_context:
      default_validation_context:
        match_subject_alt_names:
        - exact: TEMPLATE_CLIENT_SPIFFE_ID
        - exact: TEMPLATE_SERVER_SPIFFE_ID
      validation_context_sds_secret_config:
        name: TEMPLATE_TRUST_DOMaIN
        sds_config:
          api_config_source:
            api_type: GRPC
            grpc_services:
              envoy_grpc:
                cluster_name: spire_agent
    tls_params:
      ecdh_curves:
      - X25519:P-256:P-521:P-384
```

## Non-TLS TCP listener (V3)
Note that this config has no transport_socket. That is intentional, the default is raw TCP. 

```
  - name: tcp-listener
    address:
      socket_address:
        address: TEMPLATE_LISTEN_ADDRESS
        port_value: TEMPLATE_LISTEN_PORT
    filter_chains:
      filters:
      - name: envoy.filters.network.tcp_proxy
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy
          stat_prefix: tcp
          cluster: upstream
          access_log:
          - name: file
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.access_loggers.file.v3.FileAccessLog
              path: TEMPLATE_ACCESS_LOG_PATH
```

## TLS+TCP listener with SPIFFE SDS server (V3)

```
  - name: mtls-listener
    address:
      socket_address:
        address: TEMPLATE_LISTEN_ADDRESS
        port_value: TEMPLATE_LISTEN_PORT
    filter_chains:
    - transport_socket:
        name: envoy.transport_sockets.tls
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.DownstreamTlsContext
          common_tls_context:
            tls_certificate_sds_secret_configs:
            - name: TEMPLATE_SERVER_SPIFFE_ID
              sds_config:
                resource_api_version: V3
                api_config_source:
                  api_type: GRPC
                  transport_api_version: V3
                  grpc_services:
                    envoy_grpc:
                      cluster_name: spire_agent
            combined_validation_context:
              default_validation_context:
                match_subject_alt_names:
                  - exact: TEMPLATE_SERVER_SPIFFE_ID # for testing
                  - exact: TEMPLATE_CLIENT_SPIFFE_ID
                  # Add more SPIFFE IDs here if necessary
              validation_context_sds_secret_config:
                name: TEMPLATE_TRUST_DOMAIN
                sds_config:
                  resource_api_version: V3
                  api_config_source:
                    api_type: GRPC
                    transport_api_version: V3
                    grpc_services:
                      envoy_grpc:
                        cluster_name: spire_agent # Must be specified in "clusters"
      filters:
      - name: envoy.filters.network.tcp_proxy
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy
          stat_prefix: tcp
          cluster: upstream
          access_log:
          - name: file
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.access_loggers.file.v3.FileAccessLog
              path: TEMPLATE_ACCESS_LOG_PATH
```

# Clusters

## SPIFFE SDS Cluster (V2)
```
  clusters:
  - name: spire_agent
    connect_timeout: 1s
    http2_protocol_options: {}
    hosts:
    - pipe:
        path: TEMPLATE_WORKLOAD_API_SOCKET
```

## SPIFFE SDS Cluster (V3)

```
  clusters:
  - name: spire_agent
    connect_timeout: 1s
    type: STATIC
    lb_policy: ROUND_ROBIN
    typed_extension_protocol_options:
      envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
        "@type": type.googleapis.com/envoy.extensions.upstreams.http.HttpProtocolOptions
        explicit_http_config:
          http2_protocol_options: {}
    load_assignment:
      cluster_name: spire_agent
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              pipe:
                path: TEMPLATE_WORKLOAD_API_SOCKET
```

## TLS Upstream (V2)

This should work for either HTTP or TCP traffic.

```
  - name: upstream
    connect_timeout: 0.25s
    type: strict_dns
    lb_policy: ROUND_ROBIN
    hosts:
    - socket_address:
        address: TEMPLATE_UPSTREAM_ADDRESS
        port_value: TEMPLATE_UPSTREAM_PORT
    tls_context:
      common_tls_context:
        tls_certificate_sds_secret_configs:
        # This is the most common choice. You could imagine in some cases needing
        # to listen with one SPIFFE ID, and then connect upstream with a different
        # SPIFFE ID, but this would be unusual. 
        - name: TEMPLATE_SERVER_SPIFFE_ID 
          sds_config:
            api_config_source:
              api_type: GRPC
              grpc_services:
                envoy_grpc:
                  cluster_name: spire_agent
    combined_validation_context:
      # validate the SPIFFE ID of the server (recommended)
      validation_context:
        match_subject_alt_names:
          exact: TEMPLATE_UPSTREAM_SPIFFE_ID
      validation_context_sds_secret_config:
        name: spiffe_validation_context
        sds_config:
          api_config_source:
            api_type: GRPC
            grpc_services:
              envoy_grpc:
                cluster_name: spire_agent
    tls_params:
      ecdh_curves:
      - X25519:P-256:P-521:P-384
```

## TLS Upstream (V3)
```
  clusters:
  - name: upstream
    connect_timeout: 1s
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: service-https
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: TEMPLATE_UPSTREAM_ADDRESS
                port_value: TEMPLATE_UPSTREAM_PORT
    transport_socket:
      name: envoy.transport_sockets.tls
      typed_config:
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
        common_tls_context:
          tls_certificate_sds_secret_configs:
          - name: {{ .Values.frontendSpiffeId }}
            sds_config:
              resource_api_version: V3
              api_config_source:
                api_type: GRPC
                transport_api_version: V3
                grpc_services:
                  envoy_grpc:
                    cluster_name: spire_agent
          combined_validation_context:
            default_validation_context:
              match_subject_alt_names:
                - exact: {{ .Values.backendSpiffeId }}
                # Add more SPIFFE IDs here if necessary
            validation_context_sds_secret_config:
              name: {{ .Values.trustDomain }}
              sds_config:
                resource_api_version: V3
                api_config_source:
                  api_type: GRPC
                  transport_api_version: V3
                  grpc_services:
                    envoy_grpc:
                      cluster_name: spire_agent # Must be specified in "clusters"
```


## Non-TLS Upstream (V2)
```
  - name: upstream
    connect_timeout: 1s
    type: strict_dns
    hosts:
      - socket_address:
          address: TEMPLATE_UPSTREAM_HOST
          port_value: TEMPLATE_UPSTREAM_PORT
```


## Non-TLS Upstream (V3)
```
  - name: upstream
    connect_timeout: 1s
    type: strict_dns
    load_assignment:
      cluster_name: local_service
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: 127.0.0.1
                port_value: TEMPLATE_UPSTREAM_PORT
```

## Using it in a Helm chart ConfigMap
ConfigMap:
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ template "envoy.fullname" . }}
  labels:
    app: {{ template "envoy.name" . }}
    chart: {{ template "envoy.chart" . }}
    release: {{ .Release.Name }}
    heritage: {{ .Release.Service }}
data:
  envoy.yaml: |
```
Then paste the envoy config in, indented 4 spaces. 

