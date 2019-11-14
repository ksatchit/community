# Setup kafka with KUDO

Notes: 
 - These steps were performed on a Konvoy cluster created on AWS

## Pre-requisites

- Setup a Kubernetes Cluster in version 1.13 or later
- Install kubectl version 1.13 or later

## Install KUDO CLI

### For Mac

```bash
brew tap kudobuilder/tap
brew install kudo-cli
```

### For Ubuntu

```bash
mkdir $GOPATH/src/github.com/kudobuilder
cd $GOPATH/src/github.com/kudobuilder
git clone https://github.com/kudobuilder/kudo.git
cd kudo/
make cli-install
```

## Install KUDO into your cluster

```bash
kubectl kudo init
```

## Installing the Operator

### Install Zookeeper

```bash
kubectl kudo install zookeeper --instance=zookeeper-instance
```

Note: If you are not passing any StorageClass it will use the default StorageClasss, Or you can pass the StorageClass using the command below:

```bash
kubectl kudo install zookeeper --instance=zookeeper-instance -p STORAGE_CLASS=<STORAGE_CLASS_NAME>
```

### Install Kafka

```bash
kubectl kudo install kafka --instance=kafka
```

Note: If you are not passing any StorageClass it will use the default StorageClasss, Or you can pass the StorageClass using the command below:

```bash
kubectl kudo install kafka --instance=kafka -p STORAGE_CLASS=<STORAGE_CLASS_NAME>
```

## Removing the Operator

```bash
kubectl delete instance kafka zookeeper-instance
kubectl delete crd operators.kudo.dev operatorversions.kudo.dev 
kubectl delete ns kudo-system
```

---
Note: Make sure you don't have any local folder with name kafka or zookeeper in the place where you are executing these commands
