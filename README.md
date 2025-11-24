# LLMD EKS

## Setup

Open Environment AWS (Demo.redhat) <https://catalog.demo.redhat.com/catalog?search=blank&item=babylon-catalog-prod%2Fsandboxes-gpte.sandbox-open.prod>

<https://docs.aws.amazon.com/eks/latest/eksctl/installation.html>

```sh
aws configure
eksctl create cluster
```

<https://github.com/llm-d/llm-d/blob/v0.3.1/guides/prereq/gateway-provider/README.md>
<https://github.com/llm-d-incubation/llm-d-infra/blob/main/charts/llm-d-infra/README.md#tldr>
<https://github.com/llm-d-incubation/llm-d-modelservice/blob/main/README.md#getting-started>

<https://docs.aws.amazon.com/batch/latest/userguide/create-gpu-cluster-eks.html>

## p5.4xlarge

<https://docs.aws.amazon.com/batch/latest/userguide/getting-started-eks.html#getting-started-eks-step-1>

```sh
namespace=my-aws-batch-namespace
cat - <<EOF | kubectl create -f -
{
  "apiVersion": "v1",
  "kind": "Namespace",
  "metadata": {
    "name": "${namespace}",
    "labels": {
      "name": "${namespace}"
    }
  }
}
EOF
cat - <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: aws-batch-cluster-role
rules:
  - apiGroups: [""]
    resources: ["namespaces"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["list"]
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["daemonsets", "deployments", "statefulsets", "replicasets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["rbac.authorization.k8s.io"]
    resources: ["clusterroles", "clusterrolebindings"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: aws-batch-cluster-role-binding
subjects:
- kind: User
  name: aws-batch
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: aws-batch-cluster-role
  apiGroup: rbac.authorization.k8s.io
EOF
namespace=my-aws-batch-namespace
cat - <<EOF | kubectl apply -f - --namespace "${namespace}"
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: aws-batch-compute-environment-role
  namespace: ${namespace}
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["create", "get", "list", "watch", "delete", "patch"]
  - apiGroups: [""]
    resources: ["serviceaccounts"]
    verbs: ["get", "list"]
  - apiGroups: ["rbac.authorization.k8s.io"]
    resources: ["roles", "rolebindings"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: aws-batch-compute-environment-role-binding
  namespace: ${namespace}
subjects:
- kind: User
  name: aws-batch
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: aws-batch-compute-environment-role
  apiGroup: rbac.authorization.k8s.io
EOF

eksctl get cluster
#<https://<acct-id>.signin.aws.amazon.com/console>
eksctl create iamidentitymapping \
    --cluster fabulous-badger-1764007365 \
    --arn "arn:aws:iam::<acct-id>:role/AWSServiceRoleForBatch" \
    --username aws-batch

aws eks describe-cluster \
    --name fabulous-badger-1764007365 \
    --query cluster.resourcesVpcConfig.clusterSecurityGroupId
# "sg-0524f68e97eac4ace"

aws iam list-entities-for-policy --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy

# {
#     "PolicyGroups": [],
#     "PolicyUsers": [],
#     "PolicyRoles": [
#         {
#             "RoleName": "eksctl-fabulous-badger-1764007365--NodeInstanceRole-fWK99bpuO1Mj",
#             "RoleId": "AROAZV6M65P4RPRIXLWHN"
#         }
#     ]
# }

aws iam list-instance-profiles-for-role --role-name eksctl-fabulous-badger-1764007365--NodeInstanceRole-fWK99bpuO1Mj      
    # "InstanceProfiles": [
    #     {
    #         "Path": "/",
    #         "InstanceProfileName": "eks-32cd5b89-690e-5165-e0ca-d13007bbf2dd",
    #         "InstanceProfileId": "AIPAZV6M65P4SNBNPDNP7",
    #         "Arn": "arn:aws:iam::<acct-id>:instance-profile/eks-32cd5b89-690e-5165-e0ca-d13007bbf2dd",
    #         "CreateDate": "2025-11-24T18:14:30+00:00",
    #         "Roles": [
```

```sh
cat <<EOF > ./batch-eks-gpu-ce.json
{
  "computeEnvironmentName": "My-Eks-GPU-CE1",
  "type": "MANAGED",
  "state": "ENABLED",
  "eksConfiguration": {
    "eksClusterArn": "arn:aws:eks:us-east-2:<acct-id>:cluster/fabulous-badger-1764007365",
    "kubernetesNamespace": "my-aws-batch-namespace"
  },
  "computeResources": {
    "type": "EC2",
    "allocationStrategy": "BEST_FIT_PROGRESSIVE",
    "minvCpus": 0,
    "maxvCpus": 128,
    "instanceTypes": [
      "p5.4xlarge"
    ],
    "subnets": [
        "subnet-0e44daa6557af8f77",
        "subnet-0e20989fa83302683",
        "subnet-04ff2bd5c4be7423b",
        "subnet-054cddb44728affd6",
        "subnet-0d346a2542a65706f",
        "subnet-01644c028380baccd"
    ],
    "securityGroupIds": [
        "sg-0524f68e97eac4ace"
    ],
    "instanceRole": "eks-32cd5b89-690e-5165-e0ca-d13007bbf2dd"
  }
}
EOF

aws batch create-compute-environment --cli-input-json file://./batch-eks-gpu-ce.json
aws batch update-compute-environment --compute-environment My-Eks-GPU-CE1 --state DISABLED
aws batch delete-compute-environment --compute-environment My-Eks-GPU-CE1

aws batch describe-compute-environments  --compute-environments My-Eks-GPU-CE1
# {
#     "computeEnvironments": [
#         {
#             "computeEnvironmentName": "My-Eks-GPU-CE1",
#             "computeEnvironmentArn": "arn:aws:batch:us-east-2:<acct-id>:compute-environment/My-Eks-GPU-CE1",
#             "tags": {},
#             "type": "MANAGED",
#             "state": "ENABLED",
#             "status": "VALID",

curl -O https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.12.2/nvidia-device-plugin.yml

kubectl apply -f nvidia-device-plugin.yml

cat <<EOF > ./batch-eks-gpu-jq.json
 {
    "jobQueueName": "My-Eks-GPU-JQ1",
    "priority": 10,
    "computeEnvironmentOrder": [
      {
        "order": 1,
        "computeEnvironment": "My-Eks-GPU-CE1"
      }
    ]
  }
EOF

aws batch create-job-queue --cli-input-json file://./batch-eks-gpu-jq.json


cat <<EOF > ./batch-eks-gpu-jd.json
{
    "jobDefinitionName": "MyGPUJobOnEks_Smi",
    "type": "container",
    "eksProperties": {
        "podProperties": {
            "hostNetwork": true,
            "containers": [
                {
                    "image": "nvcr.io/nvidia/cuda:10.2-runtime-centos7",
                    "command": ["nvidia-smi"],
                    "resources": {
                        "limits": {
                            "cpu": "1",
                            "memory": "1024Mi",
                            "nvidia.com/gpu": "1"
                        }
                    }
                }
            ]
        }
    }
}
EOF

aws batch register-job-definition --cli-input-json file://./batch-eks-gpu-jd.json

aws batch submit-job --job-queue My-Eks-GPU-JQ1 --job-definition MyGPUJobOnEks_Smi --job-name My-Eks-GPU-Job

# {
#     "jobArn": "arn:aws:batch:us-east-2:<acct-id>:job/69d1c1d4-5382-4130-9e8b-33da57d6f23d",
#     "jobName": "My-Eks-GPU-Job",
#     "jobId": "69d1c1d4-5382-4130-9e8b-33da57d6f23d"
# }
aws batch describe-jobs --job "69d1c1d4-5382-4130-9e8b-33da57d6f23d" | jq '.jobs[].eksProperties.podProperties | {podName, nodeName}'


```
