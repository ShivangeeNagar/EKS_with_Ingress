# EKS_with_Ingress

Prerequisites

kubectl – A command line tool for working with Kubernetes clusters. For more information, see Installing or updating kubectl.

eksctl – A command line tool for working with EKS clusters that automates many individual tasks. For more information, see Installing or updating.

AWS CLI – A command line tool for working with AWS services, including Amazon EKS. For more information, see Installing, updating, and uninstalling the AWS CLI in the AWS Command Line Interface User Guide. After installing the AWS CLI, we recommend that you also configure it. For more information, see Quick configuration with aws configure in the AWS Command Line Interface User Guide

1) Install EKS / Create cluster using EKS & Fargate ( through CLI) - This creates a public/ private subnet for us. ( In the private subnet, we will deploy our application) Don't worry, this might take 15/20 minutes! Make sure to checkout your cluster on AWS console and its defualt resources/ configuration once you create it.

eksctl create cluster --name demo-cluster --region us-east-1 --fargate

2) Instead of always going to the resource tab and verifying from there, let's download the kubeconfig file -

aws eks update-kubeconfig --name demo-cluster --region us-east-1

3) Create a new farget profile for a different namespace for your application. Confirm this fargate profile visibility on aws console after the success of the command.

eksctl create fargateprofile \
    --cluster demo-cluster \
    --region us-east-1 \
    --name alb-sample-app \
    --namespace game-2048

4) Now deploying my yaml files - deployment, service and ingress

kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml

5) To checkout the running pods / status -

kubectl get pods -n game-2048

6) To checkout the service -

kubectl get svc -n game-2048

Note : when you see the service, you will notice, it does not have any external IP yet

7) To checkout your ingress -

kubectl get ingress -n game-2048

Note : when you see the ingress, please note that it does not have any address yet, which means there has to be an ingress controller which can provide this address and that address can be used to access your application on the browser.

8) In order to create a ALB ingress controller, there is a prerequisite - we must integrate an indentity provider for this pod. In our case, we will create an IAM OIDC provider as our pod ( controller) needs access to other aws resources.

eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve

9) Also, for the same reason, we need to create IAM policy -

curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json

aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json

10) Crate IAM role / IAM service account for our controller (pod)

eksctl create iamserviceaccount \
  --cluster=<your-cluster-name> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

11) Creation of the Controller -

    a) We are going to use helm charts for this. The helm chart will create the actual controller and will use the service account for running the pod

       brew install helm

    b) Add the helm repo -

       helm repo add eks https://aws.github.io/eks-charts

    c) helm repo update eks -

       helm repo update eks

12) Now, install aws load balancer controller usinh helm charts -

  helm install aws-load-balancer-controller eks/aws-load-balancer-controller \            
  -n kube-system \
  --set clusterName=<your-cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<region> \
  --set vpcId=<your-vpc-id>

13) Verify that the deployments are running -

kubectl get deployment -n kube-system aws-load-balancer-controller

14) List created  and deployed pods -

kubectl get pods -n kube-system

kubectl get deploy -n kube-system

15) In case your aws load balancer controller is not createe and wish to troubleshoot it. Use this command and check the status in the file and the error it generates -

kubectl edit deploy/aws-load-balancer-controller -n kube-system

16) Final step -

Go inside EC2, if you see a load balancer there, Yippeee! This load balancer is created by the aws-load-balancer controller using our ingress resource.

Run this command - kubectl get ingress -n game-2048 

Checkout the address on this resource now! 
This address is same as the DNS ip address of your load balanacer on aws console. Now if your load balancer state is active. 
Go to this URL and checkout your web-app being sucessfully deployed on the browser, finally accessed from the outside world!




