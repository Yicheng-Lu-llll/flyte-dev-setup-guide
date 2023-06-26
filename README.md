# !!!WIP!!!


# Setting Sail with Flyte: A Beginners Guide to Dev Environment Setup

If you're reading this, you likely have an interest in Flyte, an open-source data, ML, and infrastructure orchestrator that simplifies the creation, scaling, and maintenance of concurrent workflows. Like many in the open-source community, you might be eager to contribute but find the initial setup process a daunting hurdle.

This experience is quite common for newcomers—I too struggled with the setup when I first started contributing (many thanks to Kevin Su for all his help!). Building a strong Flyte community, where everyone feels welcome to participate, requires us to break down these barriers and make the contribution process more accessible. With this goal in mind, I've crafted this guide as a supplement to the official resources. It aims to provide a clear and simple, step-by-step approach to setting up a local development environment, reflecting the latest updates and modifications. It especially simplifies the setup for features that involve multiple components.

So, let's embark on this journey together, easing your transition into the world of Flyte contributions.



## Prequest

- This guide is used and tested on AWS EC2 with ubuntu 22.04 image.
- Docker

## Content
- How to set dev environment for flyteidl, flyteadmin, flyteplugins, flytepropeller?
  
- How to set dev environment for flyteconsole?
  
- How to set dev environment for flytekit?
  - Run workflow locally
  - Run workflow in sandbox
  - Develop flytekit plugins, using Deck as an example
  
- Putting everyhing together, How to set dev environment if your PR involve multiple components?
  
- FAQ

## How to set dev environment for flyteidl, flyteadmin, flyteplugins, flytepropeller?

1. install Flytectl
```shell
curl -sL https://ctl.flyte.org/install | bash
```

2. Build an k3s cluster that runs minio and postgres pod.
minio is used to simulate AWS S3 that will later used to stroe task output, input etc.
postgres is used by flyteadmin to store all flyte infomation.
```shell
flytectl demo start --dev

kubectl get pod -n flyte
```

3. download flyte repo and cd to the repo
```shell
git clone https://github.com/flyteorg/flyte.git
cd flyte
```

4. Now we will build a Single binary that bundles all backends(flyteidl, flyteadmin, flyteplugins, flytepropeller)
```shell
go mod tidy
sudo make compile
flyte start --config flyte_local.yaml
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














