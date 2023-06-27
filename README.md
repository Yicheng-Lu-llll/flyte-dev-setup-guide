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

3. [Optional] Access Minio console via `http://localhost:30080/minio/login`.
The default Username is `minio`, the default Password is `miniostorage`.
Later, when developing, You amy need to look at the input.pb, output.pb or deck.html in the Minio.
 
5. now, let's start all backends(flyteidl, flyteadmin, flyteplugins, flytepropeller) and HTTP Server in a single binary 
```shell
# Step1: download flyte
git clone https://github.com/flyteorg/flyte.git
cd flyte

# Step2: build a Single binary that bundles all backends(flyteidl, flyteadmin, flyteplugins, flytepropeller) and HTTP Server.
The version of flyteidl, flyteadmin, flyteplugins and flytepropeller you used to build Single binary is defined in `go.mod`.
sudo apt-get -y install jq # You may need to install jq
go mod tidy
sudo make compile

# Step3: Running the Single binary. flyte_local.yaml is the config file. It is write to fit with the all you previous built. So, you do not need to change the flyte_local.yaml.
flyte start --config flyte_local.yaml
# All logs from flyteadmin, flyteplugins, flytepropeller ... will appear in the terminal
```

6. [Optional] Now, you are able to access Flyte UI:http://localhost:30080/console.
   
7. Previously, we build Single binary that bundles all backends(flyteidl, flyteadmin, flyteplugins, flytepropeller) and HTTP Server, now let's replace with you own code.
For all below instruction, I will assume you will change flyteidl, flyteadmin, flyteplugins and flytepropeller at the same time(features that involve multiple components),
If you actually do not need to change some components, simpliy ignore the instruction for that component.
```shell
# Step1: Modify the source code for flyteidl, flyteadmin, flyteplugins and flytepropeller. 

# Step2: flyteidl/flyteadmin/flyteplugins/flytepropeller uses go1.19, so make sure you switch to go1.19.
export PATH=$PATH:$(go env GOPATH)/bin
go install golang.org/dl/go1.19@latest
go1.19 download
export GOROOT=$(go1.19 env GOROOT)
export PATH="$GOROOT/bin:$PATH"


# Step3.1: Under flyteidl folder, before building the Single binary, You should run:
make lint
make generate
make test_unit

# Step3.2: Under flyteadmin folder, before building the Single binary, You should run:
go mod edit -replace github.com/flyteorg/flytepropeller=/home/ubuntu/flytepropeller #replace with your own local path to flytepropeller
go mod edit -replace github.com/flyteorg/flyteidl=/home/ubuntu/flyteidl #replace with your own local path to flyteidl
go mod edit -replace github.com/flyteorg/flyteplugins=/home/ubuntu/flyteplugins # replace with your own local path to flyteplugins
make lint
make generate
make test_unit

# Step3.3: Under flyteplugins folder, before building the Single binary, You should run:
go mod edit -replace github.com/flyteorg/flyteidl=/home/ubuntu/flyteidl #replace with your own local path to flyteidl

# Step3.4: Under flytepropeller folder, before building the Single binary, You should run:
go mod edit -replace github.com/flyteorg/flyteidl=/home/ubuntu/flyteidl #replace with your own local path to flyteidl
go mod edit -replace github.com/flyteorg/flyteplugins=/home/ubuntu/flyteplugins # replace with your own local path to flyteplugins
make lint
make generate
make test_unit

# Step4: Now, We can build the Single binary, Under Flyte folder, Run `go mod edit -replace`. This will replace with you own code. 
go mod edit -replace github.com/flyteorg/flyteadmin=/home/ubuntu/flyteadmin #replace with your own local path to flyteadmin
go mod edit -replace github.com/flyteorg/flytepropeller=/home/ubuntu/flytepropeller #replace with your own local path to flytepropeller
go mod edit -replace github.com/flyteorg/flyteidl=/home/ubuntu/flyteidl #replace with your own local path to flyteidl
go mod edit -replace github.com/flyteorg/flyteplugins=/home/ubuntu/flyteplugins # replace with your own local path to flyteplugins

# Step5: Rebuild and rerun the Single binary based on your own code.
go mod tidy
sudo make compile
flyte start --config flyte_local.yaml
```

8. After finish developing, we can teardown the k3s cluster.
```shell
flytectl demo teardown
# context removed for "flyte-sandbox".
# 🧹 🧹 Sandbox cluster is removed successfully.
# ❇️ Run the following command to unset sandbox environment variables for accessing flytectl
#        unset FLYTECTL_CONFIG 
```


## How to set dev environment for flytekit?

1. If you also change the code for flyteidl, flyteadmin, flyteplugins or flytepropeller, you can refer to the `How to set dev environment for flyteidl, flyteadmin, flyteplugins, flytepropeller?` to build the backends.
If not, We can simply run `flytectl demo start`

```shell
virtualenv ~/.virtualenvs/flytekit
source ~/.virtualenvs/flytekit/bin/activate
make setup
pip install -e .
pip install gsutil awscli
cd plugins
pip install -e .
```



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













