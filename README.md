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

## 高级批处理工作流 ##
让我们看一个更复杂的工作流程，包括在作业、多个依赖项等之间传递工件。

使用以下命令创建 teardrop.yaml： 
```
cat <<EoF > ~/environment/batch_policy/teardrop.yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: teardrop-
spec:
  entrypoint: teardrop
  templates:
  - name: create-chain
    container:
      image: alpine:latest
      command: ["sh", "-c"]
      args: ["echo '' >> /tmp/message"]
    outputs:
      artifacts:
      - name: chain
        path: /tmp/message
  - name: whalesay
    inputs:
      parameters:
      - name: message
      artifacts:
      - name: chain
        path: /tmp/message
    container:
      image: docker/whalesay
      command: ["sh", "-c"]
      args: ["echo Chain: ; cat /tmp/message* | sort | uniq | tee /tmp/message; cowsay This is Job {{inputs.parameters.message}}! ; echo {{inputs.parameters.message}} >> /tmp/message"]
    outputs:
      artifacts:
      - name: chain
        path: /tmp/message
  - name: whalesay-reduce
    inputs:
      parameters:
      - name: message
      artifacts:
      - name: chain-0
        path: /tmp/message.0
      - name: chain-1
        path: /tmp/message.1
    container:
      image: docker/whalesay
      command: ["sh", "-c"]
      args: ["echo Chain: ; cat /tmp/message* | sort | uniq | tee /tmp/message; cowsay This is Job {{inputs.parameters.message}}! ; echo {{inputs.parameters.message}} >> /tmp/message"]
    outputs:
      artifacts:
      - name: chain
        path: /tmp/message
  - name: teardrop
    dag:
      tasks:
      - name: create-chain
        template: create-chain
      - name: Alpha
        dependencies: [create-chain]
        template: whalesay
        arguments:
          parameters: [{name: message, value: Alpha}]
          artifacts:
            - name: chain
              from: "{{tasks.create-chain.outputs.artifacts.chain}}"
      - name: Bravo
        dependencies: [Alpha]
        template: whalesay
        arguments:
          parameters: [{name: message, value: Bravo}]
          artifacts:
            - name: chain
              from: "{{tasks.Alpha.outputs.artifacts.chain}}"
      - name: Charlie
        dependencies: [Alpha]
        template: whalesay
        arguments:
          parameters: [{name: message, value: Charlie}]
          artifacts:
            - name: chain
              from: "{{tasks.Alpha.outputs.artifacts.chain}}"
      - name: Delta
        dependencies: [Bravo]
        template: whalesay
        arguments:
          parameters: [{name: message, value: Delta}]
          artifacts:
            - name: chain
              from: "{{tasks.Bravo.outputs.artifacts.chain}}"
      - name: Echo
        dependencies: [Bravo, Charlie]
        template: whalesay-reduce
        arguments:
          parameters: [{name: message, value: Echo}]
          artifacts:
            - name: chain-0
              from: "{{tasks.Bravo.outputs.artifacts.chain}}"
            - name: chain-1
              from: "{{tasks.Charlie.outputs.artifacts.chain}}"
      - name: Foxtrot
        dependencies: [Charlie]
        template: whalesay
        arguments:
          parameters: [{name: message, value: Foxtrot}]
          artifacts:
            - name: chain
              from: "{{tasks.create-chain.outputs.artifacts.chain}}"
      - name: Golf
        dependencies: [Delta, Echo]
        template: whalesay-reduce
        arguments:
          parameters: [{name: message, value: Golf}]
          artifacts:
            - name: chain-0
              from: "{{tasks.Delta.outputs.artifacts.chain}}"
            - name: chain-1
              from: "{{tasks.Echo.outputs.artifacts.chain}}"
      - name: Hotel
        dependencies: [Echo, Foxtrot]
        template: whalesay-reduce
        arguments:
          parameters: [{name: message, value: Hotel}]
          artifacts:
            - name: chain-0
              from: "{{tasks.Echo.outputs.artifacts.chain}}"
            - name: chain-1
              from: "{{tasks.Foxtrot.outputs.artifacts.chain}}"
EoF
```

此工作流使用[有向无环图 (DAG)](https://en.wikipedia.org/wiki/Directed_acyclic_graph) 来明确定义作业依赖项。 工作流中的每个作业都会调用一个 whalesay 模板并传递一个具有唯一名称的参数。 一些工作调用了一个whalesay-reduce 模板，它接受多个工件并将它们组合成一个工件。

工作流中的每个作业都会提取工件并将它们列在“链”中，然后为当前作业调用 whalesay。 然后，每个作业都会有一个先前作业依赖链的列表（在当前作业可以运行之前必须完成的所有作业的列表）。

运行工作流。 

```
argo -n argo submit --watch ~/environment/batch_policy/teardrop.yaml
```

```
Name:                teardrop-vqbmb
Namespace:           argo
ServiceAccount:      default
Status:              Succeeded
Conditions:
 Completed           True
Created:             Tue Jul 07 20:32:12 +0000 (42 seconds ago)
Started:             Tue Jul 07 20:32:12 +0000 (42 seconds ago)
Finished:            Tue Jul 07 20:32:54 +0000 (now)
Duration:            42 seconds
ResourcesDuration:   32s*(1 cpu),32s*(100Mi memory)

STEP               TEMPLATE         PODNAME                    DURATION  MESSAGE
 ✔ teardrop-vqbmb  teardrop
 ├-✔ create-chain  create-chain     teardrop-vqbmb-1083106731  3s
 ├-✔ Alpha         whalesay         teardrop-vqbmb-2236987393  3s
 ├-✔ Bravo         whalesay         teardrop-vqbmb-1872757121  4s
 ├-✔ Charlie       whalesay         teardrop-vqbmb-2266260663  4s
 ├-✔ Delta         whalesay         teardrop-vqbmb-2802530727  18s
 ├-✔ Echo          whalesay-reduce  teardrop-vqbmb-2599957478  4s
 ├-✔ Foxtrot       whalesay         teardrop-vqbmb-1298400165  4s
 ├-✔ Hotel         whalesay-reduce  teardrop-vqbmb-3381869223  8s
 └-✔ Golf          whalesay-reduce  teardrop-vqbmb-1766004759  8s
 ```

## Argo Dashboard ##

从 V2.5 开始，Argo UI 已被 Argo Server 取代。 新 UI 不是只读的——它还具有直接在浏览器中创建和更新数据的能力。 点击[这里](https://blog.argoproj.io/argo-workflows-v2-5-released-ce7553bfd84c)查看更多信息 
![image](https://www.eksworkshop.com/images/argo-workflow/argo-dashboard.png)
## 访问 Argo Server ##

为了访问仪表板，我们将通过添加 LoadBalancer 来公开 argo-server 服务。 

```
kubectl -n argo patch svc argo-server  \
   -p '{"spec": {"type": "LoadBalancer"}}'
```
要访问 Argo 仪表板，请点击以下命令生成的 URL：

```
ARGO_URL=$(kubectl -n argo get svc argo-server --template "{{ range (index .status.loadBalancer.ingress 0) }}{{ . }}{{ end }}")

echo ARGO DASHBOARD: http://${ARGO_URL}:2746
```

您将看到 [Advanced Batch Workflow](https://www.eksworkshop.com/advanced/410_batch/workflow-advanced/) 中的*teardrop*工作流程。 单击它以查看工作流程的可视化。 

![image](https://www.eksworkshop.com/images/argo-workflow/argo-workflow.png)

工作流应该看起来像一个泪珠，并为每个作业提供实时状态。 单击 *Hotel*以查看Hotel工作的摘要。 

![image](https://www.eksworkshop.com/images/argo-workflow/argo-hotel-job.png)

这详细说明了有关该作业的基本信息，并包括指向日志的链接。 Hotel作业日志列出了作业依赖链和当前的 whalesay，应该类似于：

![image](https://www.eksworkshop.com/images/argo-workflow/argo-logs.png)

探索工作流中的其他作业以查看每个作业的状态和日志。 

## 清理


**删除所有工作流 **

```
argo -n argo delete --all
```

**删除Artifact Repository**

```
aws s3 rb s3://batch-artifact-repository-${ACCOUNT_ID}/ --force
```

**删除AGRO**

```
kubectl delete -n argo -f https://raw.githubusercontent.com/argoproj/argo-workflows/${ARGO_VERSION}/manifests/install.yaml

kubectl delete namespace argo
```

**删除Kubernetes Job**

```
kubectl delete job/whalesay
```

**删除内联策略**

```
 aws iam delete-role-policy --role-name $ROLE_NAME --policy-name S3-Policy-For-Worker
```

**删除目录和文件**

```
rm -rf ~/environment/batch_policy
```
