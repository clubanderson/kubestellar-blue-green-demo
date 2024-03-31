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


OLD APPROACH
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
        --role-name AmazonEKSLoadBalancerControllerRoleWec2 \
        --attach-policy-arn=arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
        --approve \
        --region us-east-1

    eksctl create iamserviceaccount \
        --cluster=bg-wec2 \
        --namespace=kube-system \
        --name=aws-load-balancer-controller-wec2 \
        --role-name AmazonEKSLoadBalancerControllerRoleWec2 \
        --attach-policy-arn=arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
        --approve \
        --region us-east-1

    NOTE: if you somehow goof up the creation of the iamserviceaccount - use 'eksctl delete iamserviceaccount --cluster=bg-core --name=aws-load-balancer-controller --namespace=kube-system' to remove it.

    KUBECONFIG=eks.kubeconfig helm repo add eks https://aws.github.io/eks-charts
    KUBECONFIG=eks.kubeconfig helm repo update eks

    KUBECONFIG=eks.kubeconfig helm --kube-context bg-core install aws-load-balancer-controller eks/aws-load-balancer-controller \
        -n kube-system \
        --set clusterName=bg-core \
        --set serviceAccount.create=false \
        --set serviceAccount.name=aws-load-balancer-controller 

    KUBECONFIG=eks.kubeconfig helm --kube-context bg-wec1 install aws-load-balancer-controller eks/aws-load-balancer-controller \
        -n kube-system \
        --set clusterName=bg-wec1 \
        --set serviceAccount.create=false \
        --set serviceAccount.name=aws-load-balancer-controller-wec1

    KUBECONFIG=eks.kubeconfig helm --kube-context bg-wec2 install aws-load-balancer-controller eks/aws-load-balancer-controller \
        -n kube-system \
        --set clusterName=bg-wec2 \
        --set serviceAccount.create=false \
        --set serviceAccount.name=aws-load-balancer-controller-wec2 

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


did not work - target wanted a service name (endpoint) in the cluster where the target
    
        git clone https://github.com/clubanderson/kubestellar-blue-green-demo


1) install dns policy

    aws iam create-policy --policy-name "AllowExternalDNSUpdates" --policy-document file://policy.json
    POLICY_ARN=$(aws iam list-policies --query 'Policies[?PolicyName==`AllowExternalDNSUpdates`].Arn' --output text)

2) create service account on each cluster
    eksctl create iamserviceaccount \
      --cluster bg-core \
      --name "external-dns" \
      --namespace "default" \
      --attach-policy-arn $POLICY_ARN \
      --approve

    eksctl create iamserviceaccount \
      --cluster bg-wec1 \
      --name "external-dns" \
      --namespace "default" \
      --attach-policy-arn $POLICY_ARN \
      --approve

    eksctl create iamserviceaccount \
      --cluster bg-wec2 \
      --name "external-dns" \
      --namespace "default" \
      --attach-policy-arn $POLICY_ARN \
      --approve

3) update domain filter in step2-external-dns deployment definition in step2-externaldns/v1/bg-wec1-external-dns-with-rbac.yml and step2-external-dns/v2/bg-wec2-external-dns-with-rbac.yml to match your domain name in route53

    - --domain-filter= your.domain.here

4) deploy external-dns with KubeStellar
    kubectl --context wds1 apply -f step2-external-dns/v1/  
    kubectl --context wds2 apply -f step2-external-dns/v2/

5) check that external-dns was applied to the remote clusters

    kubectl --context bg-wec1 --namespace=kube-system get pods -l "app.kubernetes.io/name=external-dns,app.kubernetes.io/instance=externaldns-release"
    kubectl --context bg-wec1 get all -n default | grep external-dns   

    kubectl --context bg-wec2 --namespace=kube-system get pods -l "app.kubernetes.io/name=external-dns,app.kubernetes.io/instance=externaldns-release"
    kubectl --context bg-wec2 get all -n default | grep external-dns   

6) update the external-dns hostname annotation in the service definition in step3-hello-kubernetes/v1/bg-wec1-hello-kubernetes.yml and step3-hello-kubernetes/v2/bg-wec2-hello-kubernetes.yml

  external-dns.alpha.kubernetes.io/hostname: hello-kube.your.domain.here

7) deploy hello-kubernetes application with KubeStellar
    kubectl --context wds1 apply -f step3-hello-kubernetes/v1/  
    kubectl --context wds2 apply -f step3-hello-kubernetes/v2/

8) check if hello-kubernetes was applied the remote clusters

    kubectl --context bg-wec1 get all -n hello-kubernetes
    kubectl --context bg-wec2 get all -n hello-kubernetes

9) check route53 for dns updates
    if you do not see updates, check logs at 

    kubectl --context bg-wec1 logs deployment/external-dns  
    kubectl --context bg-wec2 logs deployment/external-dns  

10) try the url you specified in the external-dns hostname annotation in the hello-kubernetes service definition (be sure to put in a nameserver from your domains NS record)
    for i in {1..500}; do domain=$(dig hello-kube.your.domain.here. TXT @your.domains.ns.record. +short); echo -e  "$domain" >> RecursiveResolver_results.txt; done
    awk ' " " ' RecursiveResolver_results.txt | sort | uniq -c

    source: https://repost.aws/knowledge-center/route-53-fix-dns-weighted-routing-issue#

    you should see 50% of traffic going to each version/service

11) update the external-dns aws-weight (shift more traffic to v2 of hello-kubernetes) annotation in the service definition in 

    edit step3-hello-kubernetes/v1/bg-wec1-hello-kubernetes.yml 

      external-dns.alpha.kubernetes.io/aws-weight: "25"

    edit step3-hello-kubernetes/v2/bg-wec2-hello-kubernetes.yml

      external-dns.alpha.kubernetes.io/aws-weight: "75"

12) re-deploy hello-kubernetes application with KubeStellar

    kubectl --context wds1 apply -f step3-hello-kubernetes/v1/  
    kubectl --context wds2 apply -f step3-hello-kubernetes/v2/

13) again, try the url you specified in the external-dns hostname annotation in the hello-kubernetes service definition (be sure to put in a nameserver from your domains NS record)
    for i in {1..500}; do domain=$(dig hello-kube.your.domain.here. TXT @your.domains.ns.record. +short); echo -e  "$domain" >> RecursiveResolver_results.txt; done
    awk ' " " ' RecursiveResolver_results.txt | sort | uniq -c

    source: https://repost.aws/knowledge-center/route-53-fix-dns-weighted-routing-issue#
    you should see 25% of traffic going to each v1 version of the hello-kubernetes service and 75% of traffic going to the v2 version of the hello-kubernetes service

TODO: show traffic graph


sources:
https://ddulic.dev/external-dns-migrate-services-between-k8s-clusters


tear down:

    KUBECONFIG=eks.kubeconfig kubectl --context WDS0 delete ing hello-kubernetes
    eksctl delete cluster --name dev --region us-east-1