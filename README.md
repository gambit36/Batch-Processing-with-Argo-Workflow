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

···mkdir  ~/environment/batch_policy/

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
 ···
