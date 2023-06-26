# !!!WIP!!!


# Setting Sail with Flyte: A Beginners Guide to Dev Environment Setup

If you're reading this, you likely have an interest in Flyte, an open-source data, ML, and infrastructure orchestrator that simplifies the creation, scaling, and maintenance of concurrent workflows. Like many in the open-source community, you might be eager to contribute but find the initial setup process a daunting hurdle.

This experience is quite common for newcomers—I too struggled with the setup when I first started contributing (many thanks to Kevin Su for all his help!). Building a strong Flyte community, where everyone feels welcome to participate, requires us to break down these barriers and make the contribution process more accessible. With this goal in mind, I've written this guide as a supplement to the official resources. It aims to provide a clear and simple, step-by-step approach to setting up a local development environment, reflecting the latest updates and modifications. It especially simplifies the setup for features that involve multiple components.

So, let's embark on this journey together, easing your transition into the world of Flyte contributions.



## Prequest

- This guide is used and tested on AWS EC2 with ubuntu 22.04 image.
- Docker
- Kubectl

## Content
- How to set dev environment for flyteidl, flyteadmin, flyteplugins, flytepropeller?
 
- How to set dev environment for flytekit?
  - Run workflow locally
  - Run workflow in sandbox
  - Develop flytekit plugins, using Deck as an example
  
- How to set dev environment for flyteconsole?
  
- Putting everyhing together, How to set dev environment if your PR involve multiple components?
  
- FAQ

## How to set dev environment for flyteidl, flyteadmin, flyteplugins, flytepropeller?

1. install [Flytectl](https://github.com/flyteorg/flytectl). It is a portable and lightweight command-line interface to work with Flyte.
```shell
# Step1: install the latest version of Flytectl
curl -sL https://ctl.flyte.org/install | bash
# flyteorg/flytectl info checking GitHub for latest tag
# flyteorg/flytectl info found version: 0.6.39 for v0.6.39/Linux/x86_64
# flyteorg/flytectl info installed ./bin/flytectl

# Step2: export Flytectl path baseed on previous log "flyteorg/flytectl info installed ./bin/flytectl"
export PATH=$PATH:/home/ubuntu/bin # replace with your path
```

2. Build an k3s cluster that runs minio and postgres pod.
[minio](https://min.io/) is a S3 compatible object store that will later be used to store task output, input etc.
[postgres](https://www.postgresql.org/) is am open source object-relational database that will later be used by flyteadmin to store all flyte infomation.

```shell
# Step1: start k3 scluster, create pod for postgres, minio. NOTE, We can not access Flyte UI yet, but we can access Minio console now.
flytectl demo start --dev
# 👨‍💻 Flyte is ready! Flyte UI is available at http://localhost:30080/console 🚀 🚀 🎉 
# ❇️ Run the following command to export demo environment variables for accessing flytectl
#         export FLYTECTL_CONFIG=/root/.flyte/config-sandbox.yaml 
# 🐋 Flyte sandbox ships with a Docker registry. Tag and push custom workflow images to localhost:30000
# 📂 The Minio API is hosted on localhost:30002. Use http://localhost:30080/minio/login for Minio console

# Step2: export FLYTECTL_CONFIG as previous log indicated.
export FLYTECTL_CONFIG=/root/.flyte/config-sandbox.yaml 

# Step3: kubeconfig will be auto copied to the user's main kubeconfig(default is `~/.kube/config`) with "flyte-sandbox" as the context name.
Check we can access K3s cluster. Check the postgres and minio is running.
# kubectl get pod -n flyte
# NAME                                                  READY   STATUS    RESTARTS   AGE
# flyte-sandbox-docker-registry-85745c899d-dns8q        1/1     Running   0          5m
# flyte-sandbox-kubernetes-dashboard-6757db879c-wl4wd   1/1     Running   0          5m
# flyte-sandbox-proxy-d95874857-2wc5n                   1/1     Running   0          5m
# flyte-sandbox-minio-645c8ddf7c-sp6cc                  1/1     Running   0          5m
# flyte-sandbox-postgresql-0                            1/1     Running   0          5m
```

3. [Optional] Access Minio console via `http://localhost:30080/minio/login`. The default 
  



3. download flyte repo and cd to the repo
```shell
git clone https://github.com/flyteorg/flyte.git
cd flyte
```

4. Now we will build a Single binary that bundles all backends(flyteidl, flyteadmin, flyteplugins, flytepropeller) and HTTP Server.
```shell
sudo apt-get -y install jq
go mod tidy
sudo make compile
flyte start --config flyte_local.yaml
```
https://github.com/flyteorg/flytectl/pull/370

```shell
# This is a sample configuration file.
# Real configuration when running inside K8s (local or otherwise) lives in a ConfigMap
# Look in the artifacts directory in the flyte repo for what's actually run
# https://github.com/lyft/flyte/blob/b47565c9998cde32b0b5f995981e3f3c990fa7cd/artifacts/flyteadmin.yaml#L72
# Flyte clusters can be run locally with this configuration
# flytectl demo start --dev
# flyte start --config flyte_local.yaml
propeller:
  rawoutput-prefix: "s3://my-s3-bucket/test/"
  kube-config: "/root/.kube/config"
  create-flyteworkflow-crd: true
webhook:
  certDir: /tmp/k8s-webhook-server/serving-certs
  serviceName: flyte-pod-webhook
  localCert: true
  servicePort: 9443
tasks:
  task-plugins:
    enabled-plugins:
      - container
      - sidecar
      - K8S-ARRAY
    default-for-task-types:
      - container: container
      - container_array: K8S-ARRAY
server:
  kube-config: "/root/.kube/config"
  httpPort: 30080
  serviceHttpEndpoint: http://localhost:30080/
  grpc:
    port: 30081
flyteadmin:
  runScheduler: false
database:
  postgres:
    port: 30001
    username: postgres
    password: postgres
    host: localhost
    dbname: flyteadmin
    options: "sslmode=disable"
storage:
  type: minio
  connection:
    access-key: minio
    auth-type: accesskey
    secret-key: miniostorage
    disable-ssl: true
    endpoint: "http://localhost:30002"
    region: my-region
  cache:
    max_size_mbs: 10
    target_gc_percent: 100
  container: "my-s3-bucket"
Logger:
  show-source: true
  level: 5
admin:
  endpoint: localhost:30081
  insecure: true
plugins:
  # All k8s plugins default configuration
  k8s:
    inject-finalizer: true
    default-env-vars:
      - AWS_METADATA_SERVICE_TIMEOUT: 5
      - AWS_METADATA_SERVICE_NUM_ATTEMPTS: 20
      - FLYTE_AWS_ENDPOINT: "http://minio.flyte:9000"
      - FLYTE_AWS_ACCESS_KEY_ID: minio
      - FLYTE_AWS_SECRET_ACCESS_KEY: miniostorage
  # Logging configuration
  logs:
    kubernetes-enabled: true
    kubernetes-url: "http://localhost:30082"
cluster_resources:
  refreshInterval: 5m
  templatePath: "/etc/flyte/clusterresource/templates"
  # -- Starts the cluster resource manager in standalone mode with requisite auth credentials to call flyteadmin service endpoints
  standaloneDeployment: false
  customData:
  - production:
    - projectQuotaCpu:
        value: "8"
    - projectQuotaMemory:
        value: "16Gi"
  - staging:
    - projectQuotaCpu:
        value: "8"
    - projectQuotaMemory:
        value: "16Gi"
  - development:
    - projectQuotaCpu:
        value: "8"
    - projectQuotaMemory:
        value: "16Gi"
  refresh: 5m
flyte:
  admin:
    disableClusterResourceManager: true
    disableScheduler: true
  propeller:
    disableWebhook: true
task_resources:
  defaults:
    cpu: 500m
    memory: 1Gi
  limits:
    cpu: 2
    memory: 4Gi
    gpu: 5
catalog-cache:
  endpoint: localhost:8081
  insecure: true
  type: datacatalog



```

5. replace with your own code
```shell

go mod edit -replace github.com/flyteorg/flyteadmin=/home/ubuntu/flyteadmin
go mod edit -replace github.com/flyteorg/flytepropeller=/home/ubuntu/flytepropeller
go mod edit -replace github.com/flyteorg/flyteidl=/home/ubuntu/flyteidl
go mod edit -replace github.com/flyteorg/flyteplugins=/home/ubuntu/flyteplugins
```

6. t
```shell
flyte start --config flyte_local.yaml
```

7. Note: ALL the components logs in this case will output together to the terminal, so you amy want to filter it.

## How to set dev environment for flyteconsole?

1. WIP
```shell
yarn build:types
yarn run build:prod
export BASE_URL=/console
export ADMIN_API_URL=http://localhost:30081
export DISABLE_AUTH=1
export ADMIN_API_USE_SSL="http"
```



## How to set dev environment for flytekit?


```shell
virtualenv ~/.virtualenvs/flytekit
source ~/.virtualenvs/flytekit/bin/activate
make setup
pip install -e .
pip install gsutil awscli
cd plugins
pip install -e .
```














