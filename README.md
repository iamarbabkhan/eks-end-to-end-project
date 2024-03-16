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
![Screenshot-from-2024-03-15-19-48-32.png](https://i.postimg.cc/xTvHkYrJ/Screenshot-from-2024-03-15-19-48-32.png)
#### Install eksctl
```
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz
sudo mv /tmp/eksctl /usr/local/bin
```
![Screenshot-from-2024-03-15-19-53-37.png](https://i.postimg.cc/TPGtyQHG/Screenshot-from-2024-03-15-19-53-37.png)
#### Install awscli
```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
./aws/install -i /usr/local/aws-cli -b /usr/local/bin
aws --version
```
![Screenshot-from-2024-03-15-20-00-52.png](https://i.postimg.cc/vTWzHVLD/Screenshot-from-2024-03-15-20-00-52.png)
Configure aws with awscli using access key and secret access key id
```
aws configure
```
![image-3.png](https://i.postimg.cc/Jnn7LfJ2/image-3.png)
#### Create EKS-cluster using Fargate
```
eksctl create cluster --name demo-cluster --region us-east-1 --fargate
```
![Screenshot-from-2024-03-15-22-50-46.png](https://i.postimg.cc/TPxDddHR/Screenshot-from-2024-03-15-22-50-46.png)
![Screenshot-from-2024-03-15-22-51-31.png](https://i.postimg.cc/sgL3mw7b/Screenshot-from-2024-03-15-22-51-31.png)
#### Update configuration
```
aws eks update-kubeconfig --name demo-cluster --region us-east-1
```
![Screenshot-from-2024-03-15-22-53-36.png](https://i.postimg.cc/fyQc9c89/Screenshot-from-2024-03-15-22-53-36.png)
![Screenshot-from-2024-03-15-22-54-31.png](https://i.postimg.cc/tJbrjGQw/Screenshot-from-2024-03-15-22-54-31.png)
#### Create Fargate profile
```
eksctl create fargateprofile \
    --cluster demo-cluster \
    --region us-east-1 \
    --name alb-sample-app \
    --namespace game-2048
```
![Screenshot-from-2024-03-15-22-56-27.png](https://i.postimg.cc/ry0PKZnh/Screenshot-from-2024-03-15-22-56-27.png)
![Screenshot-from-2024-03-15-22-59-01.png](https://i.postimg.cc/t45rLKJg/Screenshot-from-2024-03-15-22-59-01.png)
#### Deploy the deployment, service and Ingress
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```
![Screenshot-from-2024-03-15-23-00-36.png](https://i.postimg.cc/T3TBMmqQ/Screenshot-from-2024-03-15-23-00-36.png)
#### commands to configure IAM OIDC provider 
```
eksctl utils associate-iam-oidc-provider --cluster demo-cluster --approve
```
![Screenshot-from-2024-03-15-23-05-14.png](https://i.postimg.cc/3xfK3bbK/Screenshot-from-2024-03-15-23-05-14.png)
#### Download IAM policy
```
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
```
![Screenshot-from-2024-03-15-23-06-19.png](https://i.postimg.cc/dVs1sCs9/Screenshot-from-2024-03-15-23-06-19.png)
#### Create IAM Policy
```
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```
![Screenshot-from-2024-03-15-23-07-05.png](https://i.postimg.cc/Y9SmJXhk/Screenshot-from-2024-03-15-23-07-05.png)
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
![Screenshot-from-2024-03-15-23-10-58.png](https://i.postimg.cc/9fLDtG1k/Screenshot-from-2024-03-15-23-10-58.png)
#### Install snap
```
sudo snap install helm --classic
```
![Screenshot-from-2024-03-15-23-15-10.png](https://i.postimg.cc/Bvjgdcbr/Screenshot-from-2024-03-15-23-15-10.png)
#### Deploy ALB controller:

#### Add helm repo

```
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
```
![Screenshot-from-2024-03-15-23-16-08.png](https://i.postimg.cc/mZyQp6Vc/Screenshot-from-2024-03-15-23-16-08.png)
#### Install aws load balancer
```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
  --set clusterName=demo-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=<your-vpc-id>
```
![Screenshot-from-2024-03-15-23-19-04.png](https://i.postimg.cc/FH67TRvg/Screenshot-from-2024-03-15-23-19-04.png)
#### Verify that the deployments are running
```
kubectl get deployment -n kube-system aws-load-balancer-controller
```
![Screenshot-from-2024-03-16-19-13-03.png](https://i.postimg.cc/N0N1fy91/Screenshot-from-2024-03-16-19-13-03.png)
#### To access the application
```
kubectl get ingress -n game-2048
```
![Screenshot-from-2024-03-16-19-13-30.png](https://i.postimg.cc/DzmS6x0X/Screenshot-from-2024-03-16-19-13-30.png)
![Screenshot-from-2024-03-16-19-15-24.png](https://i.postimg.cc/q771LH2x/Screenshot-from-2024-03-16-19-15-24.png)
