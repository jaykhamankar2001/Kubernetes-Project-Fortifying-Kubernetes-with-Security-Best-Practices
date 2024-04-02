# Kubernetes-Project-Fortifying-Kubernetes-with-Security-Best-Practices
***Kubernetes Project:  Fortifying Kubernetes with Security Best Practices***
In this Kubernetes project, I have implemented several security measures listed below for practice purposes

Topics covered in this project:
1) Network Policies
2) Nginx Ingress with Cert-Manager integration
3) Kyverno policy enforcement
4) Security context
5) RBAC
6) Sealed secrets


Apply all Deployment files
```
kubectl apply -f frontend-deployment.yaml
kubectl apply -f api-deployment.yaml
kubectl apply -f mongo-statefulset.yaml
```
Apply all Service files
```
kubectl apply -f frontend-service.yaml
kubectl apply -f api-service.yaml
kubectl apply -f mongo-service.yaml
```
Install Ingress
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```
Apply all Ingress files
```
kubectl apply -f frontend-ingress.yaml
kubectl apply -f api-ingress.yaml
```

### Install cert-manager

Cert-manager is a Kubernetes native certificate management controller that automates the management and issuance of TLS certificates. It integrates with Let's Encrypt, a free and automated Certificate Authority, to provide TLS certificates for your applications running in Kubernetes clusters

```
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.7.1/cert-manager.yaml

```
For cert-manager, we need to create a **cluster issuer** which  is a Kubernetes resource used to manage the issuance and renewal of TLS certificates at the cluster level

```
kubectl apply -f staging-issuer.yaml
```

### Sealed Secrets
Secrets in kubernetes are base64 encoded not encrypted so anyone can decode it if stored in repository this problem is solved by sealed secrets 
Sealed Secrets in Kubernetes are a way to securely manage and store encrypted secrets. They are essentially encrypted versions of Kubernetes Secrets that can be created by anyone but can only be decrypted by the controller running in the target cluster. The key feature of Sealed Secrets is that the encrypted data can be safely stored in a code repository.



- Create secret for mongo-db
```
kubectl apply -f mongo-secret.yaml
```
- Insatll Sealed Secrets
```
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets

helm install sealed-secrets -n kube-system --set-string fullnameOverride=sealed-secrets-controller sealed-secrets/sealed-secrets
```
- Install kubeseal CLI
```
KUBESEAL_VERSION='' # Set this to, for example, KUBESEAL_VERSION='0.23.0'
wget "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION:?}/kubeseal-${KUBESEAL_VERSION:?}-linux-amd64.tar.gz"
tar -xvzf kubeseal-${KUBESEAL_VERSION:?}-linux-amd64.tar.gz kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```
- Encrypting secret
```
kubeseal --controller-name sealed-secrets-controller --controller-namespace kube-system --format yaml < mongo-secret.yaml > mongo-sealed-secret.yaml
```
- Applying Sealed secret in Kubernetes Cluster
```
kubectl apply -f mongo-sealed-secret.yaml
```

### Kynvero
Kyverno is an open-source Kubernetes-native policy engine. It allows you to enforce policies for validating, mutating, and generating Kubernetes resources at runtime. With Kyverno, you can define policies using YAML files, which are then applied to your Kubernetes cluster to ensure compliance with your organization's security

- Install Kyverno
```
kubectl create -f https://github.com/kyverno/kyverno/releases/download/v1.10.0/install.yaml
```
- Apply kyverno policy 
  This policy enforces security context settings at the container level, including runAsNonRoot, readOnlyRootFilesystem, and privileged
  ```
  kubectl apply -f kyverno-enforce-security-context.yaml
  ```
 ![image](https://github.com/jaykhamankar2001/Kubernetes-Project-Fortifying-Kubernetes-with-Security-Best-Practices/assets/74448020/7dc1727e-d686-4dbc-873c-9699c945ce6b)

  In this image, you can see my attempt to install frontend-deployment.yaml without runAsNonRoot, Kyverno showed that I need to include runAsNonRoot

  
### Apply Network Policies

Network policies in Kubernetes provide a way to control traffic between pods and external network endpoints
```
kubectl apply -f network-policy-deny-all.yaml
network-policy-mongo-to-api.yaml
```

### Implenment RBAC

- Add IAM User to EKS Cluster
Create ClusterRole with read-only access to the Kubernetes cluster and bind it to the reader group via ClusterRoleBinding.

- Apply RBAC policies 
```
kubectl apply -f cluster-role-n-binding.yaml
```
- Create IAM policy to let users view nodes and workloads for all clusters in the AWS Management Console. Give it a name AmazonEKSViewNodesAndWorkloadsPolicy.
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "eks:DescribeNodegroup",
                "eks:ListNodegroups",
                "eks:DescribeCluster",
                "eks:ListClusters",
                "eks:AccessKubernetesApi",
                "ssm:GetParameter",
                "eks:ListUpdates",
                "eks:ListFargateProfiles"
            ],
            "Resource": "*"
        }
    ]
}
```
- We can attach the IAM policy directly to the IAM user or follow the best practice and create an IAM group first. Let's call it developers and attach AmazonEKSViewNodesAndWorkloadsPolicy IAM policy.

- Then, we need an IAM user. Let's create one and call it developer. We're going to place it in developers IAM grop with read only access. Don't forget to download credentials, we will use them to configure aws cli.

- Now, we need to create local aws profile using developer's user credentials. To do that simplly add --profile flag to aws configure command.

```
aws configure --profile developer
```
- To map IAM user with Kubernetes RBAC system, we need to modify aws-auth configmap. Open the config map and add arn of the IAM user under mapUsers key.

```

 kubectl edit -n kube-system configmap/aws-auth
 ```
 ```
 mapUsers: |
    - userarn: arn:aws:iam::614754387669:user/developer
      username: developer
      groups: 
      - reader
 ```
 - Now, we need to switch to the developer user. We need to update Kubernetes context using the developer profile. Don't forget to update region and the cluster name.
 ```
 aws eks update-kubeconfig \
  --region us-east-1 \
  --name demo \
  --profile developer
```
You can check if RBAC has been successfully implemented or not
```
kubectl auth can-i get pods
```
