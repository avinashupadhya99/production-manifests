# Manifests for deploying `Production` application on EKS

```bash
eksctl create cluster -f eks-cluster-config.yaml
```

```bash
alias k=kubectl
```

```bash
k apply -f database-secret.yaml
k apply -f produce-service/deployment.yaml
k apply -f produce-service/service.yaml
k apply -f package-service/deployment.yaml
k apply -f package-service/service.yaml
```

https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html

```bash
aws eks describe-cluster --name production-cluster --query "cluster.identity.oidc.issuer" --output text
```

```bash
aws iam list-open-id-connect-providers | grep XXXXXXXXXXXXXXXXXXXXX
```

```bash
eksctl utils associate-iam-oidc-provider --cluster production-cluster --approve
```


```bash
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

```bash
eksctl create iamserviceaccount \
  --cluster=production-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name "AmazonEKSProductionLoadBalancerControllerRole" \
  --attach-policy-arn=arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=production-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set image.repository=602401143452.dkr.ecr.ap-south-1.amazonaws.com/amazon/aws-load-balancer-controller \
  --set region=ap-south-1 \
  --set vpcId=$VPC_ID
```

https://aws.amazon.com/premiumsupport/knowledge-center/eks-set-up-externaldns/

```bash
aws iam create-policy \
    --policy-name Route53ExternalDNSIAMPolicy \
    --policy-document file://route53_iam_policy.json
```

```bash
eksctl create iamserviceaccount \
  --name external-dns \
  --namespace kube-system \
  --cluster production-cluster  \
  --attach-policy-arn arn:aws:iam::$ACCOUNT_ID:policy/Route53ExternalDNSIAMPolicy \
  --approve
```

```bash
k apply -f external-dns.yaml
``` 

```bash
k apply -f ingress.yaml
```

### Logging

https://aws.amazon.com/blogs/containers/fluent-bit-for-amazon-eks-on-aws-fargate-is-here/

```bash
k apply -f fluentbit-config.yaml
```

```bash
aws iam create-policy \
        --policy-name FluentBitEKSFargate \
        --policy-document file://permissions.json 
```

```
aws iam attach-role-policy \
        --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/FluentBitEKSFargate \
        --role-name AmazonEKSFargatePodExecutionRole
```