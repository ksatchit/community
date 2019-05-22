# LITMUS CHAOS OPERATOR WORKFLOW DEMO STEPS

## Pre-Requisites

- Ensure you are in the admin context in order to setup RBAC for the various components involved in the demo
- Ensure you have setup Helm (with Tiller) on your Kubernetes Cluster. If not already installed, please follow
  the steps at: https://github.com/litmuschaos/chaos-charts/blob/master/README.md
- Ensure remove any prior instance of litmuschaos CRDs (chaosengine, chaosexperiment, chaosreults, litmusresults) 
  if installed. Else, they may cause some of the helm commands listed below to fail. However, a re-run of the
  failed commands should still work

*Note: This demo steps were carried out on GKE (v1.12.6-gke.10) platform*

## Demo Steps

### Setup the Litmuschaos Helm Repository

- Add the remote helm repository for Litmuschaos

```
root@chaos-go:~# helm repo add chaos-charts https://litmuschaos.github.io/chaos-charts
"chaos-charts" has been added to your repositories
```

- Ensure that the helm repository is successfully added & is the latest

```
root@chaos-go:~# helm repo list
NAME            URL                                             
stable          https://kubernetes-charts.storage.googleapis.com
local           http://127.0.0.1:8879/charts                    
chaos-charts    https://litmuschaos.github.io/chaos-charts     

root@chaos-go:~# helm repo update chaos-charts
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "chaos-charts" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete.

root@chaos-go:~# helm search chaos-charts
NAME                            CHART VERSION   APP VERSION     DESCRIPTION                                                 
chaos-charts/chaosoperator      0.1.0           1.0             A Helm chart to install litmus chaos operator on Kubernetes 
chaos-charts/k8schaos           0.1.0           1.0             Helm chart for Litmus Kubernetes Chaos Experiments          
chaos-charts/litmusinfra        0.1.0           1.0             A Helm chart to install litmus infra components on Kubern...
```

### Install Litmus Infra Components 

- This step performs the Litmus RBAC setup & applies the chaosresults & litmusresults CRDs. 

*Note: Ensure that the namespace selected is "litmus"* 

```
root@chaos-go:~# helm install chaos-charts/litmusInfra --namespace=litmus
NAME:   wandering-uakari
LAST DEPLOYED: Wed May 22 11:17:23 2019
NAMESPACE: litmus
STATUS: DEPLOYED

RESOURCES:
==> v1/ServiceAccount
NAME    SECRETS  AGE
litmus  1        0s

==> v1beta1/ClusterRole
NAME    AGE
litmus  0s

==> v1beta1/ClusterRoleBinding
NAME    AGE
litmus  0s
```

- Verify that the litmus service account, clusterrole, clusterrolebinding & result CRDs are applied

```
root@chaos-go:~# kubectl get sa litmus -n litmus 
NAME     SECRETS   AGE
litmus   1         5m51s

root@chaos-go:~# kubectl get clusterroles,clusterrolebinding,crds | grep litmus 
clusterrole.rbac.authorization.k8s.io/litmus                                                        4m20s
clusterrolebinding.rbac.authorization.k8s.io/litmus                                                 4m20s
customresourcedefinition.apiextensions.k8s.io/chaosresults.litmuschaos.io                           2019-05-22T11:17:23Z
customresourcedefinition.apiextensions.k8s.io/litmusresults.litmus.io 				    2019-05-22T11:17:24Z
```

### Install Litmus Chaos Operator

- Install the helm chart for Litmus Chaos Operator. 

```
root@chaos-go:~# helm install chaos-charts/chaosOperator
NAME:   solemn-manta
LAST DEPLOYED: Wed May 22 11:25:49 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Deployment
NAME            READY  UP-TO-DATE  AVAILABLE  AGE
chaos-operator  0/1    1           0          0s

==> v1/Pod(related)
NAME                             READY  STATUS             RESTARTS  AGE
chaos-operator-69f7dd446f-g2knp  0/1    ContainerCreating  0         0s

==> v1/ServiceAccount
NAME            SECRETS  AGE
chaos-operator  1        0s

==> v1beta1/ClusterRole
NAME            AGE
chaos-operator  0s

==> v1beta1/ClusterRoleBinding
NAME            AGE
chaos-operator  0s
```

- Verify that the Chaos Operator deployment, service & RBAC components are setup successfully

```
root@chaos-go:~# kubectl get deploy,svc,sa,clusterrole,clusterrolebinding chaos-operator
NAME                                   DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/chaos-operator   1         1         1            1           4m38s

NAME                     TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
service/chaos-operator   ClusterIP   10.23.252.3   <none>        8383/TCP   4m32s

NAME                            SECRETS   AGE
serviceaccount/chaos-operator   1         4m38s

NAME                                                   AGE
clusterrole.rbac.authorization.k8s.io/chaos-operator   4m38s

NAME                                                          AGE
clusterrolebinding.rbac.authorization.k8s.io/chaos-operator   4m38s
```

### Install an example application deployment

- This deployment will be used as the application under test (AUT). Lets deploy, say, nginx with 2 replicas

```
root@chaos-go:~# kubectl apply -f ./feature-demos/chaos-operator-workflow-22May2019/nginx.yaml 
deployment.apps/nginx-deployment created
  
root@chaos-go:~# kubectl get pods 
NAME                                READY   STATUS    RESTARTS   AGE
chaos-operator-69f7dd446f-g2knp     1/1     Running   0          11m
nginx-deployment-5c689d88bb-bc4kz   1/1     Running   0          3m45s
nginx-deployment-5c689d88bb-gpvbn   1/1     Running   0          3m45s
```

### Annotate the application deployment for chaos

```
root@chaos-go:~# kubectl annotate deploy nginx-deployment litmuschaos.io/chaos="true"
deployment.extensions/nginx-deployment annotated
```

### Install the ChaosExperiment helm Charts (k8sChaos) to install the experiment CRs 

```

root@chaos-go:~# helm install chaos-charts/k8sChaos
NAME:   opulent-gnat
LAST DEPLOYED: Wed May 22 11:54:45 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1alpha1/ChaosExperiment
NAME            AGE
container-kill  0s
pod-delete      0s

NOTES:
##TODO: Describe helpful chaos-related kubectl commands constructed using templates
```

### Create the ChaosEngine resource with the above two experiments listed

```
root@chaos-go:~# kubectl apply -f ./feature-demos/chaos-operator-workflow-22May2019/chaosengine.yaml 
chaosengine.litmuschaos.io/engine-nginx created
```

### Reconcile: Verify that the Engine Runner Pod & Monitor services are created & chaos experiments executed

- Verify Secondary resource creation

```
root@chaos-go:~# kubectl get pods 
NAME                                READY   STATUS    RESTARTS   AGE
chaos-operator-69f7dd446f-g2knp     1/1     Running   0          35m
engine-nginx-runner                 2/2     Running   0          7s
nginx-deployment-5c689d88bb-bc4kz   1/1     Running   0          27m
nginx-deployment-5c689d88bb-gpvbn   1/1     Running   0          27m

root@chaos-go:~# kubectl get svc
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
chaos-operator         ClusterIP   10.23.252.3     <none>        8383/TCP   38m
engine-nginx-monitor   ClusterIP   10.23.250.214   <none>        8080/TCP   3m23s
kubernetes             ClusterIP   10.23.240.1     <none>        443/TCP    35d
```

- Verify Chaos experiments are executed successfully by looking up the chaosresult CRs

*Note: It is recommended to keep another terminal with a watch created on the app & litmus pods 
to see the experiments in progress*

```
root@chaos-go:~# kubectl describe chaosresult engine-nginx-pod-delete
Name:         engine-nginx-pod-delete
Namespace:    default
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"litmuschaos.io/v1alpha1","kind":"ChaosResult","metadata":{"annotations":{},"name":"engine-nginx-pod-delete","namespace":"de...
API Version:  litmuschaos.io/v1alpha1
Kind:         ChaosResult
Metadata:
  Creation Timestamp:  2019-05-22T12:10:19Z
  Generation:          9
  Resource Version:    8898730
  Self Link:           /apis/litmuschaos.io/v1alpha1/namespaces/default/chaosresults/engine-nginx-pod-delete
  UID:                 911ada69-7c8a-11e9-b37f-42010a80019f
Spec:
  Experimentstatus:
    Phase:    <nil>
    Verdict:  pass
Events:       <none>
```


```
root@chaos-go:~# kubectl describe chaosresult engine-nginx-container-kill 
Name:         engine-nginx-container-kill
Namespace:    default
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"litmuschaos.io/v1alpha1","kind":"ChaosResult","metadata":{"annotations":{},"name":"engine-nginx-container-kill","namespace"...
API Version:  litmuschaos.io/v1alpha1
Kind:         ChaosResult
Metadata:
  Creation Timestamp:  2019-05-22T12:12:36Z
  Generation:          4
  Resource Version:    8898368
  Self Link:           /apis/litmuschaos.io/v1alpha1/namespaces/default/chaosresults/engine-nginx-container-kill
  UID:                 e31b48ca-7c8a-11e9-b37f-42010a80019f
Spec:
  Experimentstatus:
    Phase:    <nil>
    Verdict:  pass
Events:       <none>
```

### Monitor the Chaos Metrics via Prometheus 

- Install the prometheus artifacts in ./feature-demos/chaos-operator-workflow-22May2019/prometheus

*Note: Ensure that the target IP in the Prometheus configmap is set to the EngineMonitor Service ClusterIP*

```
root@chaos-go:~# kubectl apply -f ./feature-demos/chaos-operator-workflow-22May2019/prometheus/prom-namespace.yaml 
namespace/monitoring created

root@chaos-go:~# kubectl apply -f .feature-demos/chaos-operator-workflow-22May2019/prometheus/prom-clusterrole.yaml 
clusterrole.rbac.authorization.k8s.io/prometheus created
clusterrolebinding.rbac.authorization.k8s.io/prometheus created

root@chaos-go:~# kubectl create configmap prometheus-config --from-file=./feature-demos/chaos-operator-workflow-22May2019/prometheus/prom-config.yaml -n monitoring

root@chaos-go:~# kubectl apply -f ./feature-demos/chaos-operator-workflow-22May2019/pro
metheus/prom-deployment.yaml -n monitoring
deployment.extensions/prometheus-deployment created

root@chaos-go:~# kubectl apply -f ./feature-demos/chaos-operator-workflow-22May2019/prometheus/prom-service.yaml -n monitoring
service/prometheus-service created
```

- Access the prometheus dashboard by accesing the *nodeIP:30000* from your browser & execute the prom queries to see the
  chaos metrics graphs

  ![chaos-metrics](/feature-demos/chaos-operator-workflow-22May2019/images/chaos-metrics.jpg)
