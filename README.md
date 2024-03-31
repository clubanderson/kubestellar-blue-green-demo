# kubestellar-blue-green-demo

[Blue-Green on AWS EKS with ALB](https://aws.amazon.com/blogs/containers/using-aws-load-balancer-controller-for-blue-green-deployment-canary-deployment-and-a-b-testing/)

    deploy with "vpc and more"
    https://clubanderson.medium.com/how-to-install-kubestellar-on-a-collection-of-aws-eks-managed-clusters-aa1615e671a0

get the kubeconfigs

    eksctl utils write-kubeconfig --cluster=bg-wec1 --kubeconfig=eks.kubeconfig --region us-east-1;eksctl utils write-kubeconfig --cluster=bg-wec2 --kubeconfig=eks.kubeconfig --region us-east-1; eksctl utils write-kubeconfig --cluster=bg-core --kubeconfig=eks.kubeconfig --region us-east-1

    NOTE: write the hub to the kubeconfig last so that your context is defaulted to the hub to start this exercise

    Stacker: Install KubeStellar (WDS0 hosted control plane, WDS1 pointing at aws-wec1 and WDS2 pointing at aws-wec2)
    add bindingpolicy for WDS1 to get 'app.kubernetes.io/part-of=v1' and WDS2 to get 'app.kubernetes.io/part-of=v2'

    install ingress-nginx (Network Load Balancer) FIRST to support kubestellar (https://clubanderson.medium.com/how-to-install-kubestellar-on-a-collection-of-aws-eks-managed-clusters-aa1615e671a0)
        kubectl --context hub-eks apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/aws/deploy.yaml

        add ssl passthrough:
        kubectl --context hub-eks edit deployment.apps/ingress-nginx-controller -n ingress-nginx
        add '- --enable-ssl-passthrough'

        add more ports for ks:       
        kubectl --context hub-eks edit service ingress-nginx-controller -n ingress-nginx

        - name: proxied-tcp-9443
          nodePort: 31345
          port: 9443
          protocol: TCP
          targetPort: 443
        - name: proxied-tcp-9080
          nodePort: 31226
          port: 9080
          protocol: TCP
          targetPort: 80

        check for a network load balancer in https://console.aws.amazon.com/ec2/home?#LoadBalancers
        test https://imbs1.kubestellar.org:9443, https://wds1.kubestellar.org:9443, https://wds2.kubestellar.org:9443


[ALB config for EKS](https://docs.aws.amazon.com/eks/latest/userguide/lbc-helm.html)

    eksctl utils associate-iam-oidc-provider --region=us-east-1 --cluster=bg-core --approve

    curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.1/docs/install/iam_policy.json

    aws iam create-policy \
        --policy-name AWSLoadBalancerControllerIAMPolicy \
        --policy-document file://iam_policy.json

    ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
    echo $ACCOUNT_ID

    eksctl create iamserviceaccount \
        --cluster=bg-core \
        --namespace=kube-system \
        --name=aws-load-balancer-controller \
        --role-name AmazonEKSLoadBalancerControllerRole \
        --attach-policy-arn=arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
        --approve \
        --region us-east-1

    eksctl create iamserviceaccount \
        --cluster=bg-wec1 \
        --namespace=kube-system \
        --name=aws-load-balancer-controller-wec1 \
        --role-name AmazonEKSLoadBalancerControllerRole \
        --attach-policy-arn=arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
        --approve \
        --region us-east-1

    eksctl create iamserviceaccount \
        --cluster=bg-wec2 \
        --namespace=kube-system \
        --name=aws-load-balancer-controller-wec2 \
        --role-name AmazonEKSLoadBalancerControllerRole \
        --attach-policy-arn=arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
        --approve \
        --region us-east-1

    NOTE: if you somehow goof up the creation of the iamserviceaccount - use 'eksctl delete iamserviceaccount --cluster=bg-core --name=aws-load-balancer-controller --namespace=kube-system' to remove it.

    KUBECONFIG=eks.kubeconfig helm repo add eks https://aws.github.io/eks-charts
    KUBECONFIG=eks.kubeconfig helm repo update eks

    (on all clusters?)
    KUBECONFIG=eks.kubeconfig helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
        -n kube-system \
        --set clusterName=bg-core \
        --set serviceAccount.create=false \
        --set serviceAccount.name=aws-load-balancer-controller 

    KUBECONFIG=eks.kubeconfig kubectl get deployment -n kube-system aws-load-balancer-controller

    IMPORTANT: you MUST TAG your public subnets (https://repost.aws/knowledge-center/eks-vpc-subnet-discovery) with
        'kubernetes.io/role/elb'	with a value of '1'

    deploy a test app on the core to see if the alb is created in AWS EC2
        kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.1/docs/examples/2048/2048_full.yaml

    check for an application load balancer in https://console.aws.amazon.com/ec2/home?#LoadBalancers

    you should see an address show up in a few minutes under the ingress listing
        kubectl get ingress/ingress-2048 -n game-2048

    put the address in your browser - do not be surprised if the site does not show for 5 minutes or more. takes time for the service to reconcile with the alb

    check logs if ALB does not create
        kubectl logs -f -n kube-system -l app.kubernetes.io/instance=aws-load-balancer-controller


NEXT STEPS:


    apiVersion: control.kubestellar.io/v1alpha1
    kind: BindingPolicy
    metadata:
      name: bg-wec1-v1-bindingpolicy
    spec:
      wantSingletonReportedState: true
      clusterSelectors:
      - matchLabels:
          name: bg-wec1
      downsync:
      - objectSelectors:
        - matchLabels: 
            app.kubernetes.io/instance: v1


    apiVersion: control.kubestellar.io/v1alpha1
    kind: BindingPolicy
    metadata:
      name: bg-wec2-v2-bindingpolicy
    spec:
      wantSingletonReportedState: true
      clusterSelectors:
      - matchLabels:
          name: bg-wec2
      downsync:
      - objectSelectors:
        - matchLabels: 
            app.kubernetes.io/instance: v2

    git clone https://github.com/paulbouwer/hello-kubernetes.git
    
    KUBECONFIG=eks.kubeconfig helm --kube-context wds1 install --create-namespace --namespace hello-kubernetes v1 \
        ./hello-kubernetes/deploy/helm/hello-kubernetes \
        --set message="You are reaching hello-kubernetes version 1" \
        --set ingress.configured=true \
        --set service.type="ClusterIP" \

    KUBECONFIG=eks.kubeconfig helm --kube-context wds2 install --create-namespace --namespace hello-kubernetes v2 \
        ./hello-kubernetes/deploy/helm/hello-kubernetes \
        --set message="You are reaching hello-kubernetes version 2" \
        --set ingress.configured=true \
        --set service.type="ClusterIP" \



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
                {
                  "serviceName":"hello-kubernetes-v2",
                  "servicePort":"80",
                  "weight":0
                }
              ]
            }
          }
      labels:
        app: hello-kubernetes
    spec:
      rules:
        - http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: blue-green
                    port:
                      name: use-annotation
    EOF

    ingress.networking.k8s.io/hello-kubernetes configured

    ELB_URL=$(kubectl get ingress -n hello-kubernetes -o=jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}')
    while true; do curl -s $ELB_URL | grep version; sleep 1; done
  
        You are reaching hello-kubernetes version 1
        You are reaching hello-kubernetes version 1
        You are reaching hello-kubernetes version 1


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
                  "weight":50
                },
                {
                  "serviceName":"hello-kubernetes-v2",
                  "servicePort":"80",
                  "weight":50
                }
              ]
            }
          }
      labels:
        app: hello-kubernetes
    spec:
      rules:
        - http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: blue-green
                    port:
                      name: use-annotation
    EOF

    ingress.networking.k8s.io/hello-kubernetes configured

    while true; do curl -s k8s-hellokub-hellokub-1c21b68bea-597504338.us-east-1.elb.amazonaws.com | grep version; sleep 1; done

        You are reaching hello-kubernetes version 2
        You are reaching hello-kubernetes version 2
        You are reaching hello-kubernetes version 2

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
                  "weight":0
                },
                {
                  "serviceName":"hello-kubernetes-v2",
                  "servicePort":"80",
                  "weight":100
                }
              ]
            }
          }
      labels:
        app: hello-kubernetes
    spec:
      rules:
        - http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: blue-green
                    port:
                      name: use-annotation
    EOF

    ingress.networking.k8s.io/hello-kubernetes configured

    while true; do curl -s k8s-hellokub-hellokub-1c21b68bea-597504338.us-east-1.elb.amazonaws.com | grep version; sleep 1; done
    
        You are reaching hello-kubernetes version 1
        You are reaching hello-kubernetes version 2
        You are reaching hello-kubernetes version 1
        You are reaching hello-kubernetes version 1
        You are reaching hello-kubernetes version 1


    
tear down:

    KUBECONFIG=eks.kubeconfig kubectl --context WDS0 delete ing hello-kubernetes
    eksctl delete cluster --name dev --region us-east-1