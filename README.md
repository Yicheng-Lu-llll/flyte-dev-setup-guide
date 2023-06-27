# !!!WIP!!!


# Setting Sail with Flyte: A Beginners Guide to Dev Environment Setup

If you're reading this, you likely have an interest in Flyte, an open-source data, ML, and infrastructure orchestrator that simplifies the creation, scaling, and maintenance of concurrent workflows. Like many in the open-source community, you might be eager to contribute but find the initial setup process a daunting hurdle.

This experience is quite common for newcomers—I too struggled with the setup when I first started contributing (many thanks to Kevin Su for all his help!). Building a strong Flyte community, where everyone feels welcome to participate, requires us to break down these barriers and make the contribution process more accessible. With this goal in mind, I've written this guide as a supplement to the official resources. It aims to provide a clear and simple, step-by-step approach to setting up a local development environment, reflecting the latest updates and modifications. It especially simplifies the setup for features that involve multiple components.

So, let's embark on this journey together, easing your transition into the world of Flyte contributions.



## Prequest

- This guide is used and tested on AWS EC2 with ubuntu 22.04 image.
- Docker
- Kubectl
- Go

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

1. install [Flytectl](https://github.com/flyteorg/flytectl), a portable and lightweight command-line interface to work with Flyte.
```shell
# Step1: Install the latest version of Flytectl
curl -sL https://ctl.flyte.org/install | bash
# flyteorg/flytectl info checking GitHub for latest tag
# flyteorg/flytectl info found version: 0.6.39 for v0.6.39/Linux/x86_64
# flyteorg/flytectl info installed ./bin/flytectl

# Step2: Export Flytectl path based on the previous log "flyteorg/flytectl info installed ./bin/flytectl"
export PATH=$PATH:/home/ubuntu/bin # replace with your path
```

2. Build a k3s cluster that runs Minio and Postgres pods.
[minio](https://min.io/) is an S3-compatible object store that will be used later to store task output, input, etc.
[postgres](https://www.postgresql.org/) is an open-source object-relational database that will later be used by flyteadmin to store all Flyte information.

```shell
# Step1: Start k3s cluster, create pods for Postgres and Minio. Note: We cannot access Flyte UI yet! but we can access the Minio console now.
flytectl demo start --dev
# 👨‍💻 Flyte is ready! Flyte UI is available at http://localhost:30080/console 🚀 🚀 🎉 
# ❇️ Run the following command to export demo environment variables for accessing flytectl
#         export FLYTECTL_CONFIG=/home/ubuntu/.flyte/config-sandbox.yaml 
# 🐋 Flyte sandbox ships with a Docker registry. Tag and push custom workflow images to localhost:30000
# 📂 The Minio API is hosted on localhost:30002. Use http://localhost:30080/minio/login for Minio console

# Step2: Export FLYTECTL_CONFIG as the previous log indicated.
FLYTECTL_CONFIG=/home/ubuntu/.flyte/config-sandbox.yaml

# Step3: The kubeconfig will be automatically copied to the user's main kubeconfig (default is `~/.kube/config`) with "flyte-sandbox" as the context name.
# Check that we can access the K3s cluster. Verify that Postgres and Minio are running.
kubectl get pod -n flyte
# NAME                                                  READY   STATUS    RESTARTS   AGE
# flyte-sandbox-docker-registry-85745c899d-dns8q        1/1     Running   0          5m
# flyte-sandbox-kubernetes-dashboard-6757db879c-wl4wd   1/1     Running   0          5m
# flyte-sandbox-proxy-d95874857-2wc5n                   1/1     Running   0          5m
# flyte-sandbox-minio-645c8ddf7c-sp6cc                  1/1     Running   0          5m
# flyte-sandbox-postgresql-0                            1/1     Running   0          5m
```

3. [Optional] You can access the Minio console via http://localhost:30080/minio/login.
The default Username is `minio` and the default Password is `miniostorage`. You might need to look at input.pb, output.pb or deck.html, etc in Minio when you are developing.
 
5. now, let's start all backends(flyteidl, flyteadmin, flyteplugins, flytepropeller) and HTTP Server in a single binary 
```shell
# Step1: Download flyte repo
git clone https://github.com/flyteorg/flyte.git
cd flyte

# Step2: Build a single binary that bundles all the backends (flyteidl, flyteadmin, flyteplugins, flytepropeller) and HTTP Server.
# The versions of flyteidl, flyteadmin, flyteplugins, and flytepropeller used to build the single binary are defined in `go.mod`.
sudo apt-get -y install jq # You may need to install jq
go mod tidy
sudo make compile

# Step3: Running the single binary. `flyte_local.yaml` is the config file. It is written to fit all your previous builds. So, you don't need to change `flyte_local.yaml`.
flyte start --config flyte_local.yaml
# All logs from flyteadmin, flyteplugins, flytepropeller, etc. will appear in the terminal.
```

6. [Optional] Now, you can access the Flyte UI at http://localhost:30080/console.
   
7. Previously, we built a single binary that bundles all backends (flyteidl, flyteadmin, flyteplugins, flytepropeller) and HTTP Server. Now let's build with your own code. 
The following instructions assume that you'll change flyteidl, flyteadmin, flyteplugins, and flytepropeller simultaneously (features that involve multiple components).
If you don't need to change some components, simply ignore the instruction for that component.
```shell
# Step1: Modify the source code for flyteidl, flyteadmin, flyteplugins, and flytepropeller.

# Step2: Flyteidl, flyteadmin, flyteplugins, and flytepropeller use go1.19, so make sure to switch to go1.19.
export PATH=$PATH:$(go env GOPATH)/bin
go install golang.org/dl/go1.19@latest
go1.19 download
export GOROOT=$(go1.19 env GOROOT)
export PATH="$GOROOT/bin:$PATH"


# Step3.1: In the flyteidl folder, before building the single binary, you should run:
make lint
make generate
make test_unit

# Step3.2: In the flyteadmin folder, before building the single binary, you should run:
go mod edit -replace github.com/flyteorg/flytepropeller=/home/ubuntu/flytepropeller #replace with your own local path to flytepropeller
go mod edit -replace github.com/flyteorg/flyteidl=/home/ubuntu/flyteidl #replace with your own local path to flyteidl
go mod edit -replace github.com/flyteorg/flyteplugins=/home/ubuntu/flyteplugins # replace with your own local path to flyteplugins
make lint
make generate
make test_unit

# Step3.3: In the flyteplugins folder, before building the single binary, you should run:
go mod edit -replace github.com/flyteorg/flyteidl=/home/ubuntu/flyteidl #replace with your own local path to flyteidl

# Step3.4: In the flytepropeller folder, before building the single binary, you should run:
go mod edit -replace github.com/flyteorg/flyteidl=/home/ubuntu/flyteidl #replace with your own local path to flyteidl
go mod edit -replace github.com/flyteorg/flyteplugins=/home/ubuntu/flyteplugins # replace with your own local path to flyteplugins
make lint
make generate
make test_unit

# Step4: Now, you can build the single binary. In the Flyte folder, run `go mod edit -replace`. This will replace the code with your own.
go mod edit -replace github.com/flyteorg/flyteadmin=/home/ubuntu/flyteadmin #replace with your own local path to flyteadmin
go mod edit -replace github.com/flyteorg/flytepropeller=/home/ubuntu/flytepropeller #replace with your own local path to flytepropeller
go mod edit -replace github.com/flyteorg/flyteidl=/home/ubuntu/flyteidl #replace with your own local path to flyteidl
go mod edit -replace github.com/flyteorg/flyteplugins=/home/ubuntu/flyteplugins # replace with your own local path to flyteplugins

# Step5: Rebuild and rerun the single binary based on your own code.
go mod tidy
sudo make compile
flyte start --config flyte_local.yaml
```
7. Let's quickly test it by running a hellow world workflow.
```shell
# Step1: install flytekit
pip install flytekit && export PATH=$PATH:~/.local/bin

# Step2: flytesnacks repo provide lots of useful examples
git clone https://github.com/flyteorg/flytesnacks && cd flytesnacks/cookbook

# Step3: Before running worflow, create flytesnacks-development namespace. Becasue by default(without creating new project), task pod will be running in flytesnacks-development namespace.
kubectl create namespace flytesnacks-development

# Step4: Run a hello_world example
pyflyte run --remote core/flyte_basics/hello_world.py my_wf
# Go to http://localhost:30080/console/projects/flytesnacks/domains/development/executions/fd63f88a55fed4bba846 to see execution in the console.
```
   
8. After you finish developing, you can tear down the k3s cluster.
```shell
flytectl demo teardown
# context removed for "flyte-sandbox".
# 🧹 🧹 Sandbox cluster is removed successfully.
# ❇️ Run the following command to unset sandbox environment variables for accessing flytectl
#        unset FLYTECTL_CONFIG 
```


## How to set dev environment for flytekit?

1. If you also change the code for flyteidl, flyteadmin, flyteplugins or flytepropeller, you can refer to the `How to set dev environment for flyteidl, flyteadmin, flyteplugins, flytepropeller?` to build the backends.
If not, run `flytectl demo start` to start 

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













