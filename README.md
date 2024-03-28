# kubestellar-blue-green-demo

[Blue-Green on AWS EKS with ALB](https://aws.amazon.com/blogs/containers/using-aws-load-balancer-controller-for-blue-green-deployment-canary-deployment-and-a-b-testing/)

get the kubeconfig

    eksctl utils write-kubeconfig --cluster=dev --kubeconfig=eks.kubeconfig --region ap-southeast-2

[ALB config for EKS](https://docs.aws.amazon.com/eks/latest/userguide/lbc-helm.html)

    ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

    curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.1/docs/install/iam_policy.json

    eksctl utils associate-iam-oidc-provider --region=ap-southeast-2 --cluster=dev --approve

    eksctl create iamserviceaccount \
        --cluster=dev \
        --namespace=kube-system \
        --name=aws-load-balancer-controller \
        --role-name AmazonEKSLoadBalancerControllerRole \
        --attach-policy-arn=arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
        --approve \
        --region ap-southeast-2

    KUBECONFIG=eks.kubeconfig helm repo add eks https://aws.github.io/eks-charts
    KUBECONFIG=eks.kubeconfig helm repo update eks

    KUBECONFIG=eks.kubeconfig helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
        -n kube-system \
        --set clusterName=my-cluster \
        --set serviceAccount.create=false \
        --set serviceAccount.name=aws-load-balancer-controller 

    KUBECONFIG=eks.kubeconfig kubectl get deployment -n kube-system aws-load-balancer-controller


NEXT STEPS:
    Install KubeStellar
    git clone https://github.com/paulbouwer/hello-kubernetes.git
    
    KUBECONFIG=eks.kubeconfig helm --kube-context wds1 install --create-namespace --namespace hello-kubernetes v1 \
        ./hello-kubernetes/deploy/helm/hello-kubernetes \
        --set message="You are reaching hello-kubernetes version 1" \
        --set ingress.configured=true \
        --set service.type="ClusterIP"

    KUBECONFIG=eks.kubeconfig helm --kube-context wds1 install --create-namespace --namespace hello-kubernetes v2 \
        ./hello-kubernetes/deploy/helm/hello-kubernetes \
        --set message="You are reaching hello-kubernetes version 2" \
        --set ingress.configured=true \
        --set service.type="ClusterIP"


    KUBECONFIG=eks.kubeconfig kubectl --context WDS0 apply -f - <<EOF
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: "hello-kubernetes"
      namespace: "hello-kubernetes"
      annotations:
        kubernetes.io/ingress.class: alb
        alb.ingress.kubernetes.io/scheme: internet-facing
        alb.ingress.kubernetes.io/target-type: ip
        alb.ingress.kubernetes.io/actions.blue-green: |
          {
            "type":"forward",
            "forwardConfig":{
              "targetGroups":[
                {
                  "serviceName":"hello-kubernetes-v1",
                  "servicePort":"80",
                  "weight":100
                },
    EOF