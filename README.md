
# platform-velero-showcases
Kubernetes Backup and Migration Strategies with Velero

- [platform-velero-showcases](#platform-velero-showcases)
- [Requirements](#requirements)
  - [AWS](#aws)
    - [Backup location](#backup-location)
    - [IAM Permissions](#iam-permissions)
    - [Initial status](#initial-status)
  - [GCP](#gcp)
    - [Cluster Creation](#cluster-creation)
- [Disaster Recovery](#disaster-recovery)
  - [Show me the code](#show-me-the-code)
- [Data Migration](#data-migration)
  - [Show me the code](#show-me-the-code-1)
- [References](#references)



The motivation of this repository is to show some Velero use cases:
* Disaster recovery
* Data migration 

In this repo a disaster recovery on EKS will be tested and a data migration between EKS and GKE will be showed. 

# Requirements

* AWS 
  * Cluster creation
  * A backup location
  * IAM Permissions
  * Initial Status
* GCP
  * Cluster creation

For testing reasons, this is done using AWS CLI and GCLOUD CLI.

## AWS

### Backup location
For the backup location a S3 bucket should be created:
```sh
# Remember to specify 
# BUCKET="k8s-days-backup"
# REGION="us-west-1"
aws s3api create-bucket \
  --bucket $BUCKET \
  --region $REGION \
  --create-bucket-configuration LocationConstraint=$REGION


```
### IAM Permissions

Velero workloads need permissions to interact with the Snapshot API from AWS and the S3 bucket. An AWS user will be created with an access key. 

```sh
cat > velero-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeVolumes",
                "ec2:DescribeSnapshots",
                "ec2:CreateTags",
                "ec2:CreateVolume",
                "ec2:CreateSnapshot",
                "ec2:DeleteSnapshot"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:PutObject",
                "s3:AbortMultipartUpload",
                "s3:ListMultipartUploadParts"
            ],
            "Resource": [
                "arn:aws:s3:::${BUCKET}/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::${BUCKET}"
            ]
        }
    ]
}
EOF

aws iam create-user --user-name velero

 aws iam put-user-policy \                              --user-name velero \
  --policy-name velero \
  --policy-document file://velero-policy.json

aws iam create-access-key --user-name velero

```

### Initial status

```sh
# Deploy a EKS Cluster using eksctl called cluster1
eksctl create cluster -f aws/cluster1/cluster.yaml
# Install Ghost workloads
cd workloads/ghost


# Initial Ghost 

helm install --create-namespace  -n ghost -f values.yaml ghost .

# Install Velero workload
cd workloads/velero
helm install --create-namespace  -n velero -f values.yaml velero .
```

## GCP

### Cluster Creation

```sh
# Deploy cluster3

gcloud container clusters create cluster3 \
    --zone europe-west1-b \
    --node-locations europe-west1-b \
    --disk-size=10GB \
    --preemptible \
    --num-nodes=3 \
    --machine-type=n2-standard-2
```

# Disaster Recovery 

Everything can fails, and Velero help us to recover from a disaster recovery. For this sample, the goal will be recover a namespace in other EKS cluster from a previous Velero backup. A sample diagram would be as follows:

## Show me the code

```sh
# Deploy a EKS Cluster called cluster2

eksctl create cluster -f aws/cluster2/cluster.yaml

# Make a backup on cluster1

velero backup create cluster1 --include-namespaces ghost --wait

# Review backup status on cluster1

velero backup describe cluster1 --details

# Restore backup on cluster2
velero restore create --from-backup cluster1 --include-namespaces ghost

# Review restore status on cluster2

velero restore describe xxxx

# Check workloads restored

```

# Data Migration

Sometimes for business reasons, data should be migrated between cloud providers. Because the snapshot API between the different cloud providers doesn't interact between each other an agnostic layer is needed. Restic help us to provide this file system level agnosticism and allow interact file system between cloud providers.
A sample workflow would be as follows:


## Show me the code
```sh

# Upgrade Velero with Restic on Cluster2
helm upgrade -f values-restic.yaml velero .

# Create a backup 

velero backup create cluster2 --include-namespaces ghost --wait

# Review detailed backup status

velero backup describe cluster2 --details

# Install Velero with Restic on Cluster3

helm install --create-namespace  -n velero -f values-restic.yaml velero .

# List backups on cluster3

velero get backups

# Restore from backup on cluster3

velero restore create --from-backup cluster2 --include-namespaces ghost

# Check workloads restored

kubectl get po -n ghost
```

# References
1. [Installation eksctl](https://eksctl.io/introduction/#installation)
2. [Create a simple EKS cluster with eksctl](https://eksctl.io/usage/creating-and-managing-clusters/)
2. [Kubernetes Migration](https://www.velotio.com/engineering-blog/kubernetes-migration-across-clusters)
3. [Velero Helm Chart](https://github.com/vmware-tanzu/helm-charts/tree/velero-2.24.0/charts/velero)
4. [Creating a GKE Zonal Cluster](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-a-zonal-cluster)
5. [Empathy Public Helm Charts](https://github.com/empathyco/helm-charts)
6. [Restic](https://restic.net/)
7. [Velero Restic Integration](https://velero.io/docs/main/restic/)
8. [AWS Permissions](https://www.fourco.nl/blogs/backup-and-restore-a-kubernetes-cluster-with-velero/)
9. [Ghost Helm Chart](https://github.com/bitnami/charts/tree/master/bitnami/ghost)