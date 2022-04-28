# k8swrld

Internet facing Kubernetes app on AWS EKS.

# Setup 

### Prerequisites

Make sure to have following tools installed and configured:
- docker - The Docker command line
- kubectl - The Kubernetes command line tool
- aws - The AWS command line interface (CLI)
- eksctl - The official CLI for Amazon EKS
- helm - The package manager for Kubernetes


## Provisioning EKS cluster

Create a cluster using a config file:
```
eksctl create cluster -f cluster.yaml
```

Create an IAM OIDC identity provider for your cluster:
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

Create a Kubernetes service account for the AWS Load Balancer Controller:
```
# set <aws-account-id>

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

Your cluster should be ready:
```
kubectl get nodes
kubectl get all -n kube-system
```

## Building docker image

To release a new version increment the value in **index.html** and match it in the following commands:

**1. Build docker image using:**

```
# set <github-username>

docker build -t ghcr.io/<github-username>/k8simg:1.1 .
```

**2. Publish to GitHub Packages repository:**
```
# set <github-username>

echo $PAT | docker login ghcr.io --username <github-username> --password-stdin

docker push ghcr.io/<github-username>/k8simg:1.1
```
Published image will be used for the rollout.

## Deploying application

Set the correct release number in **helm/values.yaml** and also for "appVersion" in **helm/Chart.yaml**.

Install or upgrade a release with one command:
```
helm upgrade --install k8swrld ./helm
```

Verify resources:
```
kubectl get all -n k8swrld-all
```

To get the public endpoint run:
```
kubectl get ingress -n k8swrld-all
```

## Removing EKS cluster

Uninstall the chart firsts to remove the load balancer which is added by the app:
```
helm uninstall k8swrld
```

Remove the cluster:
```
eksctl delete cluster --name k8swrld
```

Verify:
```
eksctl get cluster k8swrld
```

# References
https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html
https://docs.aws.amazon.com/eks/latest/userguide/sample-deployment.html
https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html
https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry
