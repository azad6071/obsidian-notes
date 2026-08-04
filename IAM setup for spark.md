
# tmv-orchestrator-service Architecture

  

## Credential Flow & Responsibilities

  

```

┌─────────────────────────────────────────────────────────────────────┐

│ Jenkins Pipeline │

│ (Builds & uploads Spark artifacts to S3) │

│ │

│ Credentials: EC2 Instance Profile or Jenkins AWS credentials │

│ Access: s3://tmv-dev-data/artifacts/* │

└───────────────────────────────┬─────────────────────────────────────┘

│

│ uploads artifacts

▼

┌───────────────────────┐

│ S3 Bucket │

│ tmv-dev-data │

│ /artifacts/ │

│ - pipeline.zip │

│ - run_extraction.py │

│ - run_disease.py │

└───────────────────────┘

▲

│ EMR reads artifacts

│

┌───────────────────────────────┼─────────────────────────────────────┐

│ EKS Cluster│ │

│ │ │

│ ┌────────────────────────────┴──────────────────────┐ │

│ │ tmv-orchestrator-service Pod │ │

│ │ │ │

│ │ 1. Claims batch from database │ │

│ │ 2. Submits EMR step with S3 artifact paths ──────┼──────┐ │

│ │ 3. Polls EMR step status │ │ │

│ │ │ │ │

│ │ Credentials: IRSA (ServiceAccount + IAM Role) │ │ │

│ │ Permissions Needed: │ │ │

│ │ ✓ elasticmapreduce:AddJobFlowSteps │ │ │

│ │ ✓ elasticmapreduce:DescribeStep │ │ │

│ │ ✓ elasticmapreduce:ListSteps │ │ │

│ │ ✗ S3 access NOT needed (only passes paths) │ │ │

│ └────────────────────────────────────────────────────┘ │ │

│ │ │

└───────────────────────────────────────────────────────────────┼───────┘

│

│ boto3.client('emr').add_job_flow_steps()

▼

┌───────────────────────┐

│ AWS EMR Cluster │

│ j-NFXTCUC5YKRC │

│ │

│ Credentials: │

│ EMR_EC2_DefaultRole │

│ │

│ Permissions: │

│ ✓ S3 Read/Write │

│ ✓ DynamoDB │

│ ✓ etc. │

└───────────────────────┘

│

│ reads/writes

▼

┌───────────────────────┐

│ S3 / RDS / etc │

│ (Data sources) │

└───────────────────────┘

```

  

## Key Points

  

### 1. Jenkins Pipeline

- **Runs on**: Jenkins server/agent (EC2 or on-premises)

- **Credentials**: AWS credentials configured on Jenkins (EC2 instance profile, stored credentials, etc.)

- **Purpose**: Build and upload Spark job artifacts to S3

- **Access**: Can write to `s3://tmv-dev-data/artifacts/`

  

### 2. tmv-orchestrator-service (Kubernetes Pod)

- **Runs on**: EKS cluster

- **Credentials**: IRSA (IAM Roles for Service Accounts) - needs to be configured

- **Purpose**: Submit and monitor EMR jobs via boto3

- **Access Needed**:

- ✅ EMR API calls (AddJobFlowSteps, DescribeStep, ListSteps)

- ❌ S3 access NOT required (only passes S3 paths as strings to EMR)

  

### 3. EMR Cluster

- **Runs on**: EC2 instances managed by EMR

- **Credentials**: EMR service roles (EMR_EC2_DefaultRole, EMR_DefaultRole)

- **Purpose**: Execute Spark jobs

- **Access Needed**:

- ✅ S3 read/write for artifacts and data

- ✅ RDS/DynamoDB/etc. for data processing

- ✅ CloudWatch for logs

  

## Why "Unable to locate credentials" Error?

  

The orchestrator service pods in Kubernetes start with **NO AWS credentials**. They need IRSA configured:

  

1. **ServiceAccount** with annotation `eks.amazonaws.com/role-arn`

2. **IAM Role** that trusts the EKS OIDC provider

3. **IAM Policy** attached to the role granting EMR permissions

  

Without IRSA, boto3 in the pod has no way to get credentials, hence the error.

  

## What You Need to Do

  

Follow the setup in `iam/README.md` to:

1. Create the IAM role with EMR permissions

2. Configure the trust relationship with your EKS OIDC provider

3. Deploy the Helm chart (already configured with the service account annotation)

  

The fact that Jenkins can access S3 doesn't help the orchestrator pods - they need their own credential mechanism (IRSA).



# AWS IAM Setup for tmv-orchestrator-service

  

This service requires AWS credentials to interact with EMR and S3. We use **IAM Roles for Service Accounts (IRSA)** for secure, credential-less authentication.

  

## Prerequisites

  

1. EKS cluster with OIDC provider enabled

2. Get your cluster's OIDC provider URL:

```bash

aws eks describe-cluster --name <cluster-name> --query "cluster.identity.oidc.issuer" --output text

# Example output: https://oidc.eks.us-east-1.amazonaws.com/id/XXXXXXXXXXXXX

```

  

## Step 1: Create IAM Policy

  

Create an IAM policy with the required permissions:

  

```json

{

"Version": "2012-10-17",

"Statement": [

{

"Effect": "Allow",

"Action": [

"elasticmapreduce:AddJobFlowSteps",

"elasticmapreduce:DescribeStep",

"elasticmapreduce:ListSteps"

],

"Resource": [

"arn:aws:elasticmapreduce:us-east-1:744958734165:cluster/j-NFXTCUC5YKRC"

]

},

{

"Effect": "Allow",

"Action": [

"s3:GetObject",

"s3:ListBucket"

],

"Resource": [

"arn:aws:s3:::tmv-dev-data",

"arn:aws:s3:::tmv-dev-data/*"

]

}

]

}

```

  

Save this to a file `tmv-orchestrator-policy.json` and create the policy:

  

```bash

aws iam create-policy \

--policy-name tmv-orchestrator-service-policy \

--policy-document file://tmv-orchestrator-policy.json

```

  

Note the policy ARN from the output.

  

## Step 2: Create IAM Role with Trust Policy

  

Replace `<OIDC_PROVIDER>` with your cluster's OIDC provider ID (the part after `/id/`):

  

```json

{

"Version": "2012-10-17",

"Statement": [

{

"Effect": "Allow",

"Principal": {

"Federated": "arn:aws:iam::744958734165:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/<OIDC_PROVIDER>"

},

"Action": "sts:AssumeRoleWithWebIdentity",

"Condition": {

"StringEquals": {

"oidc.eks.us-east-1.amazonaws.com/id/<OIDC_PROVIDER>:sub": "system:serviceaccount:default:tmv-orchestrator-service",

"oidc.eks.us-east-1.amazonaws.com/id/<OIDC_PROVIDER>:aud": "sts.amazonaws.com"

}

}

}

]

}

```

  

**Important:** Update the namespace in the `sub` field if you're not deploying to the `default` namespace. The format is:

```

system:serviceaccount:<NAMESPACE>:<SERVICE_ACCOUNT_NAME>

```

  

Save this to `trust-policy.json` and create the role:

  

```bash

aws iam create-role \

--role-name tmv-orchestrator-service-role \

--assume-role-policy-document file://trust-policy.json

```

  

## Step 3: Attach Policy to Role

  

```bash

aws iam attach-role-policy \

--role-name tmv-orchestrator-service-role \

--policy-arn arn:aws:iam::744958734165:policy/tmv-orchestrator-service-policy

```

  

## Step 4: Update values.yaml

  

The `values.yaml` has already been updated with the role ARN:

  

```yaml

serviceAccount:

create: true

name: ""

annotations:

eks.amazonaws.com/role-arn: "arn:aws:iam::744958734165:role/tmv-orchestrator-service-role"

```

  

## Step 5: Deploy

  

Deploy the updated Helm chart:

  

```bash

helm upgrade --install tmv-orchestrator-service . \

-f values.yaml \

--namespace <your-namespace>

```

  

## Verification

  

After deployment, verify the service account has the annotation:

  

```bash

kubectl get sa tmv-orchestrator-service -o yaml

```

  

You should see:

```yaml

metadata:

annotations:

eks.amazonaws.com/role-arn: arn:aws:iam::744958734165:role/tmv-orchestrator-service-role

```

  

Check that the pod has the AWS environment variables injected:

  

```bash

kubectl describe pod <pod-name> | grep AWS

```

  

You should see:

- `AWS_ROLE_ARN`

- `AWS_WEB_IDENTITY_TOKEN_FILE`

  

## Troubleshooting

  

If you still get "Unable to locate credentials":

  

1. Check the service account annotation:

```bash

kubectl get sa tmv-orchestrator-service -o jsonpath='{.metadata.annotations}'

```

  

2. Check the pod's environment variables:

```bash

kubectl exec <pod-name> -- env | grep AWS

```

  

3. Verify the IAM role trust policy allows your specific service account

  

4. Check EKS OIDC provider is configured correctly:

```bash

aws eks describe-cluster --name <cluster-name> --query "cluster.identity.oidc"

```

  

5. Check pod logs for more detailed AWS error messages:

```bash

kubectl logs <pod-name>

```


# Quick Start Guide

  

This guide gets you from the current broken state to a working deployment in ~10 minutes.

  

## Prerequisites

  

- `kubectl` configured for your EKS cluster

- `aws` CLI with admin permissions

- Access to deploy the Helm chart

  

## Step 1: Get Your EKS OIDC Provider (1 min)

  

```bash

CLUSTER_NAME="<your-eks-cluster-name>" # e.g., "hilabs-eks-cluster"

  

# Get the OIDC provider URL

OIDC_URL=$(aws eks describe-cluster --name $CLUSTER_NAME --query "cluster.identity.oidc.issuer" --output text)

  

# Extract just the ID (everything after /id/)

OIDC_ID=$(echo $OIDC_URL | sed 's|https://oidc.eks.us-east-1.amazonaws.com/id/||')

  

echo "OIDC Provider ID: $OIDC_ID"

```

  

## Step 2: Create IAM Policy (1 min)

  

```bash

cd tmv-helm-charts/tmv-orchestrator-service/iam

  

aws iam create-policy \

--policy-name tmv-orchestrator-service-policy \

--policy-document file://policy.json \

--description "Allows tmv-orchestrator-service to submit and monitor EMR jobs"

  

# Save the ARN for later

POLICY_ARN=$(aws iam list-policies --query "Policies[?PolicyName=='tmv-orchestrator-service-policy'].Arn" --output text)

echo "Policy ARN: $POLICY_ARN"

```

  

## Step 3: Create Trust Policy (2 min)

  

```bash

NAMESPACE="default" # Change if deploying to a different namespace

  

# Create trust policy from template

cat trust-policy-template.json | \

sed "s/<REPLACE_WITH_OIDC_PROVIDER_ID>/$OIDC_ID/g" | \

sed "s/<REPLACE_WITH_NAMESPACE>/$NAMESPACE/g" > trust-policy.json

  

# Verify it looks correct

cat trust-policy.json

```

  

## Step 4: Create IAM Role (1 min)

  

```bash

aws iam create-role \

--role-name tmv-orchestrator-service-role \

--assume-role-policy-document file://trust-policy.json \

--description "IRSA role for tmv-orchestrator-service to access EMR"

  

# Attach the policy to the role

aws iam attach-role-policy \

--role-name tmv-orchestrator-service-role \

--policy-arn $POLICY_ARN

  

echo "✅ IAM setup complete!"

```

  

## Step 5: Deploy Updated Application (3 min)

  

First, rebuild and deploy the application with the database fix:

  

```bash

cd ../../tmv-orchestrator-service

  

# Build new Docker image (update version as needed)

docker build -t 744958734165.dkr.ecr.us-east-1.amazonaws.com/tmv-orchestrator-service:20 .

  

# Push to ECR

aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 744958734165.dkr.ecr.us-east-1.amazonaws.com

docker push 744958734165.dkr.ecr.us-east-1.amazonaws.com/tmv-orchestrator-service:20

```

  

## Step 6: Deploy Helm Chart (2 min)

  

```bash

cd ../tmv-helm-charts/tmv-orchestrator-service

  

# Update the image tag in values.yaml to match your new image (e.g., "20")

# The serviceAccount section is already configured

  

# Deploy

helm upgrade --install tmv-orchestrator-service . \

-f values.yaml \

--namespace $NAMESPACE

  

# Wait for pod to be ready

kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=tmv-orchestrator-service -n $NAMESPACE --timeout=60s

```

  

## Step 7: Verify (2 min)

  

```bash

# 1. Check service account has the annotation

kubectl get sa tmv-orchestrator-service -n $NAMESPACE -o yaml | grep eks.amazonaws.com/role-arn

  

# 2. Check pod has AWS environment variables

POD_NAME=$(kubectl get pods -n $NAMESPACE -l app.kubernetes.io/name=tmv-orchestrator-service -o jsonpath='{.items[0].metadata.name}')

kubectl exec $POD_NAME -n $NAMESPACE -- env | grep AWS_

  

# You should see:

# AWS_ROLE_ARN=arn:aws:iam::744958734165:role/tmv-orchestrator-service-role

# AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token

  

# 3. Check pod logs

kubectl logs $POD_NAME -n $NAMESPACE --tail=50

  

# 4. Test the API

curl -X POST http://tmv-orchestrator.hilabs.internal/api/v1/spark/jobs \

-H "Content-Type: application-json" \

-d '{}'

  

# Expected: Success response with batch info, not "Unable to locate credentials"

```

  

## Troubleshooting

  

### Error: "Policy already exists"

```bash

# Use existing policy ARN

POLICY_ARN=$(aws iam list-policies --query "Policies[?PolicyName=='tmv-orchestrator-service-policy'].Arn" --output text)

```

  

### Error: "Role already exists"

```bash

# Delete and recreate

aws iam detach-role-policy --role-name tmv-orchestrator-service-role --policy-arn $POLICY_ARN

aws iam delete-role --role-name tmv-orchestrator-service-role

# Then retry Step 4

```

  

### Pod still has "Unable to locate credentials"

```bash

# Check the service account annotation

kubectl describe sa tmv-orchestrator-service -n $NAMESPACE

  

# Check if AWS mutating webhook is running

kubectl get mutatingwebhookconfigurations | grep pod-identity-webhook

  

# Restart the pod to re-inject credentials

kubectl delete pod $POD_NAME -n $NAMESPACE

```

  

### Database connection still failing

Make sure you deployed the application with the fixed `config.py` (image tag should be updated).

  

## Summary

  

You've now:

1. ✅ Fixed the database connection issue (proper URL parsing)

2. ✅ Fixed the AWS credentials issue (IRSA configured)

3. ✅ Deployed and verified the service

  

The orchestrator should now be able to:

- Connect to PostgreSQL

- Submit EMR jobs

- Poll job status

  

Test end-to-end by checking if a batch is processed successfully!


# IAM Configuration Files

  

This directory contains the IAM policy documents needed to grant the orchestrator service access to AWS resources.

  

## Files

  

- **`policy.json`**: IAM policy granting EMR and S3 permissions

- **`trust-policy-template.json`**: Trust policy template for IRSA (requires customization)

  

## Quick Setup

  

### 1. Get your EKS OIDC Provider ID

  

```bash

aws eks describe-cluster --name <your-cluster-name> --query "cluster.identity.oidc.issuer" --output text

# Output: https://oidc.eks.us-east-1.amazonaws.com/id/XXXXXXXXXXXXX

# ^^^^^^^^^^^^^^^^

# This is your OIDC_PROVIDER_ID

```

  

### 2. Create the IAM Policy

  

```bash

aws iam create-policy \

--policy-name tmv-orchestrator-service-policy \

--policy-document file://policy.json

```

  

### 3. Customize Trust Policy

  

Copy the template and replace placeholders:

  

```bash

cp trust-policy-template.json trust-policy.json

  

# Replace <REPLACE_WITH_OIDC_PROVIDER_ID> with your OIDC provider ID (3 occurrences)

# Replace <REPLACE_WITH_NAMESPACE> with your Kubernetes namespace (default: "default")

  

# Example using sed (macOS):

sed -i '' 's/<REPLACE_WITH_OIDC_PROVIDER_ID>/ABC123DEF456/g' trust-policy.json

sed -i '' 's/<REPLACE_WITH_NAMESPACE>/default/g' trust-policy.json

  

# Example using sed (Linux):

sed -i 's/<REPLACE_WITH_OIDC_PROVIDER_ID>/ABC123DEF456/g' trust-policy.json

sed -i 's/<REPLACE_WITH_NAMESPACE>/default/g' trust-policy.json

```

  

### 4. Create the IAM Role

  

```bash

aws iam create-role \

--role-name tmv-orchestrator-service-role \

--assume-role-policy-document file://trust-policy.json

```

  

### 5. Attach Policy to Role

  

```bash

aws iam attach-role-policy \

--role-name tmv-orchestrator-service-role \

--policy-arn arn:aws:iam::744958734165:policy/tmv-orchestrator-service-policy

```

  

### 6. Deploy Helm Chart

  

The `values.yaml` already has the service account annotation configured:

  

```bash

cd ..

helm upgrade --install tmv-orchestrator-service . \

-f values.yaml \

--namespace <your-namespace>

```

  

## Verification

  

```bash

# Verify service account

kubectl get sa tmv-orchestrator-service -n <your-namespace> -o yaml

  

# Check pod for AWS credentials

kubectl get pods -n <your-namespace>

kubectl describe pod <pod-name> -n <your-namespace> | grep -A 5 "AWS"

  

# Test the endpoint

curl -X POST http://tmv-orchestrator.hilabs.internal/api/v1/spark/jobs \

-H "Content-Type: application/json" \

-d '{}'

```

  

## Notes

  

- The trust policy uses IRSA (IAM Roles for Service Accounts) for secure, credential-less authentication

- The service account name is `tmv-orchestrator-service` (auto-generated from the chart name)

- Don't commit `trust-policy.json` (the customized version) to version control

- **S3 Access**: The orchestrator service only needs EMR permissions. S3 access is handled by the EMR cluster's own IAM role (`EMR_EC2_DefaultRole`), not by this service account.