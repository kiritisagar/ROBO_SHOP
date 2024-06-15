# STEP1:SETUP EKS CLUSTER
__________________________________________________________________________________________________
# install awscli
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

# install unzip software
apt install unzip

# Unzip the Installer
unzip awscliv2.zip

# Run the Installer
sudo ./aws/install

# Verify the Installation:
aws --version

#create access key and secreat keys

# configure the AWS Command Line Interface & establish connection vm to aws
aws configure

# Ensure curl and tar are Installed
sudo apt update
sudo apt install curl tar -y

# Download the Correct eksctl File
curl -LO "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz"

# Extract the tar.gz File
tar -xzf eksctl_Linux_amd64.tar.gz

# Move eksctl to /usr/local/bin
sudo mv eksctl /usr/local/bin/

# Verify the Installation
eksctl version

# Install using Fargate
eksctl create cluster \
  --name demo-sagar \
  --region us-east-1 \
  --zones us-east-1a,us-east-1b,us-east-1c \
  --nodegroup-name demo-nodes \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 1 \
  --nodes-max 4 \
  --managed

______________________________________________________________________________________________
eksctl create cluster --name demo-cluster-three-tier-1 --region us-east-1
eksctl delete cluster --name demo-cluster-three-tier-1 --region us-east-1
_________________________________________________________________________________________________

# commands to configure IAM OIDC provider
export cluster_name=<CLUSTER-NAME>
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5) 

# Check if there is an IAM OIDC provider configured already
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4

# If not, run the below command
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve

____________________________________________________________________________________________________
# How to setup alb add on
# Download IAM policy

curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json

# Create IAM Policy
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json

# Create IAM Role
eksctl create iamserviceaccount \
  --cluster=demo-cluster-three-tier-2 \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::730335535801:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

# Deploy ALB controller

# install helm
snap install helm --classic

# Add helm repo
helm repo add eks https://aws.github.io/eks-charts

# Install

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \            
  -n kube-system \
  --set clusterName=<your-cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<region> \
  --set vpcId=<your-vpc-id>

# Verify that the deployments are running
kubectl get deployment -n kube-system aws-load-balancer-controller

# EBS CSI Plugin configuration
eksctl create iamserviceaccount \
    --name ebs-csi-controller-sa \
    --namespace kube-system \
    --cluster <YOUR-CLUSTER-NAME> \
    --role-name AmazonEKS_EBS_CSI_DriverRole \
    --role-only \
    --attach-policy-arn arn:aws:iam:<Account ID:>:aws:policy/service-role/AmazonEBSCSIDriverPolicy \
    --approve
# Run the following command. Replace with the name of your cluster, with your account ID.
eksctl create addon --name aws-ebs-csi-driver --cluster <YOUR-CLUSTER-NAME> --service-account-role-arn arn:aws:iam::<AWS-ACCOUNT-ID>:role/AmazonEKS_EBS_CSI_DriverRole --force

# Helm v3.x
snap install kubectl --classic
$ kubectl create ns robot-shop
$ helm install robot-shop --namespace robot-shop .

# goto repo
cd EKS/helm

kubectl apply -f ingress.yaml



_________________________________________________________________________________________________________________________
_________________________________________________________________________________________________________________________

# Three Tier Architecture Deployment on AWS EKS

Stan's Robot Shop is a sample microservice application you can use as a sandbox to test and learn containerised application orchestration and monitoring techniques. It is not intended to be a comprehensive reference example of how to write a microservices application, although you will better understand some of those concepts by playing with Stan's Robot Shop. To be clear, the error handling is patchy and there is not any security built into the application.

You can get more detailed information from my [blog post](https://www.instana.com/blog/stans-robot-shop-sample-microservice-application/) about this sample microservice application.

This sample microservice application has been built using these technologies:
- NodeJS ([Express](http://expressjs.com/))
- Java ([Spring Boot](https://spring.io/))
- Python ([Flask](http://flask.pocoo.org))
- Golang
- PHP (Apache)
- MongoDB
- Redis
- MySQL ([Maxmind](http://www.maxmind.com) data)
- RabbitMQ
- Nginx
- AngularJS (1.x)

The various services in the sample application already include all required Instana components installed and configured. The Instana components provide automatic instrumentation for complete end to end [tracing](https://docs.instana.io/core_concepts/tracing/), as well as complete visibility into time series metrics for all the technologies.

To see the application performance results in the Instana dashboard, you will first need an Instana account. Don't worry a [trial account](https://instana.com/trial?utm_source=github&utm_medium=robot_shop) is free.

## Build from Source
To optionally build from source (you will need a newish version of Docker to do this) use Docker Compose. Optionally edit the `.env` file to specify an alternative image registry and version tag; see the official [documentation](https://docs.docker.com/compose/env-file/) for more information.

To download the tracing module for Nginx, it needs a valid Instana agent key. Set this in the environment before starting the build.

```shell
$ export INSTANA_AGENT_KEY="<your agent key>"
```

Now build all the images.

```shell
$ docker-compose build
```

If you modified the `.env` file and changed the image registry, you need to push the images to that registry

```shell
$ docker-compose push
```

## Run Locally
You can run it locally for testing.

If you did not build from source, don't worry all the images are on Docker Hub. Just pull down those images first using:

```shell
$ docker-compose pull
```

Fire up Stan's Robot Shop with:

```shell
$ docker-compose up
```

If you want to fire up some load as well:

```shell
$ docker-compose -f docker-compose.yaml -f docker-compose-load.yaml up
```

If you are running it locally on a Linux host you can also run the Instana [agent](https://docs.instana.io/quick_start/agent_setup/container/docker/) locally, unfortunately the agent is currently not supported on Mac.

There is also only limited support on ARM architectures at the moment.

## Marathon / DCOS

The manifests for robotshop are in the *DCOS/* directory. These manifests were built using a fresh install of DCOS 1.11.0. They should work on a vanilla HA or single instance install.

You may install Instana via the DCOS package manager, instructions are here: https://github.com/dcos/examples/tree/master/instana-agent/1.9

## Kubernetes
You can run Kubernetes locally using [minikube](https://github.com/kubernetes/minikube) or on one of the many cloud providers.

The Docker container images are all available on [Docker Hub](https://hub.docker.com/u/robotshop/).

Install Stan's Robot Shop to your Kubernetes cluster using the [Helm](K8s/helm/README.md) chart.

To deploy the Instana agent to Kubernetes, just use the [helm](https://github.com/instana/helm-charts) chart.

## Accessing the Store
If you are running the store locally via *docker-compose up* then, the store front is available on localhost port 8080 [http://localhost:8080](http://localhost:8080/)

If you are running the store on Kubernetes via minikube then, find the IP address of Minikube and the Node Port of the web service.

```shell
$ minikube ip
$ kubectl get svc web
```

If you are using a cloud Kubernetes / Openshift / Mesosphere then it will be available on the load balancer of that system.

## Load Generation
A separate load generation utility is provided in the `load-gen` directory. This is not automatically run when the application is started. The load generator is built with Python and [Locust](https://locust.io). The `build.sh` script builds the Docker image, optionally taking *push* as the first argument to also push the image to the registry. The registry and tag settings are loaded from the `.env` file in the parent directory. The script `load-gen.sh` runs the image, it takes a number of command line arguments. You could run the container inside an orchestration system (K8s) as well if you want to, an example descriptor is provided in K8s directory. For End-user Monitoring ,load is not automatically generated but by navigating through the Robotshop from the browser .For more details see the [README](load-gen/README.md) in the load-gen directory.  

## Website Monitoring / End-User Monitoring

### Docker Compose

To enable Website Monioring / End-User Monitoring (EUM) see the official [documentation](https://docs.instana.io/website_monitoring/) for how to create a configuration. There is no need to inject the JavaScript fragment into the page, this will be handled automatically. Just make a note of the unique key and set the environment variable `INSTANA_EUM_KEY` and `INSTANA_EUM_REPORTING_URL` for the web image within `docker-compose.yaml`.

### Kubernetes

The Helm chart for installing Stan's Robot Shop supports setting the key and endpoint url required for website monitoring, see the [README](K8s/helm/README.md).

## Prometheus

The cart and payment services both have Prometheus metric endpoints. These are accessible on `/metrics`. The cart service provides:

* Counter of the number of items added to the cart

The payment services provides:

* Counter of the number of items perchased
* Histogram of the total number of items in each cart
* Histogram of the total value of each cart

To test the metrics use:

```shell
$ curl http://<host>:8080/api/cart/metrics
$ curl http://<host>:8080/api/payment/metrics
```

