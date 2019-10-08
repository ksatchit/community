# LITMUS CHAOS OPERATOR WORKFLOW DEMO STEPS

## Pre-Requisites

- Ensure you are in the admin context in order to setup RBAC for the various components involved in the demo

*Note: This demo steps were carried out on GKE (v1.12.6-gke.10) platform*

## Demo Steps

### Install Litmus Infra Components 

- This step performs the Litmus RBAC setup, deploys the chaos operator & installs the required CRDs. 

*Note: Ensure that the namespace selected is "litmus"* 

```
root@chaos-go:~# root@chaos-go:~# kubectl apply -f https://litmuschaos.github.io/pages/litmus-operator-latest.yaml

namespace/litmus created
serviceaccount/litmus created
clusterrole.rbac.authorization.k8s.io/litmus created
clusterrolebinding.rbac.authorization.k8s.io/litmus created
deployment.apps/chaos-operator-ce created
customresourcedefinition.apiextensions.k8s.io/chaosengines.litmuschaos.io created
customresourcedefinition.apiextensions.k8s.io/chaosexperiments.litmuschaos.io created
customresourcedefinition.apiextensions.k8s.io/chaosresults.litmuschaos.io created
```

- Verify that the litmus service account, clusterrole, clusterrolebinding, litmus chaos operator deployment & litmuschaos CRDs are applied

```
root@chaos-go:~# kubectl get sa litmus -n litmus 
NAME     SECRETS   AGE
litmus   1         108s

root@chaos-go:~# kubectl get clusterroles,clusterrolebinding,crds | grep "litmus\|chaos"

clusterrole.rbac.authorization.k8s.io/litmus                                          2m26s

clusterrolebinding.rbac.authorization.k8s.io/litmus                                   2m26s

customresourcedefinition.apiextensions.k8s.io/chaosengines.litmuschaos.io             2019-07-31T03:10:24Z
customresourcedefinition.apiextensions.k8s.io/chaosresults.litmuschaos.io             2019-07-31T03:10:25Z
customresourcedefinition.apiextensions.k8s.io/litmusresults.litmus.io                 2019-07-31T03:10:26Z

root@chaos-go:~# kubectl get deploy -n litmus 
NAME     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
litmus   1         1         1            1           4m5s
root@chaos-go:~# 

oot@chaos-go:~# kubectl get pods -n litmus 
NAME                                 READY   STATUS    RESTARTS   AGE
chaos-operator-ce-777fdfcf5f-hv5xz   1/1     Running   0          49s
```

### Install an example application deployment

- This deployment will be used as the application under test (AUT). Lets deploy, say, nginx with 2 replicas

```
root@chaos-go:~# kubectl apply -f ./feature-demos/chaos-operator-workflow/nginx/nginx.yaml 
deployment.apps/nginx-deployment created

root@chaos-go:~# kubectl get pods 
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-5c689d88bb-rk4xn   1/1     Running   0          16s
nginx-deployment-5c689d88bb-txxbb   1/1     Running   0          16s
```

### Annotate the application deployment for chaos

```
root@chaos-go:~# kubectl annotate deploy nginx-deployment litmuschaos.io/chaos="true"
deployment.extensions/nginx-deployment annotated
```
### Create a chaos service account in the application's namespace

- This is the service account that will be used by the chaos executor. The permissions can be set on the basis of chaos desired
  by the users. 

  ```
  root@chaos-go:~# kubectl apply -f ./feature-demos/chaos-operator-workflow/nginx/rbac.yaml 
  serviceaccount/nginx created
  role.rbac.authorization.k8s.io/nginx created
  rolebinding.rbac.authorization.k8s.io/nginx created
  ```

### Install the genric (kubernetes) chaos chart to install the experiment CRs 

```
root@chaos-go:~#  kubectl create -f https://raw.githubusercontent.com/litmuschaos/chaos-charts/master/charts/generic/experiment.yaml -n litmus

chaosexperiment.litmuschaos.io/container-kill created
chaosexperiment.litmuschaos.io/pod-delete created
chaosexperiment.litmuschaos.io/pod-network-delay created
```

### Examine the experiment CRs to view the chaos params

- Use `kubectl get chaosexperiment <name> -o yaml` to view the CRs. Below snippets are formatted for readability

  *Note: The tunables in the experiments such as LIB, KILL_MODE are placeholders and are WIP* 

```yaml
apiVersion: litmuschaos.io/v1alpha1
description:
  message: |
    Deletes a pod belonging to a deployment/statefulset/daemonset
kind: ChaosExperiment
metadata:
  labels:
    litmuschaos.io/name: kubernetes
  name: pod-delete
  namespace: default
spec:
  definition:
    args:
    - -c
    - ansible-playbook ./experiments/chaos/kubernetes/pod_delete/test.yml -i /etc/ansible/hosts
      -vv; exit 0
    command:
    - /bin/bash
    env:
    - name: ANSIBLE_STDOUT_CALLBACK
      value: default
    - name: TOTAL_CHAOS_DURATION
      value: 15
    - name: CHAOS_INTERVAL
      value: 5
    - name: LIB
      value: "" 
    image: ""
    labels:
      name: pod-delete
    litmusbook: /experiments/chaos/kubernetes/pod_delete/run_litmus_test.yml
```

```yaml
apiVersion: litmuschaos.io/v1alpha1
description:
  message: "Kills a container belonging to an application pod \n"
kind: ChaosExperiment
metadata:
  labels:
    litmuschaos.io/name: kubernetes
  name: container-kill
  namespace: default
spec:
  definition:
    args:
    - -c
    - ansible-playbook ./experiments/chaos/kubernetes/container_kill/test.yml -i /etc/ansible/hosts
      -vv; exit 0
    command:
    - /bin/bash
    env:
    - name: ANSIBLE_STDOUT_CALLBACK
      value: default
    - name: TARGET_CONTAINER
      value: nginx
    - name: KILL_MODE
      value: ""
    - name: LIB
      value: ""
    image: ""
    labels:
      name: container-kill
    litmusbook: /experiments/chaos/kubernetes/container_kill/run_litmus_test.yml
```

### Create the ChaosEngine resource with the above two experiments listed

- This is a user-facing CR which lists experiments to be performed.

```yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: engine-nginx
spec:
  appinfo: 
    appns: default
    applabel: "app=nginx"
    appkind: deployment
  chaosServiceAccount: nginx
  experiments:
    - name: pod-delete 
      spec:
        components: 
    - name: container-kill
      spec:
        components:
```

```
root@chaos-go:~# kubectl apply -f ./feature-demos/chaos-operator-workflow/chaosengine.yaml 
chaosengine.litmuschaos.io/engine-nginx created
```

### Reconcile: Verify that the Engine Runner Pod & Monitor services are created 

- Verify Secondary resource creation. The engine runner pod spawns the chaos jobs (litmusbooks) listed in
  the chaosengine & monitor service exposes the chaos metrics.

```
root@chaos-go:~# kubectl get pods 
NAME                                READY   STATUS    RESTARTS   AGE
engine-nginx-runner                 2/2     Running   0          17s
nginx-deployment-5c689d88bb-rk4xn   1/1     Running   0          100m
nginx-deployment-5c689d88bb-txxbb   1/1     Running   0          100m

root@chaos-go:~# kubectl get svc
NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
engine-nginx-monitor   ClusterIP   10.86.11.173   <none>        8080/TCP   22s
kubernetes             ClusterIP   10.86.0.1      <none>        443/TCP    179m

```

### View execution of chaos experiments 

- In this demo, the a pod-delete experiment is executed followed by a container-kill experiment. 

  *Note: The duration of these tests are kept at a minimum. It is recommended to keep another terminal with a watch 
   created on the app & litmus pods to see the experiments in progress*

```
Every 1.0s: kubectl get pods                                                                                                                

NAME                                READY   STATUS        RESTARTS   AGE         1/1     Running       0          14s
engine-nginx-runner                 2/2     Running       0          35s
nginx-deployment-5c689d88bb-cd7x5   1/1     Running       0          2s
nginx-deployment-5c689d88bb-g4qjn   1/1     Running       1          3m26s
nginx-deployment-5c689d88bb-vm24q   0/1     Terminating   2          6m28s
pod-delete-sz6m4-5flpt              1/1     Running       0          44s
``` 

```
Every 1.0s: kubectl get pods                                                                                                                

NAME                                READY   STATUS      RESTARTS   AGE
container-kill-lzf62-kxz7c          1/1     Running     0          24s
engine-nginx-runner                 2/2     Running     0          2m14s
nginx-deployment-5c689d88bb-fhsrl   1/1     Running     0          62s
nginx-deployment-5c689d88bb-mhzwq   1/1     error       1          72s
pod-delete-fklf8-45g7v              0/1     Completed   0          93s
pumba-pzdz4                         1/1     Running     0          11s
pumba-rkx62                         1/1     Running     0          15s
```

### View Results of experiments 

- Verify Chaos experiments are executed successfully by looking up the chaosresult CRs. Below snippets are formatted for readability

*Note: It is recommended to keep another terminal with a watch created on the app & litmus pods 
to see the experiments in progress*

```yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosResult
metadata:
  name: engine-nginx-pod-delete
  namespace: default
spec:
  experimentstatus:
    phase: null
    verdict: pass
```

```yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosResult
metadata:
  name: engine-nginx-container-kill
  namespace: default
spec:
  experimentstatus:
    phase: null
    verdict: pass
```

### Monitor the Chaos Metrics via Prometheus 

- Install the prometheus artifacts in ./feature-demos/chaos-operator-workflow/prometheus

*Note: Ensure that the target IP in the Prometheus configmap is set to the EngineMonitor Service ClusterIP*

```
root@chaos-go:~# kubectl apply -f ./feature-demos/chaos-operator-workflow/prometheus/prom-namespace.yaml 
namespace/monitoring created

root@chaos-go:~# kubectl apply -f ./feature-demos/chaos-operator-workflow/prometheus/prom-clusterrole.yaml 
clusterrole.rbac.authorization.k8s.io/prometheus created
clusterrolebinding.rbac.authorization.k8s.io/prometheus created

root@chaos-go:~# kubectl create configmap prometheus-config --from-file=./feature-demos/chaos-operator-workflow/prometheus/prom-config.yaml -n monitoring

root@chaos-go:~# kubectl apply -f ./feature-demos/chaos-operator-workflow/prometheus/prom-deployment.yaml -n monitoring
deployment.extensions/prometheus-deployment created

root@chaos-go:~# kubectl apply -f ./feature-demos/chaos-operator-workflow/prometheus/prom-service.yaml -n monitoring
service/prometheus-service created
```

- Access the prometheus dashboard by accesing the *nodeIP:30000* from your browser & execute the prom queries to see the
  chaos metrics graphs

  ![chaos-metrics](/feature-demos/chaos-operator-workflow/images/chaos-metrics-prometheus.jpg)
