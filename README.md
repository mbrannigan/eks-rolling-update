<p align="center">
  <img height="150px" src="./logo.png"  alt="EKS Rolling Update" title="EKS Rolling Update">
</p>

# EKS Rolling Update

[![Build Status](https://travis-ci.org/hellofresh/eks-rolling-update.svg?branch=master)](https://travis-ci.org/hellofresh/eks-rolling-update)

> EKS Rolling Update is a utility for updating the launch configuration or template of worker nodes in an EKS cluster.


- [Intro](#intro)
- [Requirements](#requirements)
- [Installation](#installation)
- [Usage](#usage)
- [Configuration](#configuration)
- [Contributing](#contributing)
- [License](#license)


<a name="intro"></a>
# Intro

EKS Rolling Update is a utility for updating the launch configuration or template of worker nodes in an EKS cluster. It
updates worker nodes in a rolling fashion and performs health checks of your EKS cluster to ensure no disruption to service.
To achieve this, it performs the following actions:

* Pauses Kubernetes Autoscaler (Optional)
* Finds a list of worker nodes that do not have a launch config or template that matches their ASG
* Scales up the desired capacity
* Ensures the ASGs are healthy and that the new nodes have joined the EKS cluster
* Cordons the outdated worker nodes
* Suspends AWS Autoscaling actions while update is in progress
* Drains outdated EKS outdated worker nodes one by one
* Terminates EC2 instances of the worker nodes one by one
* Detaches EC2 instances from the ASG one by one
* Scales down the ASG to original count (in case of failure)
* Resumes AWS Autoscaling actions
* Resumes Kubernetes Autoscaler (Optional)

<a name="requirements"></a>
## Requirements

* [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) installed
* `KUBECONFIG` environment variable set
* AWS credentials [configured](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/configuration.html#guide-configuration)

EKS Rolling Update expects that you have valid `KUBECONFIG` and AWS credentials set prior to running.

<a name="installation"></a>
## Installation

Install

```
virtualenv -p python3 venv
source venv/bin/activate
pip3 install -r requirements.txt
```

Set KUBECONFIG and context

```
export KUBECONFIG=~/.kube/config
ktx <environment>
```

<a name="usage"></a>
## Usage

```
usage: eks_rolling_update.py [-h] --cluster_name CLUSTER_NAME [--plan [PLAN]]

Rolling update on cluster

optional arguments:
  -h, --help            show this help message and exit
  --cluster_name CLUSTER_NAME, -c CLUSTER_NAME
                        the cluster name to perform rolling update on
  --plan [PLAN], -p [PLAN]
                        perform a dry run to see which instances are out of
                        date
```

Example:

```
eks-rolling-update.py -c my-eks-cluster
```

## Configuration

| Parameter                 | Description                                                                                                        | Default                              |
|---------------------------|--------------------------------------------------------------------------------------------------------------------|--------------------------------------|
| K8S_AUTOSCALER_ENABLED    | If True Kubernetes Autoscaler will be paused before running update                                                 | False                                 |
| K8S_AUTOSCALER_NAMESPACE  | Namespace where Kubernetes Autoscaler is deployed                                                                  | "default"                                   |
| K8S_AUTOSCALER_DEPLOYMENT | Deployment name of Kubernetes Autoscaler                                                                           | "cluster-autoscaler"                                   |
| ASG_DESIRED_STATE_TAG     | Temporary tag which will be saved to the ASG to store the state of the EKS cluster prior to update                 | eks-rolling-update:desired_capacity  |
| ASG_ORIG_CAPACITY_TAG     | Temporary tag which will be saved to the ASG to store the state of the EKS cluster prior to update                 | eks-rolling-update:original_capacity |
| ASG_ORIG_MAX_CAPACITY_TAG | Temporary tag which will be saved to the ASG to store the state of the EKS cluster prior to update                 | eks-rolling-update:original_max_capacity |
| CLUSTER_HEALTH_WAIT       | Number of seconds to wait after ASG has been scaled up before checking health of the cluster                       | 90                                   |
| GLOBAL_MAX_RETRY          | Number of attempts of a health check                                                                               | 12                                   |
| GLOBAL_HEALTH_WAIT        | Number of seconds to wait before retrying a health check                                                           | 20                                   |
| BETWEEN_NODES_WAIT        | Number of seconds to wait after removing a node before continuing on                                               | 0                                    |
| RUN_MODE                  | See Run Modes section below                                                                                        | 1                                    |
| DRY_RUN                   | If True, only a query will be run to determine which worker nodes are outdated without running an update operation | False                                |

## Run Modes
There are a number of different values which can be set for the `RUN_MODE` environment variable. 

`1` is the default.

| Mode Number   | Description                                                                                     |
|---------------|-------------------------------------------------------------------------------------------------|
| 1             | Scale up and cordon the outdated nodes of each ASG one-by-one, just before we drain them.       |
| 2             | Scale up and cordon the outdated nodes of all ASGs all at once at the beginning of the run.     |
| 3             | Cordon the outdated nodes of all ASGs at the beginning of the run but scale each ASG one-by-one.|

Each of them have different advantages and disadvantages.
* Scaling up all ASGs at once may cause AWS EC2 instance limits to be exceeded
* Only cordoning the nodes on a per-ASG basis will mean that pods are likely to be moved more than once
* Cordoning the nodes for all ASGs at once could cause issues if new pods needs to start during the process

## Examples
  - enable the updater to operate on `cluster-autoscaler`
    ```
    #
    # set your AWS environment before running eks rolling update
    #

    # NOTE: This examople will only work if you have cluster-autoscaler operating inside your cluster!
    # Let the updater know that cluster-autoscaler is running in your cluster

    $ export  K8S_AUTOSCALER_ENABLED=1 \
              K8S_AUTOSCALER_NAMESPACE="CA_NAMESPACE" \
              K8S_AUTOSCALER_DEPLOYMENT="CA_DEPLOYMENT_NAME"

    # plan
    $ python eks_rolling_update.py --cluster_name YOUR_EKS_CLUSTER_NAME --plan
    ...

    # apply changes
    $ python eks_rolling_update.py --cluster_name YOUR_EKS_CLUSTER_NAME
    ```
- disable operations on `cluster-autoscaler`
  ```
    $ unset K8S_AUTOSCALER_ENABLED
  ```
- enable features only for one run
  ```
    # DRY_RUN will only be used by the current updater session
    $ DRY_RUN=1 python eks_rolling_update.py --cluster_name YOUR_CLUSTER_NAME

    # operate on cluster-autoscaler only for this updater session
    $ K8S_AUTOSCALER_ENABLED=1 \
      K8S_AUTOSCALER_NAMESPACE="somenamespace" \
      K8S_AUTOSCALER_DEPLOYMENT="deploymentname" \
      python eks_rolling_update.py --cluster_name YOUR_CLUSTER_NAME
  ```
- `.env` file
  ```
  # you can use .env file within your project directory to load updater settings
  # e.g:
  
  $ cat .env
  DRY_RUN=1
  $
  ```

<a name="contributing"></a>
## Contributing

Please read [CONTRIBUTING.md](CONTRIBUTING.md) for details on our code of conduct, and the process for submitting pull requests to us.

<a name="licence"></a>
## License

This project is licensed under the Apache 2.0 License - see the [LICENSE](LICENSE) file for details
