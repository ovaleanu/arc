# Connecting a simple Kind cluster to Azure Arc

Azure Arc is part of the Microsoft strategy for hybrid cloud deployments and management. With Azure Arc you can now manage virtual machines, Kubernetes clusters, and databases as if they are running in Azure.

![](https://github.com/ovaleanujnpr/arc/blob/master/images/azure-arc-control-plane.png)

Now Azure Arc allows to manage the following resources outside of Azure:

Servers - Bare Metal or Virtual Machines running Linux or Windows
Kubernetes clusters - Multiple distributions are supported
Data Services - Azure SQL Database and PostgreSQL Hyperscale services

Reference [Azure Arc overview](https://docs.microsoft.com/en-us/azure/azure-arc/overview)

In this tutorial I will concentrate on [Arc enabled Kubernetes](https://docs.microsoft.com/en-us/azure/azure-arc/kubernetes/overview). Today Azure Arc enabled Kubernetes is in Preview.
Azure Arc enabled Kubernetes supports onboarding variuous Kubernetes distributions.

I will connect a simple Kind cluster running on my Macbook to Azure Arc.

### Prerequisites

You need the Azure CLI version 2.12.0 or later installed and configured. Run az --version to find the version. If you need to install or upgrade, see [Install the Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli).

```
$ az --version
azure-cli                         2.12.1 *

core                              2.12.1 *
telemetry                          1.0.6
```


A Kind cluster running on your Macbook. To install Kind follow the instalaltion guide from [here](https://kind.sigs.k8s.io/docs/user/quick-start/).
The kind configuration on my cluster is very simple.

```
$ cat config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  apiServerAddress: "127.0.0.1"
  apiServerPort: 6443
  podSubnet: "192.168.0.0/16"
  disableDefaultCNI: true
nodes:
- role: control-plane
- role: worker
- role: worker
```

Create the cluster using the following command:

```
$ kind create cluster --config cluster.yaml
```

After the cluster is created you can install the CNI

```
$ kubectl apply -f https://docs.projectcalico.org/v3.16/manifests/calico.yaml
```

You kind cluster should be up and running

```
$ kubectl get nodes -o wide
NAME                 STATUS   ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                                     KERNEL-VERSION     CONTAINER-RUNTIME
kind-control-plane   Ready    master   14h   v1.19.1   172.18.0.3    <none>        Ubuntu Groovy Gorilla (development branch)   4.19.76-linuxkit   containerd://1.4.0
kind-worker          Ready    <none>   14h   v1.19.1   172.18.0.2    <none>        Ubuntu Groovy Gorilla (development branch)   4.19.76-linuxkit   containerd://1.4.0
kind-worker2         Ready    <none>   14h   v1.19.1   172.18.0.4    <none>        Ubuntu Groovy Gorilla (development branch)   4.19.76-linuxkit   containerd://1.4.0

$ kubectl get pods -n kube-system
NAME                                         READY   STATUS    RESTARTS   AGE
calico-kube-controllers-7d569d95-fs6kx       1/1     Running   0          14h
calico-node-ddhfl                            1/1     Running   0          14h
calico-node-qbckv                            1/1     Running   0          14h
calico-node-tvnw9                            1/1     Running   0          14h
coredns-f9fd979d6-dtw4w                      1/1     Running   0          14h
coredns-f9fd979d6-qk48t                      1/1     Running   0          14h
etcd-kind-control-plane                      1/1     Running   0          14h
kube-apiserver-kind-control-plane            1/1     Running   0          14h
kube-controller-manager-kind-control-plane   1/1     Running   1          14h
kube-proxy-csrzt                             1/1     Running   0          14h
kube-proxy-d77g9                             1/1     Running   0          14h
kube-proxy-llsmw                             1/1     Running   0          14h
kube-scheduler-kind-control-plane            1/1     Running   1          14h
```

You will also need Helm v3 installed. For installation details, check [here](https://helm.sh/docs/intro/install/#from-homebrew-macos).

### Connect Kind to Azure Arc

To connect a Kubernetes cluster to Azure, the cluster administrator needs to deploy agents. These agents run in a Kubernetes namespace named `azure-arc` and are standard Kubernetes deployments. The agents are responsible for connectivity to Azure, collecting Azure Arc logs and metrics, and watching for configuration requests.

First you need to register Arc resource providers.

```
$ az provider register --namespace Microsoft.Kubernetes
$ az provider register --namespace Microsoft.KubernetesConfiguration
```

Check if the resources are registered

```
$ az provider show -n Microsoft.Kubernetes -o table
Namespace             RegistrationPolicy    RegistrationState
--------------------  --------------------  -------------------
Microsoft.Kubernetes  RegistrationRequired  Registered

$ az provider show -n Microsoft.KubernetesConfiguration -o table
Namespace                          RegistrationPolicy    RegistrationState
---------------------------------  --------------------  -------------------
Microsoft.KubernetesConfiguration  RegistrationRequired  Registered
```

Install az CLI extensions for Arc

```
$ az extension add --name connectedk8s
$ az extension add --name k8sconfiguration
```

Create a Service Principal to connect the cluster to Azure Arc

```
$ az ad sp create-for-rbac --role Contributor --name demo-arc
Changing "demo-arc" to a valid URI of "http://demo-arc", which is the required format used for service principal names
Creating a role assignment under the scope of "/subscriptions/f39bcbf9-3a71-414d-a17f-794d0fffe65c"
{
  "appId": ".............",
  "displayName": "demo-arc",
  "name": "http://demo-arc",
  "password": "...............",
  "tenant": "............."
}
```

Create a resource group for your cluster. Currently Arc enabled Kubernetes is available on US East and West Europe regions.

```
$ az group create -l westeurope -n ov-kind-arc
```

Export the following variables

```
$ export appId='..................'
$ export password='..................'
$ export tenantId='...............'
$ export resourceGroup='ov-kind-arc'
$ export arcClusterName='ov-kind'
```

_Note_
You have appID, password and tenantID variables when you created the Service Principal.

Connect the cluster to Azure Arc

```
$ az connectedk8s connect --name $arcClusterName --resource-group $resourceGroup
```

Azure Arc pods are running in `azure-arc` namespace

```
$ kubectl get pods --all-namespaces
NAMESPACE            NAME                                         READY   STATUS    RESTARTS   AGE
azure-arc            cluster-metadata-operator-c65fcd8c5-k855r    2/2     Running   0          14h
azure-arc            clusteridentityoperator-769b8b4b4d-vwc9x     3/3     Running   1          14h
azure-arc            config-agent-6dbbcdfb84-2gsjj                3/3     Running   0          14h
azure-arc            controller-manager-5fdc98b965-96ckn          3/3     Running   1          14h
azure-arc            flux-logs-agent-59b5f6c66f-sqdml             2/2     Running   0          14h
azure-arc            metrics-agent-5464649849-sffml               2/2     Running   0          14h
azure-arc            resource-sync-agent-54ff8dd988-rxdcq         3/3     Running   0          14h
kube-system          calico-kube-controllers-7d569d95-fs6kx       1/1     Running   0          14h
kube-system          calico-node-ddhfl                            1/1     Running   0          14h
kube-system          calico-node-qbckv                            1/1     Running   0          14h
kube-system          calico-node-tvnw9                            1/1     Running   0          14h
kube-system          coredns-f9fd979d6-dtw4w                      1/1     Running   0          14h
kube-system          coredns-f9fd979d6-qk48t                      1/1     Running   0          14h
kube-system          etcd-kind-control-plane                      1/1     Running   0          14h
kube-system          kube-apiserver-kind-control-plane            1/1     Running   0          14h
kube-system          kube-controller-manager-kind-control-plane   1/1     Running   1          14h
kube-system          kube-proxy-csrzt                             1/1     Running   0          14h
kube-system          kube-proxy-d77g9                             1/1     Running   0          14h
kube-system          kube-proxy-llsmw                             1/1     Running   0          14h
kube-system          kube-scheduler-kind-control-plane            1/1     Running   1          14h
local-path-storage   local-path-provisioner-78776bfc44-plpdd      1/1     Running   0          14h
```

The cluster is visible on Azure Arc portal as well

![](https://github.com/ovaleanujnpr/arc/blob/master/images/image1.png)

### Configure Azure Policy

To configure Azure Policy you need to install [Azure Policy Add-on for Azure Arc enabled Kubernetes](https://docs.microsoft.com/en-gb/azure/governance/policy/concepts/policy-for-kubernetes#install-azure-policy-add-on-for-azure-arc-enabled-kubernetes).

Enable the Microsoft.PolicyInsights resource provider

```
$ az provider register --namespace 'Microsoft.PolicyInsights'

$ az provider show -n Microsoft.PolicyInsights -o table
Namespace                 RegistrationPolicy    RegistrationState
------------------------  --------------------  -------------------
Microsoft.PolicyInsights  RegistrationRequired  Registered
```

Create a Service Principal

```
$ az ad sp create-for-rbac --role "Policy Insights Data Writer (Preview)" --scopes "/subscriptions/<subscriptionId>/resourceGroups/ov-kind-arc/providers/Microsoft.Kubernetes/connectedClusters/ov-kind"
```

_Note_ Replace <subscriptionId> with your subscription ID

Add the Azure Policy Add-on repo to Helm:

```
$ helm repo add azure-policy https://raw.githubusercontent.com/Azure/azure-policy/master/extensions/policy-addon-kubernetes/helm-charts
```

Install the Azure Policy Add-on using Helm Chart:

```
$ helm install azure-policy-addon azure-policy/azure-policy-addon-arc-clusters \
    --set azurepolicy.env.resourceid=/subscriptions/<subscriptionId>/resourceGroups/ov-kind-arc/providers/Microsoft.Kubernetes/connectedClusters/ov-kind \
    --set azurepolicy.env.clientid=<ServicePrincipalAppId> \
    --set azurepolicy.env.clientsecret=<ServicePrincipalPassword> \
    --set azurepolicy.env.tenantid=<ServicePrincipalTenantId>

_Note_ Replace <ServicePrincipalAppId>, <ServicePrincipalPassword> and <ServicePrincipalTenantId> with the values from the output of creating the Service Principal previously
```

Check if installation was successful and that the azure-policy and gatekeeper pods are running

```
$ kubectl get pods -n kube-system
NAME                                         READY   STATUS    RESTARTS   AGE
azure-policy-798977b44d-9w4sj                1/1     Running   0          3h14m
azure-policy-webhook-685d956678-tmhzx        1/1     Running   0          3h14m
calico-kube-controllers-7d569d95-fs6kx       1/1     Running   0          18h
calico-node-ddhfl                            1/1     Running   0          18h
calico-node-qbckv                            1/1     Running   0          18h
calico-node-tvnw9                            1/1     Running   0          18h
coredns-f9fd979d6-dtw4w                      1/1     Running   0          18h
coredns-f9fd979d6-qk48t                      1/1     Running   0          18h
etcd-kind-control-plane                      1/1     Running   0          18h
kube-apiserver-kind-control-plane            1/1     Running   0          18h
kube-controller-manager-kind-control-plane   1/1     Running   1          18h
kube-proxy-csrzt                             1/1     Running   0          18h
kube-proxy-d77g9                             1/1     Running   0          18h
kube-proxy-llsmw                             1/1     Running   0          18h
kube-scheduler-kind-control-plane            1/1     Running   1          18h

$ kubectl get pods -n gatekeeper-system
NAME                                            READY   STATUS    RESTARTS   AGE
gatekeeper-controller-manager-76cdc6f4c-4rn7p   1/1     Running   1          3h15m
```

Assign the 'Kubernetes cluster pod security baseline standards for Linux-based workloads' built-in initiative following these [steps](https://docs.microsoft.com/en-gb/azure/governance/policy/concepts/policy-for-kubernetes#assign-a-built-in-policy-definition).

After aproximately 20 minutes the initiative is synced with the cluster

```
kubectl get constrainttemplate
NAME                                     AGE
k8sazureallowedcapabilities              30m
k8sazureblockhostnamespace               30m
k8sazurecontainernoprivilege             30m
k8sazurehostfilesystem                   30m
k8sazurehostnetworkingports              30m
```

![](https://github.com/ovaleanujnpr/arc/blob/master/images/image2.png)

![](https://github.com/ovaleanujnpr/arc/blob/master/images/image3.png)



This initiative has a policy not to allow running priviledge pods. You can validate this running the following priviledge pod.

```
$ cat nginx-priviledge.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-privileged
spec:
  containers:
    - name: nginx-privileged
      image: nginx
      securityContext:
        privileged: true

$ kubectl apply -f nginx-priviledge.yaml
```

The pod should not be able to be created due to the policy enforcement.

_Note_ If your cluster is 1.19, this version is not supported yet https://github.com/Azure/AKS/issues/1869.
