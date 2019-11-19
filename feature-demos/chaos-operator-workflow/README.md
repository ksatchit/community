# LITMUS CHAOS OPERATOR WORKFLOW DEMO STEPS


## Pre-Requisites

- Ensure you are in the admin context in order to setup RBAC for the various components involved in the demo

*Note: This demo steps were carried out on GKE (v1.12.6-gke.10) platform*

## Demo Steps

- In this guide, the steps to perform a simple pod-delete chaos experiment is explained. 

### Install Litmus Infra Components 

- This step performs the Litmus RBAC setup, deploys the chaos operator & installs the required CRDs. 

```
root@chaos-go:~# kubectl apply -f https://litmuschaos.github.io/pages/litmus-operator-latest.yaml

namespace/litmus created
serviceaccount/litmus created
clusterrole.rbac.authorization.k8s.io/litmus created
clusterrolebinding.rbac.authorization.k8s.io/litmus created
deployment.apps/chaos-operator-ce created
customresourcedefinition.apiextensions.k8s.io/chaosengines.litmuschaos.io created
customresourcedefinition.apiextensions.k8s.io/chaosexperiments.litmuschaos.io created
customresourcedefinition.apiextensions.k8s.io/chaosresults.litmuschaos.io created
```

- Verify that the Litmus Service Account, Clusterrole, Clusterrolebinding, Litmus Chaos Operator Deployment & Litmuschaos CRDs are applied

```
root@chaos-go:~# kubectl get sa litmus -n litmus 
NAME     SECRETS   AGE
litmus   1         108s

root@chaos-go:~# kubectl get clusterroles,clusterrolebinding,crds | grep "litmus\|chaos"

clusterrole.rbac.authorization.k8s.io/litmus                                          2m26s
clusterrolebinding.rbac.authorization.k8s.io/litmus                                   2m26s
customresourcedefinition.apiextensions.k8s.io/chaosengines.litmuschaos.io             2019-07-31T03:10:24Z
customresourcedefinition.apiextensions.k8s.io/chaosexperiments.litmuschaos.io         2019-07-31T03:10:25Z
customresourcedefinition.apiextensions.k8s.io/chaosresults.litmus.io                  2019-07-31T03:10:26Z

root@chaos-go:~# kubectl get pods -n litmus 
NAME                                 READY   STATUS    RESTARTS   AGE
chaos-operator-ce-777fdfcf5f-hv5xz   1/1     Running   0          49s
```

### Install an Example Application Deployment

- This deployment will be used as the application under test (AUT), that will be subject to chaos.  
  Lets deploy, say, nginx with 2 replicas

```
root@chaos-go:~# kubectl apply -f https://raw.githubusercontent.com/litmuschaos/community/master/feature-demos/chaos-operator-workflow/nginx/nginx.yaml
deployment.apps/nginx-deployment created

root@chaos-go:~# kubectl get pods 
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-5c689d88bb-rk4xn   1/1     Running   0          16s
nginx-deployment-5c689d88bb-txxbb   1/1     Running   0          16s
```

### Install the pod-delete chaos experiment CR

```
root@chaos-go:~#  kubectl apply -f https://raw.githubusercontent.com/litmuschaos/chaos-charts/master/charts/generic/pod-delete/experiment.yaml 

chaosexperiment.litmuschaos.io/pod-delete created
```

### Examine the Experiment CRs to View the Chaos Parameters

- Use `kubectl get chaosexperiment pod-delete -o yaml` to view the experiment spec in detail. Below snippets are formatted for readability


```yaml
apiVersion: litmuschaos.io/v1alpha1
description:
  message: |
    Deletes a pod belonging to a deployment/statefulset/daemonset
kind: ChaosExperiment
metadata:
  name: pod-delete
  version: 0.1.4
spec:
  definition:
    permissions:
      apiGroups:
        - ""
        - "extensions"
        - "apps"
        - "batch"
        - "litmuschaos.io"
      resources:
        - "daemonsets"
        - "deployments"
        - "statefulsets"
        - "jobs"
        - "pods"
        - "chaosengines"
        - "chaosexperiments"
        - "chaosresults"
      verbs:
        - "*"
    image: "litmuschaos/ansible-runner:ci"
    args:
    - -c
    - ansible-playbook ./experiments/generic/pod_delete/pod_delete_ansible_logic.yml -i /etc/ansible/hosts -vv; exit 0
    command:
    - /bin/bash
    env:

    - name: ANSIBLE_STDOUT_CALLBACK
      value: 'default'

    - name: TOTAL_CHAOS_DURATION
      value: '15'

    - name: FORCE
      value: 'true'

    - name: CHAOS_INTERVAL
      value: '5'

    - name: LIB
      value: ''    
    labels:
      name: pod-delete
```

### Create a Chaos Service Account in the Application's Namespace

- This is the service account that will be used by the chaos executor pods. Typically, the permissions to be associated with this 
  serviceaccount can be gauged by the `spec.permissions` section in the experiment CR. 

  ```
  root@chaos-go:~# kubectl apply -f https://raw.githubusercontent.com/litmuschaos/community/master/feature-demos/chaos-operator-workflow/nginx/rbac.yaml
  serviceaccount/nginx created
  role.rbac.authorization.k8s.io/nginx created
  rolebinding.rbac.authorization.k8s.io/nginx created
  ```

### Annotate the Application Deployment for Chaos

- This is necessary to ensure that the chaos operator identifie this deployment as a candidate for chaos. 

```
root@chaos-go:~# kubectl annotate deploy nginx-deployment litmuschaos.io/chaos="true"
deployment.extensions/nginx-deployment annotated
```

### Prepare the ChaosEngine Resource with the Pod-Delete Experiment

- This is a user-facing CR which maps the experiment to be performed with the nginx application, while also allowing to
  override some of the experiment tunables. In this case, we shall perform a pod-delete using a graceful pod termination policy. 

```yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: chaos-nginx
spec:
  monitoring: false
  jobCleanUpPolicy: delete
  appinfo: 
    appns: default
    applabel: 'app=nginx'
    appkind: deployment
  chaosServiceAccount: nginx
  experiments:
    - name: pod-delete 
      spec:
        components: 
          - name: FORCE
            value: 'false'
```

### Execute the Experiment: Verify that the Chaos Runner Pod is created

- Install the ChaosEngine CR

```
root@chaos-go:~# kubectl apply -f https://raw.githubusercontent.com/litmuschaos/community/master/feature-demos/chaos-operator-workflow/chaosengine.yaml  
chaosengine.litmuschaos.io/chaos-nginx created
```

- Verify that a chaos-runner pod (in this case, named `chaos-nginx-runner`) is created that in turn spawns the chaos experiment pod
  corresponding to experiment mentioned in the chaosengine 

```
root@chaos-go:~# kubectl get pods 
NAME                                READY   STATUS    RESTARTS   AGE
chaos-nginx-runner                  2/2     Running   0          17s
nginx-deployment-5c689d88bb-rk4xn   1/1     Running   0          100m
nginx-deployment-5c689d88bb-txxbb   1/1     Running   0          100m

```

### View Execution of the Pod-Delete Chaos Experiment

- Over the course of execution of this experiment, the nginx replicas are randomly killed over 2-3 iterations. 
  The duration of these tests are kept at a minimum. It is recommended to keep another terminal with a watch setup on the pods in the
  app's namespace (in this case default) to see the experiment in progress. 

```
Every 1.0s: kubectl get pods                                                                                                                

NAME                                READY   STATUS        RESTARTS   AGE      
chaos-nginx-runner                  2/2     Running       0          35s
nginx-deployment-5c689d88bb-cd7x5   1/1     Running       0          2s
nginx-deployment-5c689d88bb-g4qjn   1/1     Running       1          3m26s
nginx-deployment-5c689d88bb-vm24q   0/1     Terminating   2          6m28s
pod-delete-sz6m4-5flpt              1/1     Running       0          44s
``` 

- Once the experiment execution is complete, the pod-delete job enters as well as the chaos-runner pods enter "completed" state. 

### View Results of Experiments 

- Verify Chaos experiments are executed successfully by looking up the chaosresult CRs. Below snippets are formatted for readability

*Note: It is recommended to keep another terminal with a watch created on the app & litmus pods 
to see the experiments in progress*

```yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosResult
metadata:
  name: chaos-nginx-pod-delete
  namespace: default
spec:
  experimentstatus:
    phase: null
    verdict: pass
```

### Cleanup the Chaos Residue

- Remove/Delete the ChaosEngine to delete the chaos-nginx-runner pod. The chaosresult CR (chaos-nginx-pod-delete) continues to remain for future reference 

  ```
  kubectl delete -f https://raw.githubusercontent.com/litmuschaos/community/master/feature-demos/chaos-operator-workflow/chaosengine.yaml 
  ```



