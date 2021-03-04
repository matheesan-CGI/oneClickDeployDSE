# One Click K3d(Jhub, Elastic) --> K3d

The purpose of this set of scripts is to eventually evolve it into a one-click solution to get a Data science environment going.

  - One click installation of DSA Environment
    - Install Jhub on K3d Eviornment
    - Eventually make this cloud agnostic
  - Currently has k3d set of features working with JupyterHub
  - Has a Azure set of features working with Elasticsearch, Kibana and JupyterHub
    - To get Jupyterhub on Azure, follow the steps in the JupyterHub setup section while ignoring the k3d cluster creation portion
  - Optional setup for Kubernetes Dashboard is included

# MacOS Jupyterhub K3d Installation

##### After installing K3d from k3d.io documentation, follow these steps to set up your cluster for JupyterHub:
Note: this cluster currently doesn't work with elasticsearch installation
```sh
#Create Cluster with chosen name
k3d cluster create <NAME> --agents 3
#Allow Kubeconfig to use k3d cluster
k3d kubeconfig merge <NAME> --switch-context
#Use the new cluster with kubectl
kubectl get nodes
```

#### Creating an Azure Kubernetes Cluster
```sh
az aks create -g matheesanmKubeEnv --name elasticCluster --node-count 7 --generate-ssh-keys
az aks get-credentials --resource-group matheesanmKubeEnv --name elasticCluster
```

#### Install HELM if not already installed:
```sh
#Install
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
#Verify helm is installed
helm list
```
#### Create config file for JupyterHub:
```sh
#Generate a random hex string representing 32 bytes AND Store this to use as a security token
openssl rand -hex 32
#Create and start editing a file called config.yaml
nano config.yaml
#The following is mandatory for jupyterHub yaml file
proxy:
  secretToken: "<RANDOM_HEX>"
```
The following portions of YAML code are Optional and can be added to config.yaml, but are not used in this implementation:
```sh
singleuser:
  memory:
    limit: 1G
    guarantee: 1G
  cpu:
    limit: 1
    guarantee: 1
  storage:
    type: none
```
## JupyterHub Installation:
```sh
#Install JupyterHub:
helm repo add jupyterhub https://jupyterhub.github.io/helm-chart/
#Update Helm
helm repo update
#Variables to use the following
RELEASE=jhub
NAMESPACE=jhub
#Now install the chart configured by your config.yaml
helm upgrade --cleanup-on-fail --install jhub jupyterhub/jupyterhub --namespace jhub --create-namespace --version=0.9.0 --values config.yaml
#See the pods being created
kubectl get pod --namespace jhub
#The External IP ofr proxy-public should be available, if the load balancer is configured correctly(This has yet to be tested)
kubectl get service --namespace jhub
#But in the case you don't get an external IP, you can access Jhub through the following command, and going to localhost:8080
kubectl port-forward -n jhub svc/proxy-public 8080:80
```

### Applying Changes to JHub
When you want to modify the config.yaml file, try following the following steps:
1. Make a change to your config.yaml.
2. Run a helm upgrade:
    ```
        RELEASE=jhub
        NAMESPACE=jhub
    
        helm upgrade --cleanup-on-fail \
                  $RELEASE jupyterhub/jupyterhub \
          --namespace $NAMESPACE
          --version=0.8.2 \
          --values config.yaml
    ```
    In this case, your release can be found through helm list
3. Verify that the hub and proxy pods entered the Running state after the upgrade completed.
```sh
NAMESPACE=jhub
kubectl get pod --namespace $NAMESPACE
```

# Azure ElasticSearch + Kibana Installation

#### If you haven't already, create an Azure cluster
```sh
az aks create -g <resourceGroupName> --name <kubernetesCluster> --node-count 7 --generate-ssh-keys
az aks get-credentials --resource-group <resourceGroupName> --name <kubernetesCluster>
```

#### Install the Elastic Operator
```sh
kubectl apply -f https://download.elastic.co/downloads/eck/1.4.0/all-in-one.yaml
#Log checking:
kubectl -n elastic-system logs -f statefulset.apps/elastic-operator
```

#### Deploy an elasticsearch cluster
```sh
cat <<EOF | kubectl apply -f -
apiVersion: elasticsearch.k8s.elastic.co/v1 
kind: Elasticsearch 
metadata: 
  name: quickstart 
spec: 
  version: 7.11.1 #Make sure you use the version of your choice 
  http: 
    service: 
      spec: 
        type: LoadBalancer #Adds a External IP 
  nodeSets: 
  - name: default 
    count: 1 
    config: 
      node.master: true 
      node.data: true 
      node.ingest: true 
      node.store.allow_mmap: false 
EOF
```

#### Monitor elasticsearch
You should eventually see the quickstart-es-http service with an IP for the External Load Balancer.
Take note of this IP as you will need it later on.
```sh

kubectl get elasticsearch

kubectl get pods -w

kubectl logs -f quickstart-es-default-0

kubectl get service quickstart-es-http
```

Get the password using the following command and try to curl into elasticsearch
Alternativley, you can visit https://<ExternalIP>:9200 and try to login
The Username is always elastic, and the password can be found using the following commands
```sh
PASSWORD=$(kubectl get secret quickstart-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 —decode)

curl -u "elastic:$PASSWORD" -k "https://52.147.212.172:9200”
```

## KIBANA Installation:

#### Deploy Kibana
```sh
cat <<EOF | kubectl apply -f - 
apiVersion: kibana.k8s.elastic.co/v1 
kind: Kibana 
metadata: 
  name: quickstart 
spec: 
  version: 7.11.1 #Make sure Kibana and Elasticsearch are on the same version. 
  http: 
    service: 
      spec: 
        type: LoadBalancer #Adds a External IP 
  count: 1 
  elasticsearchRef: 
    name: quickstart 
EOF
```

#### Follow the same pattern as for elasticsearch to login into Kibana
```sh
kubectl get kibana

kubectl get -n elasticSearch

curl -u "elastic:$PASSWORD" -k "https://52.147.212.172:9200”
```

In the background, run the following for now(Until proper Nodeport setup is added to the yaml):
The following is only needed if you're not using an External IP

```sh
kubectl proxy
```

```sh
kubectl port-forward service/quickstart-es-http 9200
```

```sh
kubectl port-forward service/quickstart-kb-http 5601
```

```sh
kubectl get secret quickstart-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 —decode
```

JupyterHub Installation(Assuming you have the config file and Helm setup already there)
```sh
helm upgrade --cleanup-on-fail --install jhub jupyterhub/jupyterhub --namespace jhub --create-namespace --version=0.9.0 --values config.yaml
```


# KUBE Dashboard Installation(Optional):
```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
```

### CREATE SERVICE ACCOUNT + Role binding: 
```sh
kubectl apply -f https://gist.githubusercontent.com/chukaofili/9e94d966e73566eba5abdca7ccb067e6/raw/0f17cd37d2932fb4c3a2e7f4434d08bc64432090/k8s-dashboard-admin-user.yaml
```

```sh
kubectl describe sa admin-user -n kube-system
```

### Get Secret to access DASHBOARD Then, use Token to sign into dashboard:
```sh
kubectl describe secret admin-user-token-ls25k -n kube-system
```

Then follow rest of JupyterHub Config as you would on k3d, but exclude the cluster creation portion to get jupyterHub on Azure.

### Python Code to verify that jupyterhub can ping to es:
```sh
pip install elasticsearch
pip install pandas


import requests
import urllib3
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

response = requests.get('https://<ExternalIPofES>:9200', verify=False, auth=('elastic', 'ZI8DOAE42j486958hqqt7izr'))

print (response.text)

try:
    import os
    import sys
    
    from elasticsearch import Elasticsearch as Elasticsearch
    
    import pandas as pd
    
    print('All modules loaded')
    
except Exception as e:
    print("some error {}".format(e))
    
es = Elasticsearch(['https://elastic:<base64-decoded_password>@<ExternalIPofES>:9200/'], verify_certs=False)



## importing socket module
import socket
## getting the hostname by socket.gethostname() method
hostname = socket.gethostname()
## getting the IP address using socket.gethostbyname() method
ip_address = socket.gethostbyname(hostname)
## printing the hostname and ip_address
print(f"Hostname: {hostname}")
print(f"IP Address: {ip_address}")

```

# END

### Current State: 
 --> See the latest 4 images for current state

### Extra Information/Resources:

Latest Todo(Connect Jupyterhub to es using python):
https://elasticsearch-py.readthedocs.io/en/7.10.0/

Look into HELK:
https://github.com/Cyb3rWard0g/HELK

Elastic Docs for Kube:
https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-deploy-elasticsearch.html

TODO Soon: Add logstash to act as middleware for ES and Kafka:
https://medium.com/@tharangarajapaksha/elk-stack-in-k8s-cluster-13bb509185e0
https://towardsdatascience.com/the-basics-of-deploying-logstash-pipelines-to-kubernetes-94a470ad34d9


Littlest JupyterHub Installatoin  --> Doesn't fully support Docker env

Hub runig w elastic kid,
Through yaml, controller, 
Be cloud agnostic
Cluster cost, build up and tear down is going to be annoying
We should learn how to do it in amazon and google
Have ability to build scripts on all 3
If we show them it in Azure
