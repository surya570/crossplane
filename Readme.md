More detail at https://www.fahdmirza.com/2023/01/step-by-step-installation-of-crossplane.html

If you want to create your cloud resources such as AWS EC2, S3 bucket etc from within Kubernetes, then you need to use Crossplane. Its an open source project. Following is step  by step instructions to install crossplane on AWS EKS.

-- Make sure kubectl version is v1.23 and helm version is v3.8.2

-- All files which are being used in this code are available at github.

Step 1: Create EKS cluster

Step 2: Run following commands:

For IAM Setup:

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)



# A permissions boundary is an advanced feature for using a managed policy to set the maximum permissions that an identity-based policy can grant to an IAM entity. permission-boundary.json file is available in github repo here.

sed -i.bak "s/ACCOUNT_ID/${ACCOUNT_ID}/g" permission-boundary.json



aws iam create-policy \

    --policy-name crossplaneBoundary \

    --policy-document file://permission-boundary.json



# Amazon EKS supports using OpenID Connect (OIDC) identity providers as a method to authenticate users to your cluster. crossplane-ssp is my cluster's name. You can use your own.

OIDC_PROVIDER=$(aws eks describe-cluster --name crossplane-ssp --query "cluster.identity.oidc.issuer" --output text | sed -e "s/^https:\/\///")



PERMISSION_BOUNDARY_ARN="arn:aws:iam::${ACCOUNT_ID}:policy/crossplaneBoundary"



read -r -d '' TRUST_RELATIONSHIP <<EOF

{

  "Version": "2012-10-17",

  "Statement": [

    {

      "Effect": "Allow",

      "Principal": {

        "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/${OIDC_PROVIDER}"

      },

      "Action": "sts:AssumeRoleWithWebIdentity",

      "Condition": {

        "StringLike": {

          "${OIDC_PROVIDER}:sub": "system:serviceaccount:crossplane-system:provider-*"

        }

      }

    }

  ]

}

EOF

echo "${TRUST_RELATIONSHIP}" > trust.json



# IAM role for provider-aws

aws iam create-role --role-name crossplane-provider-aws --assume-role-policy-document file://trust.json --description "IAM role for provider-aws" --permissions-boundary ${PERMISSION_BOUNDARY_ARN}



aws iam attach-role-policy --role-name crossplane-provider-aws --policy-arn=arn:aws:iam::aws:policy/AdministratorAccess



# Annotate the service account to use IRSA.

sed -i.bak "s/ACCOUNT_ID/${ACCOUNT_ID}/g" aws-provider.yaml



# Install Crossplane

kubectl create namespace crossplane-system



helm repo add crossplane-stable https://charts.crossplane.io/stable

helm repo update



helm install crossplane --namespace crossplane-system --version 1.10.1 crossplane-stable/crossplane



# wait for the provider CRD to be ready.

kubectl wait --for condition=established --timeout=300s crd/providers.pkg.crossplane.io

kubectl apply -f aws-provider.yaml



# wait for the AWS provider CRD to be ready.

kubectl wait --for condition=established --timeout=300s crd/providerconfigs.aws.crossplane.io

kubectl apply -f aws-provider-config.yaml



#create resources



kubectl apply -f ec2.yaml

kubectl get instance

kubectl describe instance



kubectl apply -f s3.yaml

kubectl get Bucket

kubectl describe Bucket

