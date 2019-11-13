# KAFKA BROKER KILL EXPERIMENT DEMO STEPS

## Pre-Requisites

- Ensure you are in the kubernetes-admin context in order to setup RBAC for the various components involved in the demo
- In this demo, we will use OpenEBS LocalPVs as the persistent storage for the kafka broker and zookeeper clusters. 
  Ensure that you have installed OpenEBS on the cluster. For steps to do this, refer the [OpenEBS quickstart guide](https://docs.openebs.io/docs/next/quickstart.html)
- We shall use customized confluent kafka helm charts to install the kafka cluster. Ensure that helm/tiller is already configured. For steps to do this, refer [Installing Helm](https://helm.sh/docs/intro/install/)

## Demo-Steps

Notes: 
 - These steps were performed on a Konvoy cluster created on AWS
 - In case you would like to use a different persistent storage type, please update the values.yml with appropriate storage class in zookeeper and kafka sections

### Step-1: Install Kafka Cluster 

- Clone the customized confluent-platform kafka helm chart github repository

  ```
  git clone https://github.com/litmuschaos/cp-helm-charts.git
  ```

- Create the kafka cluster (in this example, lets use the namespace: default) 

  ```
  helm install --name kafka cp-helm-charts/.
  ```

- Verify that the kafka-broker statefulset & zookeeper statefulset replicas are ready and available

  ```
  kubectl get sts kafka-cp-kafka
  ```

  ```
  kubectl get sts kafka-cp-zookeeper
  ```

### Step-2: Install the  Litmus Chaos Operator 

- Install the litmus operator with RBAC & chaos CRDs

  ```
  kubectl apply -f https://litmuschaos.github.io/pages/litmus-operator-latest.yaml
  ```

- Verify that the chaos operator is ready and available 

  ```
  kubectl get pods -n litmus 
  ```

### Step-3: Download the Kafka Broker Pod Failure Chaos Experimnent  

- The experiment CR can be obtained from the [ChaosHub](https://hub.litmuschaos.io/). 

  ```
  kubectl apply -f https://hub.litmuschaos.io/api/chaos?file=charts/kafka/kafka-broker-pod-failure/experiment.yaml
  ```

### Step-4: Create a Chaos Service Account 

- The experiments are expected to be run with a user-defined service account which has permissions necessary to execute it. 
  The permission list can be derived by the `spec.permissons` section of the experiment. 

  Note: Create the service account in the namespace where the chaos is executed/app resides. In this example, we shall use default

  ```
  kubectl apply -f https://raw.githubusercontent.com/litmuschaos/community/master/feature-demos/kafka-chaos-examples/kafka-broker-pod-failure/kafka-chaos-serviceaccount.yaml   
  ```

### Step-5: Annotate the Kafka Statefulset for Chaos

- The chaos operator looks for the chaos annotation on the specified statefulset (derived by labels and namespace)

  ```
  kubectl annotate sts/kafka-cp-kafka litmuschaos.io/chaos="true" 
  
  ```

### Step-6: Prepare the ChaosEngine manifest to hold experiment tunables

- Provide the application (kafka, zookeeper) info in the ChaosEngine spec and other tunables as experiment ENV (overrides defaults in chaosexperiment CR)

  A sample ChaosEngine, to kill the leader broker for a partition of a simple kafka topic (with an active message stream) is provided below. 

```yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: kafka-chaos
  namespace: default
spec:
  appinfo: 
    appns: default
    applabel: 'app=cp-kafka'
    appkind: statefulset
  chaosServiceAccount: kafka-sa
  monitoring: false
  jobCleanUpPolicy: delete
  experiments:
    - name: kafka-broker-pod-failure
      spec:
        components:  
          # choose based on available kafka broker replicas           
          - name: KAFKA_REPLICATION_FACTOR
            value: '3'

          # get via "kubectl get pods --show-labels -n <kafka-namespace>"
          - name: KAFKA_LABEL
            value: 'app=cp-kafka'

          - name: KAFKA_NAMESPACE
            value: 'default'
     
          # get via "kubectl get svc -n <kafka-namespace>" 
          - name: KAFKA_SERVICE
            value: 'kafka-cp-kafka-headless'

          # get via "kubectl get svc -n <kafka-namespace>  
          - name: KAFKA_PORT
            value: '9092'

          - name: ZOOKEEPER_NAMESPACE
            value: 'default'

          # get via "kubectl get pods --show-labels -n <zk-namespace>"
          - name: ZOOKEEPER_LABEL
            value: 'app=cp-zookeeper'

          # get via "kubectl get svc -n <zk-namespace>  
          - name: ZOOKEEPER_SERVICE
            value: 'kafka-cp-zookeeper-headless'

          # get via "kubectl get svc -n <zk-namespace>  
          - name: ZOOKEEPER_PORT
            value: '2181'

          # set chaos duration (in sec) as desired
          - name: TOTAL_CHAOS_DURATION
            value: '30'
``` 

### Step-7: Launch chaos experiment and verify behaviour

- Create the ChaosEngine CR to trigger the chaos experiment

  ```
  kubectl apply -f https://raw.githubusercontent.com/litmuschaos/community/master/feature-demos/kafka-chaos-examples/kafka-broker-pod-failure/chaosengine.yaml
  ```

- Watch the pods on the app (default) namespace. Look out for the following:

  - A chaos-runner is launched at the outset, which runs the kafka-broker-pod-failure job. 
  - The experiment job, as part of the experiement execution launches a liveness pod (`kafka-liveness`) that creates a continuous message stream, 
    the producer/consumer running as separate containers of the same pod. 
  - The message stream is expected to continue without interruption, with partition leaders being switched.

  ```
  watch -n 1 kubectl get pods
  ```

  - The message stream is expected to continue without interruption, with partition leaders being switched. View the kafka-liveness pod logs 
    during the course of the broker-kills to verify uninterrupted message stream

  ```
  kubectl logs -f kafka-liveness -c kafka-consumer
  ```

- The chaos experiment job will be automatically removed once the chaos execution completes


### Step-8: Verify ChaosResult 

- View the verdict of the kafka-broker-pod-failure chaos experiment to check whether the kafka cluster is resilient to the broker loss

  ```
  kubectl get chaosresult kafka-chaos-kafka-broker-pod-failure -o yaml
  ```

### Step-9: Delete the ChaosEngine to remove any chaos residue (runner pods)

- Typically, the chaos-runner enters completed state post chaos execution, while the monitor pod continues to run (in this example, monitoring is disabled)
  Removal of the ChaosEngine also deletes the runner pod. ChaosExperiment & ChaosResult CRs continue to be maintained for future use/reference.


  ```
  kubectl delete -f https://raw.githubusercontent.com/litmuschaos/community/master/feature-demos/kafka-chaos-examples/kafka-broker-pod-failure/chaosengine.yaml
  ```

### Step-10: Optional/Additional Modes for the Kafka Broker Failure Experiment

- The kafka-broker-pod-failure experiment also supports few more execution modes apart from the one employed in the demo-steps. Suppose, you already have a kafka
  cluster running 

  ```
  # specify kafka broker pod to be killed
  - name: KAFKA_BROKER
    value: 'kafka-cp-kafka-0'
  ```
