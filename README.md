# kubestellar-blue-green-demo

[Blue-Green on AWS EKS with ALB](https://aws.amazon.com/blogs/containers/using-aws-load-balancer-controller-for-blue-green-deployment-canary-deployment-and-a-b-testing/)

get the kubeconfig

    eksctl utils write-kubeconfig --cluster=dev --kubeconfig=/path/to/custom/kubeconfig --region ap-southeast-2

[ALB config for EKS](https://docs.aws.amazon.com/eks/latest/userguide/lbc-helm.html)