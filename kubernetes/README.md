# Kubernetes Integration
{{< img src="integrations/kubernetes/k8sdashboard.png" alt="Kubernetes Dashboard" responsive="true" popup="true">}}
## Overview

Get metrics from kubernetes service in real time to:

* Visualize and monitor kubernetes states
* Be notified about kubernetes failovers and events.

## Setup
### Installation

Install the `dd-check-kubernetes` package manually or with your favorite configuration manager

### Configuration

Edit the `kubernetes.yaml` file to point to your server and port, set the masters to monitor. See the [sample kubernetes.yaml](https://github.com/DataDog/integrations-core/blob/master/kubernetes/conf.yaml.example) for all available configuration options.

### Gathering kubernetes events

As the 5.17.0 release, Datadog Agent now supports built in leader election option for the Kubernetes event collector. Agents coordinate by performing a leader election among members of the Datadog DaemonSet through kubernetes to ensure only one leader agent instance is gathering events at a given time.
If the leader agent instance fails, a re-election occurs and another cluster agent will take over collection.

**This functionality is disabled by default**. 

To enable leader election you need to set the variable `leader_candidate` to true in your kubernetes.yaml file.

This feature relies on [ConfigMaps](https://kubernetes.io/docs/api-reference/v1.7/#configmap-v1-core) , so you will need to grant Datadog Agent get, list, delete and create access to the ConfigMap resource.

Use these Kubernetes RBAC entities for your Datadog agent to properly configure the previous permissions by [applying this datadog service account to your pods](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/).

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: datadog
rules:
- nonResourceURLs:
  - "/version"  # Used to get apiserver version metadata
  - "/healthz"  # Healthcheck
  verbs: ["get"]
- apiGroups: [""]
  resources:
    - "nodes"
    - "namespaces"  #
    - "events"      # Cluster events + kube_service cache invalidation
    - "services"    # kube_service tag
  verbs: ["get", "list"]
- apiGroups: [""]
  resources:
    - "configmaps"
  resourceNames: ["datadog-leader-elector"]
  verbs: ["get", "delete", "update"]
- apiGroups: [""]
  resources:
    - "configmaps"
  verbs: ["create"]
---
# You need to use that account for your dd-agent DaemonSet
apiVersion: v1
kind: ServiceAccount
metadata:
  name: datadog
automountServiceAccountToken: true
---
# Your admin user needs the same permissions to be able to grant them
# Easiest way is to bind your user to the cluster-admin role
# See https://cloud.google.com/container-engine/docs/role-based-access-control#setting_up_role-based_access_control
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: datadog
subjects:
- kind: ServiceAccount
  name: datadog
  namespace: default
roleRef:
  kind: ClusterRole
  name: datadog
  apiGroup: rbac.authorization.k8s.io
```

In your `kubernetes.yaml` file you will see the [leader_lease_duration](https://github.com/DataDog/integrations-core/blob/master/kubernetes/conf.yaml.example#L118) parameter. It's the duration for which a leader stays elected. **It should be > 30 seconds**. 
The longer it is, the less hard your agent hits the apiserver with requests, but it also means that if the leader dies (and under certain conditions) there can be an event blackout until the lease expires and a new leader takes over.

### Validation

[Run the Agent's `info` subcommand](https://docs.datadoghq.com/agent/faq/agent-status-and-information/) and look for `kubernetes` under the Checks section:

    Checks
    ======

        kubernetes
        -----------
          - instance #0 [OK]
          - Collected 39 metrics, 0 events & 7 service checks

## Compatibility

The kubernetes check is compatible with all major platforms

## Data Collected
### Metrics
See [metadata.csv](https://github.com/DataDog/integrations-core/blob/master/kubernetes/metadata.csv) for a list of metrics provided by this integration.

### Events

As the 5.17.0 release, Datadog Agent now supports built in [leader election option](#gathering-kubernetes-events) for the Kubernetes event collector. Once enabled, you no longer need to deploy an additional event collection container to your cluster. Instead agents will coordinate to ensure only one agent instance is gathering events at a given time, events below will be available:

* Backoff
* Conflict
* Delete
* DeletingAllPods
* Didn't have enough resource
* Error
* Failed
* FailedCreate
* FailedDelete
* FailedMount
* FailedSync
* Failedvalidation
* FreeDiskSpaceFailed
* HostPortConflict
* InsufficientFreeCPU
* InsufficientFreeMemory
* InvalidDiskCapacity
* Killing
* KubeletsetupFailed
* NodeNotReady
* NodeoutofDisk
* OutofDisk
* Rebooted
* TerminatedAllPods
* Unable
* Unhealthy

### Service Checks
The Kubernetes check does not include any service check at this time.

## Troubleshooting
### Can I install the agent on my Kubernetes master node(s) ?
Yes, since Kubernetes 1.6, the concept of [Taints and tolerations](http://blog.kubernetes.io/2017/03/advanced-scheduling-in-kubernetes.html) was introduced. Now rather than the master being off limits, it's simply tainted.  Add the required toleration to the pod to run it:

Add the following lines to your Deployment (or Daemonset if you are running a multi-master setup):
```
spec:
 tolerations: 
 - key: node-role.kubernetes.io/master
   effect: NoSchedule
```

### Why is the Kubernetes check failing with a ConnectTimeout error to port 10250?
The agent assumes that the kubelet API is available at the default gateway of the container. If that's not the case because you are using a software defined networks like Calico or Flannel, the agent needs to be specified using an environment variable:
```
          - name: KUBERNETES_KUBELET_HOST
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
```
See [this PR](https://github.com/DataDog/dd-agent/pull/3051)

###  Why is there a container in each Kubernetes pod with 0% CPU and minimal disk/ram?
These are pause containers (docker_image:gcr.io/google_containers/pause.*) that K8s injects into every pod to keep it populated even if the "real” container is restarting/stopped. 

The docker_daemon check ignores them through a default exclusion list, but they will show up for K8s metrics like *kubernetes.cpu.usage.total* and *kubernetes.filesystem.usage*.

## Further Reading
### Datadog Blog
To get a better idea of how (or why) to integrate your Kubernetes service, check out our [series of blog posts](https://www.datadoghq.com/blog/monitoring-kubernetes-era/) about it.

### Knowledge Base
* [How to get more out of your Kubernetes integration?](https://docs.datadoghq.com/agent/faq/how-to-get-more-out-of-your-kubernetes-integration)
* [How to report host disk metrics when dd-agent runs in a docker container?](https://docs.datadoghq.com/agent/faq/how-to-report-host-disk-metrics-when-dd-agent-runs-in-a-docker-container)
