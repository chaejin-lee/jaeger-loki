This is the repository that contains local loki object storage plugin for Jaeger.

You are free to use this software under a permissive open-source MIT license.

To fund further work and maintenance on this plugin, work will be done by flitnetics.

If you require additional support for your infrastructure, you can contact [Sales](mailto:sales@flitnetics.com)

## About

Local Grafana Loki object storage support for Jaeger.

The configuration is mostly identical on how you configure storage, indexes, compactors, rulers, etc in Loki.

With this plugin, you won't need to run and maintain Tempo, at all!


**Version 2 of this plugin is not compatible with Version 1**

## Build/Compile
In order to compile the plugin from source code you can use `go build`:

```
cd /path/to/jaeger-objectstorage
go build ./cmd/jaeger-objectstorage
```

## Configuration
#### Storage
[https://github.com/grafana/loki/blob/37a7189d4ed76655144d982e2eeebf495e0809ea/docs/sources/configuration/_index.md#storage_config](https://github.com/grafana/loki/blob/37a7189d4ed76655144d982e2eeebf495e0809ea/docs/sources/configuration/_index.md#storage_config)
#### Index (schema config)
[https://github.com/grafana/loki/blob/37a7189d4ed76655144d982e2eeebf495e0809ea/docs/sources/configuration/_index.md#schema_config](https://github.com/grafana/loki/blob/37a7189d4ed76655144d982e2eeebf495e0809ea/docs/sources/configuration/_index.md#schema_config)
#### More info
[https://grafana.com/docs/loki/latest/operations/storage/boltdb-shipper/](https://grafana.com/docs/loki/latest/operations/storage/boltdb-shipper/)

*  Can share with Grafana Loki configuration

## Start
In order to start plugin just tell jaeger the path to a config compiled plugin.

```
GRPC_STORAGE_PLUGIN_BINARY="./jaeger-objectstorage" GRPC_STORAGE_PLUGIN_CONFIGURATION_FILE=./config-example.yaml SPAN_STORAGE_TYPE=grpc-plugin  GRPC_STORAGE_PLUGIN_LOG_LEVEL=DEBUG ./all-in-one --sampling.strategies-file=/location/of/your/jaeger/cmd/all-in-one/sampling_strategies.json
```

For Jaeger Operator on Kubernetes for testing/demo **!!NOT PRODUCTION!!**, sample manifest:

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: observability

commonLabels:
  app.kubernetes.io/instance: observability

resources:
  - https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/crds/jaegertracing.io_jaegers_crd.yaml
  - https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/service_account.yaml
  - https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/role.yaml
  - https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/role_binding.yaml
  - https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/operator.yaml
  - https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/cluster_role.yaml
  - https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/cluster_role_binding.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger-operator
  app: jaeger
spec:
  template:
    spec:
      containers:
      - name: jaeger-operator
        image: jaegertracing/jaeger-operator:master
        args: ["start"]
        env:
        - name: LOG-LEVEL
          value: debug
---
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger-objectstorage
spec:
  strategy: allInOne
  allInOne:
    image: jaegertracing/all-in-one:latest
    options:
      log-level: debug
  storage:
    type: grpc-plugin
    grpcPlugin:
      image: tmaxcloudck/jaeger-loki-plugin:v2.0.1
    options:
      grpc-storage-plugin:
        binary: /plugin/jaeger-objectstorage
        configuration-file: /plugin-config/config-example.yaml
        log-level: debug
  volumeMounts:
    - name: config-volume
      mountPath: /plugin-config
  volumes:
    - name: config-volume
      configMap:
        name: jaeger-objectstorage-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: jaeger-objectstorage-config
data:
  config-example.yaml: |-
  
    auth_enabled: false
    ingester:
      chunk_idle_period: 5m
      max_chunk_age: 1h
      chunk_retain_period: 30s
      max_transfer_retries: 0
      wal:
        enabled: false
      lifecycler:
        address: 0.0.0.0
        ring:
          replication_factor: 1
          kvstore:
            store: inmemory
        final_sleep: 0s
        
    querier:
      max_concurrent: 2048
      query_ingesters_within: 0
      query_timeout: 5m
      
    query_scheduler:
      max_outstanding_requests_per_tenant: 2048
      
    limits_config:
      retention_period: 168h # 7d
      #retention_stream:
      #- selector: '{namespace=""}' or '{pod_name=""}'
      #  priority: 1
      #  period: 24h ## 최소 설정 가능 기간 24h
      enforce_metric_name: false
      #reject_old_samples: true
      #reject_old_samples_max_age: 72h
      ingestion_rate_mb: 16
      ingestion_burst_size_mb: 32
      max_query_series: 100000
      per_stream_rate_limit: 512mb
      per_stream_rate_limit_burst: 1024mb
      
    schema_config:
      configs:
      - from: 2022-07-10
        store: boltdb-shipper
        object_store: filesystem
        schema: v11
        index:
          prefix: index_
          period: 24h
    server:
      http_listen_port: 3100
      http_server_read_timeout: 5m
      http_server_write_timeout: 5m
     # log_level:

    storage_config:
      boltdb_shipper:
        active_index_directory: /loki/index
        cache_location: /loki/index_cache
        cache_ttl: 24h         # Can be increased for faster performance over longer query periods, uses more disk space
        shared_store: filesystem
      filesystem:
        directory: /loki/chunks
    
    compactor:
      retention_enabled: true
      retention_delete_delay: 30m
      working_directory: /loki/compactor
      shared_store: filesystem
      
```

## License

The S3 Storage gRPC Plugin for Jaeger is an [MIT licensed](LICENSE) open source project.
