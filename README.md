# kubernates

K8-Link full doc

https://www.learnitguide.net/2018/10/kubernetes-tutorial-for-beginners-full.html

## Kubernetes Ingress with AWS ALB Ingress Controller

**Link** : https://aws.amazon.com/blogs/opensource/kubernetes-ingress-aws-alb-ingress-controller/


Kubernetes Ingress with AWS ALB Ingress Controller
by Yang Yang and Michael Hausenblas | on 20 NOV 2018 | in Amazon Elastic Kubernetes Service, Elastic Load Balancing, Open Source | Permalink |  Comments |  Share
Note: This post has been updated in January, 2020, to reflect new best practices in container security since we launched native least-privileges support at the pod level, and the instructions have been updated for the latest controller version. You can also learn about Using ALB Ingress Controller with Amazon EKS on Fargate.

Kubernetes Ingress is an API resource that allows you manage external or internal HTTP(S) access to Kubernetes services running in a cluster. Amazon Elastic Load Balancing Application Load Balancer (ALB) is a popular AWS service that load balances incoming traffic at the application layer (layer 7) across multiple targets, such as Amazon EC2 instances, in a region. ALB supports multiple features including host or path based routing, TLS (Transport Layer Security) termination, WebSockets, HTTP/2, AWS WAF (Web Application Firewall) integration, integrated access logs, and health checks.

The open source AWS ALB Ingress controller triggers the creation of an ALB and the necessary supporting AWS resources whenever a Kubernetes user declares an Ingress resource in the cluster. The Ingress resource uses the ALB to route HTTP(S) traffic to different endpoints within the cluster. The AWS ALB Ingress controller works on any Kubernetes cluster including Amazon Elastic Kubernetes Service (Amazon EKS).

Terminology
We will use the following acronyms to describe the Kubernetes Ingress concepts in more detail:

ALB: AWS Application Load Balancer
ENI: Elastic Network Interfaces
NodePort: When a user sets the Service type field to NodePort, Kubernetes allocates a static port from a range and each worker node will proxy that same port to said Service.
How Kubernetes Ingress works with aws-alb-ingress-controller
The following diagram details the AWS components that the aws-alb-ingress-controller creates whenever an Ingress resource is defined by the user. The Ingress resource routes ingress traffic from the ALB to the Kubernetes cluster.

diagram: How Kubernetes Ingress works with aws-alb-ingress-controller

Ingress Creation
Following the steps in the numbered blue circles in the above diagram:

The controller watches for Ingress events from the API server. When it finds Ingress resources that satisfy its requirements, it starts the creation of AWS resources.
An ALB is created for the Ingress resource.
TargetGroups are created for each backend specified in the Ingress resource.
Listeners are created for every port specified as Ingress resource annotation. If no port is specified, sensible defaults (80 or 443) are used.
Rules are created for each path specified in your Ingress resource. This ensures that traffic to a specific path is routed to the correct TargetGroup created.
Ingress traffic
AWS ALB Ingress controller supports two traffic modes: instance mode and ip mode. Users can explicitly specify these traffic modes by declaring the alb.ingress.kubernetes.io/target-type annotation on the Ingress and the service definitions.

instance mode: Ingress traffic starts from the ALB and reaches the NodePort opened for your service. Traffic is then routed to the pods within the cluster.
ip mode: Ingress traffic starts from the ALB and reaches the pods within the cluster directly. To use this mode, the networking plugin for the Kubernetes cluster must use a secondary IP address on ENI as pod IP, also known as the AWS CNI plugin for Kubernetes.
Note: If you are using the ALB ingress with EKS on Fargate you want to use ip mode.

Deploy Amazon EKS with eksctl
First, let’s deploy an Amazon EKS cluster with eksctl. For this you can use, for example, Homebrew on macOS:

brew install weaveworks/tap/eksctl
Bash
Next, create the EKS cluster with cluster name attractive-gopher:

eksctl create cluster --name=attractive-gopher
Bash
Now create an IAM OIDC provider and associate it with your cluster:

eksctl utils associate-iam-oidc-provider --cluster=attractive-gopher --approve
Bash
Learn more about IAM Roles for Service Accounts in the Amazon EKS documentation.

Deploy AWS ALB Ingress controller
Next, let’s deploy the AWS ALB Ingress controller into our EKS cluster using the steps below.

First off, deploy the relevant RBAC roles and role bindings as required by the AWS ALB Ingress controller:

kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/rbac-role.yaml
Bash
Next, create an IAM policy named ALBIngressControllerIAMPolicy to allow the ALB Ingress controller to make AWS API calls on your behalf. Record the Policy.Arn in the command output, you will need it in the next step:

aws iam create-policy \
    --policy-name ALBIngressControllerIAMPolicy \
    --policy-document https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/iam-policy.json
Bash
Next, create a Kubernetes service account and an IAM role (for the pod running the AWS ALB Ingress controller) by substituting $PolicyARN with the recorded value from the previous step:

eksctl create iamserviceaccount \
       --cluster=attractive-gopher \
       --namespace=kube-system \
       --name=alb-ingress-controller \
       --attach-policy-arn=$PolicyARN \
       --override-existing-serviceaccounts \
       --approve
Bash
Then deploy the AWS ALB Ingress controller:

curl -sS "https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/alb-ingress-controller.yaml" \
     | sed "s/# - --cluster-name=devCluster/- --cluster-name=attractive-gopher/g" \
     | kubectl apply -f -
Bash
Finally, verify that the deployment was successful and the controller started:

$ kubectl logs -n kube-system $(kubectl get po -n kube-system | egrep -o alb-ingress[a-zA-Z0-9-]+)
-------------------------------------------------------------------------------
        AWS ALB Ingress controller
          Release:    v1.1.4
          Build:      git-0db46039
          Repository: https://github.com/kubernetes-sigs/aws-alb-ingress-controller
-------------------------------------------------------------------------------
Bash
Deploy sample application
Now let’s deploy a sample 2048 game into our Kubernetes cluster and use the Ingress resource to expose it to traffic.

Deploy 2048 game resources:

kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/2048/2048-namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/2048/2048-deployment.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/2048/2048-service.yaml
Bash
Deploy an Ingress resource for the 2048 game:

kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/2048/2048-ingress.yaml
Bash
After few seconds, verify that the Ingress resource is enabled:

$ kubectl get ingress/2048-ingress -n 2048-game
NAME           HOSTS   ADDRESS                                                                  PORTS   AGE
2048-ingress   *       DNS-Name-Of-Your-ALB   80      3m
Bash
Open a browser and copy-paste your DNS-Name-Of-Your-ALB and you should be able to access your newly deployed 2048 game – have fun!
