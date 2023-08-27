# ZabbixMonitoringK8s
I added my kubernetes cluster to my zabbix server.  Monitoring a K8S cluster is a new feature built into zabbix 6.0. 
I followed the [User guide to setting up Zabbix monitoring of the Kubernetes](https://www.zabbix.com/integrations/kubernetes), selecting version 6.0.

The following steps add a zabbix agent to each of the nodes, as daemon sets.  Also a zabbix proxy is added to the cluster; the agents talk to the zabbix server through the proxy. 

## Setup helm
THe basic helm setup is pretty easy.  The key thing is to edit the zabbix_values.yaml file to point to the zabbix server. In my case it is linode3.kozik.net
```

[jkozik@dell2 ~]$ helm repo add zabbix-chart-6.0  https://cdn.zabbix.com/zabbix/integrations/kubernetes-helm/6.0
"zabbix-chart-6.0" has been added to your repositories
[jkozik@dell2 ~]$ helm show values zabbix-chart-6.0/zabbix-helm-chrt > $HOME/zabbix_values.yaml

[jkozik@dell2 ~]$ helm version
version.BuildInfo{Version:"v3.6.3", GitCommit:"d506314abfb5d21419df8c7e7e68012379db2354", GitTreeState:"clean", GoVersion:"go1.16.5"}

[jkozik@dell2 ~]$ vi zabbix_values.yaml
      ZBX_SERVER_HOST:      linode3.kozik.net

```

## install Zabbix Helm Chart
The helm command pulls in the zabbix_values.yaml file and creates the Zabbix monitoring pods for the cluster.  Note: the installation wants ins own namespace, monitoring in my case.
```
[jkozik@dell2 ~]$ kubectl create namespace monitoring
namespace/monitoring created

[jkozik@dell2 ~]$ helm install zabbix zabbix-chart-6.0/zabbix-helm-chrt --dependency-update -f $HOME/zabbix_values.yaml -n monitoring
NAME: zabbix
LAST DEPLOYED: Wed Aug 23 22:12:57 2023
NAMESPACE: monitoring
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Thank you for installing zabbix-helm-chrt.

Your release is named zabbix.
Zabbix agent installed:  "zabbix/zabbix-agent2:alpine-6.0.19"
Zabbix proxy installed:  "zabbix/zabbix-proxy-sqlite3:alpine-6.0.19"

Annotations:
app.kubernetes.io/name: zabbix-zabbix-helm-chrt
helm.sh/chart: zabbix-helm-chrt-1.3.3
app.kubernetes.io/version: "6.0.19"
app.kubernetes.io/managed-by: Helm


Service account created:
    zabbix-service-account

To learn more about the release, try:

  $ helkm status zabbix -n monitoring
  $ helm get all zabbix -n monitoring
[jkozik@dell2 ~]$

[jkozik@dell2 ~]$ kubectl -nmonitoring get pods
NAME                                         READY   STATUS    RESTARTS   AGE
zabbix-agent-bvtgl                           1/1     Running   0          3d16h
zabbix-agent-jqj4g                           1/1     Running   0          3d16h
zabbix-agent-qwbdt                           1/1     Running   0          3d16h
zabbix-kube-state-metrics-8474985c46-6r9jt   1/1     Running   0          3d16h
zabbix-proxy-659685cbdf-9c9dd                1/1     Running   0          3d16h
[jkozik@dell2 ~]$

```
The agent pods are in a DaemonSet with one on each node.  The proxy is running on a pod on any node. The kube-state pod is the same module used by Promethieus. 

```
[jkozik@dell2 ~]$ kubectl -nmonitoring logs pod/zabbix-proxy-659685cbdf-9c9dd
170:20230824:031341.002 cannot send proxy data to server at "linode3.kozik.net": proxy "zabbix-proxy" not found
170:20230824:031342.033 cannot send proxy data to server at "linode3.kozik.net": proxy "zabbix-proxy" not found
170:20230824:031343.078 cannot send proxy data to server at "linode3.kozik.net": proxy "zabbix-proxy" not found
...
```
It is useful to verify that the proxy is basically cycling by checking the log file.  Since the server has not been setup, the log file will be rolling with not found errors.

## get cluster URL and access token
The zabbix server needs to have the URL and access token for the cluster.  
```
[jkozik@dell2 ~]$ kubectl get secret zabbix-service-account -n monitoring -o jsonpath={.data.token} | base64 -d
eyJhbGciOiJSUzI1NiIsImtp...

[jkozik@dell2 ~]$ kubectl curl -k https://192.168.100.172:6443/api/v1/nodes -H "Authorization: Bearer
eyJhbGciOiJSUzI1NiIsIm...

[jkozik@dell2 ~]$ kubectl get serviceaccounts -A | grep monitoring
monitoring             default                                   1         2d10h
monitoring             zabbix-agent-service-account              1         2d10h
monitoring             zabbix-kube-state-metrics                 1         2d10h
monitoring             zabbix-service-account                    1         2d10h

[jkozik@dell2 ~]$ kubectl cluster-info
Kubernetes control plane is running at https://192.168.100.172:6443
CoreDNS is running at https://192.168.100.172:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
[jkozik@dell2 ~]$
```
Key fields:  eyJhbGciOiJSUzI1NiIsIm... and https://192.168.100.172:6443

For me the access token was 952 characters.

## Add the proxy to the Zabbix Server
Following the steps in the [Add the Proxy](https://blog.zabbix.com/monitoring-kubernetes-with-zabbix/25055/#add-the-proxy) section, go into the zabbix dashboard, Administration->Proxies. Add a proxy called zabbix-proxy (name was specified in the helm chart values file.

![](images/AddZabbixProxy082723.png)


## Create host named Kubernetes Nodes
Go to the zabbix template repository and download the template [Kubernetes nodes by HTTP](https://git.zabbix.com/projects/ZBX/repos/zabbix/browse/templates/app/kubernetes_http/kubernetes_nodes_http/template_kubernetes_nodes.yaml).  Be careful to select the one for the correct version. Download this file and import it at Configuration->Templates->Import Templates.
### Import Temeplate Kubernetes nodes by HTTP
![](images/ImportKubernetesNodeTemplate082723.png)

Note:  I had an initial error message on this import.  I first needed to update the template "Linux by zabbix agent" to the current release. 

### Create Kubernetes Nodes host
On the zabbix server create a new host called Kubernetes Nodes, at Configuration->Hosts->Create Host menu. Use the Template Kubernetes nodes by HTTP and access the host through the proxy zabbix-proxy. 

![](images/CreateKubernetesNodesHost082723.png)


### Set URL and Token values
The proxy needs to know the URL to the cluster and the access token.  Set the MACROS $KUBE.API.TOKEN and 
$KUBE.API.URL
![](images/SetTokenValues082723)









