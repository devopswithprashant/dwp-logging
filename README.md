# ELK Stack logging deployment via argoCD

### Steps:

#### Set cluster
aws eks update-kubeconfig --name dwp-platform-nonprod --region eu-west-1

#### Create logging namespace
Kubectl create namespace logging

#### Check if EBS CSI driver addon is installed
aws eks describe-addon \
  --cluster-name dwp-platform-nonprod \
  --addon-name aws-ebs-csi-driver \
  --region eu-west-1

#### If not installed, add it:
aws eks create-addon \
  --cluster-name dwp-platform-nonprod \
  --addon-name aws-ebs-csi-driver \
  --region eu-west-1


#### update gp2
kubectl delete storageclass gp2
Or 
kubectl patch storageclass gp2 \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'



# Then create gp2 storage class if missing:
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp2
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp2
  fsType: ext4
EOF







#### Install eksctl tool

Install eksctl tool: https://docs.aws.amazon.com/eks/latest/eksctl/installation.html





#### Fix — attach IAM role to the service account (IRSA). Run this:



1. Step 1 — associate OIDC provider (required for IRSA)
eksctl utils associate-iam-oidc-provider \
  --cluster dwp-platform-nonprod \
  --region eu-west-1 \
  --approve

2. Step 2 — verify OIDC is now associated
aws eks describe-cluster \
  --name dwp-platform-nonprod \
  --region eu-west-1 \
  --query "cluster.identity.oidc.issuer" \
  --output text


3. Step 3 — create IAM role and attach to the service account 
eksctl create iamserviceaccount \ 
  --name ebs-csi-controller-sa \ 
  --namespace kube-system \ 
  --cluster dwp-platform-nonprod \ 
  --region eu-west-1 \ 
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \ 
  --approve \ 
  --override-existing-serviceaccounts




#### Deploy ELK-Stack from argoCD UI

Fill in — Helm section
ArgoCD UI field
Scroll down to the Helm section. This appears automatically because ArgoCD detects your Chart.yaml.
Values Files:
1. values/elasticsearch.yaml
1. values/logstash.yaml
1. values/filebeat.yaml
1. values/kibana.yaml


#### Update Kibana service type

kubectl patch svc elk-logging-kibana -n logging-stack \
  -p '{"spec":{"type":"LoadBalancer"}}'

