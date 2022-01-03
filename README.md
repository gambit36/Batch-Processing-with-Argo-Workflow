# Batch-Processing-with-Argo-Workflow (中文版)

## Batch Processing
在本章中，我们将使用 Kubernetes 和 [Argo](https://argoproj.github.io/) 部署常见的批处理场景。 

![image](./argo-logo.png)

## ARGO是什么？
Argo Workflows 是一个开源容器原生工作流引擎，用于在 Kubernetes 上编排并行作业。 Argo 工作流作为 Kubernetes CRD（自定义资源定义）实现。
* 定义工作流，其中工作流中的每个步骤都是一个容器。
* 将多步骤工作流建模为一系列任务或使用有向无环图 (DAG) 捕获任务之间的依赖关系。
* 使用 Kubernetes 上的 Argo Workflows 在很短的时间内轻松运行用于机器学习或数据处理的计算密集型作业。
* 在 Kubernetes 上本地运行 CI/CD 管道，无需配置复杂的软件开发产品。

Argo is a [Cloud Native Computing Foundation (CNCF)](https://cncf.io/) hosted project.

## 介绍 
批处理是指以重复和无人值守的方式执行工作单元，称为作业。 作业通常组合在一起并分批处理（因此得名）。 
Kubernetes 包括对[运行作业](https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/)的原生支持。 作业可以并行运行多个 Pod，直到收到一定数量的完成。 每个 Pod 可以包含多个容器作为单个工作单元。
Argo 通过引入许多功能来增强批处理体验： 
* 基于步骤的工作流声明
* Artifact支持
* step级别输入和输出
* 循环
* 条件
* 可视化（使用 Argo 仪表板）
* …和更多 

在本模块中，我们将构建一个简单的 Kubernetes 作业，在 Argo 中重新创建该作业，并为更高级的批处理添加通用功能和工作流。

## Kubernetes Jobs 
一项作业创建一个或多个 Pod，并确保指定数量的 Pod 成功终止。 当 Pod 成功完成时，该作业会跟踪成功完成情况。 当达到指定的成功完成次数时，作业本身就完成了。 删除 Job 将清理它创建的 Pod。

让我们从使用此命令创建 job-whalesay.yaml 清单开始 

```
mkdir  ~/environment/batch_policy/

cat <<EoF > ~/environment/batch_policy/job-whalesay.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: whalesay
spec:
  template:
    spec:
      containers:
      - name: whalesay
        image: docker/whalesay
        command: ["cowsay",  "This is a Kubernetes Job!"]
      restartPolicy: Never
  backoffLimit: 4
EoF

```

使用 whalesay image 运行示例 Kubernetes 作业。 

```
kubectl apply -f ~/environment/batch_policy/job-whalesay.yaml
```

等待作业成功完成。 
```
kubectl get job/whalesay
```

```
NAME       COMPLETIONS   DURATION   AGE
whalesay   1/1           3s         21s
```

确认输出

```
kubectl logs -l job-name=whalesay
```

```

 ___________________________ 
< This is a Kubernetes Job! >
 --------------------------- 
    \
     \
      \     
                    ##        .            
              ## ## ##       ==            
           ## ## ## ##      ===            
       /""""""""""""""""___/ ===        
  ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~   
       \______ o          __/            
        \    \        __/             
          \____\______/   
```
## 安装 Argo CLI
在我们开始配置 argo 之前，我们需要先安装您将与之交互的命令行工具。 为此，请运行以下命令。 
```
# set argo version
export ARGO_VERSION="v2.12.13"
```
```
# Download the binary
curl -sLO https://github.com/argoproj/argo-workflows/releases/download/${ARGO_VERSION}/argo-linux-amd64.gz

# Unzip
gunzip argo-linux-amd64.gz

# Make binary executable
chmod +x argo-linux-amd64

# Move binary to path
sudo mv ./argo-linux-amd64 /usr/local/bin/argo

# Test installation
argo version --short
```
输出
```

argo: v2.12.13
```

## 部署 Argo Controller

Argo 在其自己的命名空间中运行并部署为 CustomResourceDefinition。

部署Controller和 UI。 

```
kubectl create namespace argo
kubectl apply -n argo -f https://raw.githubusercontent.com/argoproj/argo-workflows/${ARGO_VERSION}/manifests/install.yaml
```
```
customresourcedefinition.apiextensions.k8s.io/clusterworkflowtemplates.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/cronworkflows.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/workflows.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/workflowtemplates.argoproj.io created
serviceaccount/argo created
serviceaccount/argo-server created
role.rbac.authorization.k8s.io/argo-role created
clusterrole.rbac.authorization.k8s.io/argo-aggregate-to-admin created
clusterrole.rbac.authorization.k8s.io/argo-aggregate-to-edit created
clusterrole.rbac.authorization.k8s.io/argo-aggregate-to-view created
clusterrole.rbac.authorization.k8s.io/argo-cluster-role created
clusterrole.rbac.authorization.k8s.io/argo-server-cluster-role created
rolebinding.rbac.authorization.k8s.io/argo-binding created
clusterrolebinding.rbac.authorization.k8s.io/argo-binding created
clusterrolebinding.rbac.authorization.k8s.io/argo-server-binding created
configmap/workflow-controller-configmap created
service/argo-server created
service/workflow-controller-metrics created
deployment.apps/argo-server created
deployment.apps/workflow-controller created
```

## 配置service account以运行工作流 ##
为了让 Argo 支持artifacts、输出、访问机密等功能，它需要使用 Kubernetes API 与 Kubernetes 资源进行通信。 为了与 Kubernetes API 通信，Argo 使用 ServiceAccount 向 Kubernetes API 验证自己。 您可以通过使用 RoleBinding 将role绑定到 ServiceAccount 来指定 Argo 使用的 ServiceAccount 的哪个角色（即哪些权限） 

```
kubectl -n argo create rolebinding default-admin --clusterrole=admin --serviceaccount=argo:default
```

## 配置 Artifact Repository##
Argo 使用Artifact Repository在工作流中的作业之间传递数据，称为Artifacts。 Amazon S3 可用作Artifact Repository。

让我们使用 AWS CLI 创建一个 S3 存储桶。 
```
aws s3 mb s3://batch-artifact-repository-${ACCOUNT_ID}/
```
接下来，我们将在 configmap workflow-controller-configmap 中添加这个桶作为 argo artifactRepository

创建 patch

```
cat <<EoF > ~/environment/batch_policy/argo-patch.yaml
data:
  config: |
    artifactRepository:
      s3:
        bucket: batch-artifact-repository-${ACCOUNT_ID}
        endpoint: s3.amazonaws.com
EoF
```

部署 patch

```
kubectl -n argo patch \
  configmap/workflow-controller-configmap \
  --patch "$(cat ~/environment/batch_policy/argo-patch.yaml)"
```

校验configmap

```
kubectl -n argo get configmap/workflow-controller-configmap -o yaml
```

输出
```
apiVersion: v1
data:
  config: |-
    artifactRepository:
      s3:
        bucket: batch-artifact-repository-197520326489
        endpoint: s3.amazonaws.com
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"ConfigMap","metadata":{"annotations":{},"name":"workflow-controller-configmap","namespace":"argo"}}
  creationTimestamp: "2020-07-07T19:07:48Z"
  name: workflow-controller-configmap
  namespace: argo
  resourceVersion: "1082653"
  selfLink: /api/v1/namespaces/argo/configmaps/workflow-controller-configmap
  uid: a1accd9e-c528-41a3-b811-226f5662c446
  ```
  
## 创建一个IAM Policy
  
为了让 Argo 读取/写入 S3 存储桶，我们需要配置内联策略并将其添加到工作节点的 EC2 实例配置文件中。

首先，我们需要确保在我们的环境中设置了我们的工作人员使用的角色名称： 
```
test -n "$ROLE_NAME" && echo ROLE_NAME is "$ROLE_NAME" || echo ROLE_NAME is not set
```

如果您收到错误或空响应，请查看：[/030_eksctl/test/](https://www.eksworkshop.com/030_eksctl/test/) 

```
# Example Output
ROLE_NAME is eksctl-eksworkshop-eksctl-nodegro-NodeInstanceRole-RPDET0Z4IJIF
```

创建和策略并附加到工作节点角色。 

```
cat <<EoF > ~/environment/batch_policy/k8s-s3-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:*"
      ],
      "Resource": [
        "arn:aws:s3:::batch-artifact-repository-${ACCOUNT_ID}",
        "arn:aws:s3:::batch-artifact-repository-${ACCOUNT_ID}/*"
      ]
    }
  ]
}
EoF

aws iam put-role-policy --role-name $ROLE_NAME --policy-name S3-Policy-For-Worker --policy-document file://~/environment/batch_policy/k8s-s3-policy.json
```

验证策略是否附加到角色 
```
aws iam get-role-policy --role-name $ROLE_NAME --policy-name S3-Policy-For-Worker
```

## 简单的批处理工作流 

创建清单工作流 whalesay.yaml，使用 Argo 之前的 whalesay 示例。 

```
cat <<EoF > ~/environment/batch_policy/workflow-whalesay.yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: whalesay-
spec:
  entrypoint: whalesay
  templates:
  - name: whalesay
    container:
      image: docker/whalesay
      command: [cowsay]
      args: ["This is an Argo Workflow!"]
EoF
```

现在使用 argo CLI 部署工作流。 

```
argo -n argo submit --watch ~/environment/batch_policy/workflow-whalesay.yaml
```

```
Name:                whalesay-rlssg
Namespace:           argo
ServiceAccount:      default
Status:              Succeeded
Conditions:
 Completed           True
Created:             Wed Jul 08 00:13:51 +0000 (3 seconds ago)
Started:             Wed Jul 08 00:13:51 +0000 (3 seconds ago)
Finished:            Wed Jul 08 00:13:54 +0000 (now)
Duration:            3 seconds
ResourcesDuration:   1s*(1 cpu),1s*(100Mi memory)

STEP               TEMPLATE  PODNAME         DURATION  MESSAGE
 ✔ whalesay-rlssg  whalesay  whalesay-rlssg  2s
 ```
 
通过运行以下命令确认输出： 
```
argo -n argo logs $(argo -n argo list -o name)
```
```
whalesay-rlssg:  ___________________________
whalesay-rlssg: < This is an Argo Workflow! >
whalesay-rlssg:  ---------------------------
whalesay-rlssg:     \
whalesay-rlssg:      \
whalesay-rlssg:       \
whalesay-rlssg:                     ##        .
whalesay-rlssg:               ## ## ##       ==
whalesay-rlssg:            ## ## ## ##      ===
whalesay-rlssg:        /""""""""""""""""___/ ===
whalesay-rlssg:   ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~
whalesay-rlssg:        \______ o          __/
whalesay-rlssg:         \    \        __/
whalesay-rlssg:           \____\______/
```
