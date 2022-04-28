# k8swrld

Internet facing Kubernetes app on AWS EKS.

# Setup 

### Prerequisites

Make sure to have the following installed and configured:
- AWS account
- Docker
- kubectl - The Kubernetes command-line tool
- aws - The AWS command-line interface (CLI)
- eksctl - The official CLI for Amazon EKS
- helm - The package manager for Kubernetes


## Provisioning EKS cluster

Create a cluster using a config file:
```
eksctl create cluster -f cluster.yaml
```

Create an IAM OIDC identity provider for your cluster with the following command:
```
eksctl utils associate-iam-oidc-provider --cluster k8swrld --approve
```

Create an IAM policy for the AWS Load Balancer Controller:
```
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.1/docs/install/iam_policy.json

aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json

rm iam_policy.json
```

Create a Kubernetes service account named aws-load-balancer-controller in the kube-system namespace for the AWS Load Balancer Controller:
```
# set aws-account-id

eksctl create iamserviceaccount \
  --cluster=k8swrld \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name "AmazonEKSLoadBalancerControllerRole" \
  --attach-policy-arn=arn:aws:iam::<aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

Install the AWS Load Balancer Controller using Helm V3:
```
helm repo add eks https://aws.github.io/eks-charts

helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=k8swrld \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set image.repository=602401143452.dkr.ecr.us-east-1.amazonaws.com/amazon/aws-load-balancer-controller
```

Verify aws-load-balancer-controller installation:
```
kubectl get deployment -n kube-system aws-load-balancer-controller
```

You cluster should be ready by now. Some useful commands:
```
kubectl get nodes
kubectl get all -n k8swrld
kubectl get all -n kube-system
```

## Building docker image

Set a new release version in index.html and match the value in the following commands.

Build docker image using:

```
# set github-username

docker build -t ghcr.io/<github-username>/k8simg:1.1 .
```

Publish docker image using:
```
# set github-username

docker push ghcr.io/<github-username>/k8simg:1.1
```
Image that is pushed to the repository will be used for the rollout.

## Deploying application

Set a new release version in helm/values.yaml and match the value of "appVersion" in helm/Chart.yaml.

Install or upgrade a release with one command:
```
helm upgrade --install k8swrld ./helm
```

## Removing EKS

Uninstall a chart firsts to remove the load balancer:
```
helm uninstall k8swrld
```

Delete the cluster and worker nodes:
```
eksctl delete cluster --name k8swrld
```

# References
https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html
