### EKS End to End Project
I have created EKS(Elastic kubernetes clustor) using fargate(serverless compute) and one public subnet and one private subnet. in public subnet i have configured load balancer and in private subnet i have created ingress resources and configured ingress controller. and ingress controller integrate with load balancer

when a user from the external world access to the application then first his request goes to load balancer which is in the public subnet then load balancer redirect the request to ingress controller which will watch for the ingress resources then ingress controller route the external traffic to pod through service so that user can access the application.

#### Pre-requisites
1. kubectl
2. eksctl
3. aws cli

#### Install kubectl
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
  curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
chmod +x kubectl
mkdir -p ~/.local/bin
mv ./kubectl ~/.local/bin/kubectl
kubectl version --client
```
#### Install eksctl
```
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz
sudo mv /tmp/eksctl /usr/local/bin
```
#### Install awscli
```

```
Configure aws with awscli using access key and secret access key id
```
aws configure
```
#### Create EKS-cluster using Fargate
```
eksctl create cluster --name demo-cluster --region us-east-1 --fargate
```
#### Update configuration
```
aws eks update-kubeconfig --name demo-cluster --region us-east-1
```
#### Create Fargate profile
```
eksctl create fargateprofile \
    --cluster demo-cluster \
    --region us-east-1 \
    --name alb-sample-app \
    --namespace game-2048
```
#### Deploy the deployment, service and Ingress
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```
#### commands to configure IAM OIDC provider 
```
eksctl utils associate-iam-oidc-provider --cluster demo-cluster --approve
```
#### Download IAM policy
```
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
```
#### Create IAM Policy
```
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```
#### Create IAM Role
```
eksctl create iamserviceaccount \
  --cluster=demo-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

#### Deploy ALB controller:

#### Add helm repo

```
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
```
#### Install aws load balancer
```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
  --set clusterName=demo-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=<your-vpc-id>
```
#### Verify that the deployments are running
```
kubectl get deployment -n kube-system aws-load-balancer-controller
```
