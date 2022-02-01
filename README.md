# aks1226

let data = materialize(
InsightsMetrics
| where Name == "kube_deployment_status_replicas_ready"
| extend Tags = parse_json(Tags)
| extend ClusterId = Tags["container.azm.ms/clusterId"]
{clusterIdWhereClause}
| where Tags.deployment in ({deploymentName})
| extend Deployment = tostring(Tags.deployment)
| extend k8sNamespace = tostring(Tags.k8sNamespace)
| extend Ready = Val/Tags.spec_replicas * 100, Updated = Val/Tags.status_replicas_updated * 100, Available = Val/Tags.status_replicas_available * 100
| project k8sNamespace, Deployment, Ready, Updated, Available, TimeGenerated, Tags
);
let data2 = data
| extend Age = (now() - todatetime(Tags["creationTime"]))/1m
| summarize arg_max(TimeGenerated, *) by k8sNamespace, Deployment
| project k8sNamespace, Deployment, Age, Ready, Updated, Available;
let ReadyData = data
| make-series ReadyTrend = avg(Ready) default = 0 on TimeGenerated from {timeRange:start} to {timeRange:end} step {timeRange:grain} by k8sNamespace, Deployment;
let UpdatedData = data
| make-series UpdatedTrend = avg(Updated) default = 0 on TimeGenerated from {timeRange:start} to {timeRange:end} step {timeRange:grain} by k8sNamespace, Deployment;
let AvailableData = data
| make-series AvailableTrend = avg(Available) default = 0 on TimeGenerated from {timeRange:start} to {timeRange:end} step {timeRange:grain} by k8sNamespace, Deployment;
data2
| join kind = inner(ReadyData) on k8sNamespace, Deployment 
| join kind = inner(UpdatedData) on k8sNamespace, Deployment 
| join kind = inner(AvailableData) on k8sNamespace, Deployment
| extend ReadyCase = case(Ready == 100, "Healthy", "Warning"),  UpdatedCase = case(Updated == 100, "Healthy", "Warning"),  AvailableCase = case(Available == 100, "Healthy", "Warning")
| extend Overall = case(ReadyCase == "Healthy" and UpdatedCase == "Healthy" and AvailableCase == "Healthy", "Healthy", "Warning")
| extend OverallFilterStatus = case('{OverallFilter}' contains "Healthy", "Healthy", '{OverallFilter}' contains "Warning", "Warning", "Healthy, Warning")
| where OverallFilterStatus has Overall
| project Deployment, Namespace = k8sNamespace, Age, Ready, ReadyTrend, Updated, UpdatedTrend, Available,AvailableTrend
| sort by Ready asc



















ContainerLog
|join kind=leftouter (
KubePodInventory
| distinct Computer, ClusterId, ServiceName, ControllerName, ContainerID, ContainerName, Name, Namespace, ClusterName
| where Namespace contains ({namespace}) 
| project-rename PodName=Name
| join kind=inner (InsightsMetrics 
| where Name == "kube_deployment_status_replicas_ready"
| extend Tags = parse_json(Tags)
| extend ClusterId = Tags["container.azm.ms/clusterId"]
{clusterIdWhereClause}
| where Tags.deployment in ({deploymentName})
| extend Deployment = tostring(Tags.deployment)
| extend k8sNamespace = tostring(Tags.k8sNamespace)
))//KubePodInventory Contains pod metadata
on ContainerID
| extend ContainerName = split(ContainerName, "/")[1] // KubePodInventory stores ContainerName in format PodUI/container name, hence we split
| where Namespace contains ({namespace})
| sort by TimeGenerated
| project TimeGenerated, Namespace, PodName, ContainerName, LogEntry, LogEntrySource
