# Batch-Processing-with-Argo-Workflow (中文版)

## Batch Processing
在本章中，我们将使用 Kubernetes 和 Argo 部署常见的批处理场景。 

![image](./argo-logo.png)

### ARGO是什么？
Argo Workflows 是一个开源容器原生工作流引擎，用于在 Kubernetes 上编排并行作业。 Argo 工作流作为 Kubernetes CRD（自定义资源定义）实现。
* 定义工作流，其中工作流中的每个步骤都是一个容器。
* 将多步骤工作流建模为一系列任务或使用有向无环图 (DAG) 捕获任务之间的依赖关系。
* 使用 Kubernetes 上的 Argo Workflows 在很短的时间内轻松运行用于机器学习或数据处理的计算密集型作业。
* 在 Kubernetes 上本地运行 CI/CD 管道，无需配置复杂的软件开发产品。

Argo is a [Cloud Native Computing Foundation (CNCF)](https://cncf.io/) hosted project.

