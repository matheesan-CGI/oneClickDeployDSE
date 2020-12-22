# One Click K3d(Jhub, Elastic) --> K3d

Dillinger is a cloud-enabled, mobile-ready, offline-storage, AngularJS powered HTML5 Markdown editor.

  - One click installation of DSA Environment
    - Install Jhub on K3d Eviornment
    - Eventually make this cloud agnostic
  - See HTML in the right
  - Magic

# MacOS Jupyterhub K3d Installation

#### K3d:
```sh
#Create Cluster with chosen name
k3d cluster create <NAME> --agents 3
#Allow Kubeconfig to use k3d cluster
k3d kubeconfig merge <NAME> --switch-context
#Use the new cluster with kubectl
kubectl get nodes
```
#### HELM:
```sh
#Install
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
#Verify helm is installed
helm list
```
#### Config file:
```sh
#Generate a random hex string representing 32 bytes AND Store this to use as a security token
openssl rand -hex 32
#Create and start editing a file called config.yaml
nano config.yaml
#The following is mandatory for jupyterHub yaml file
proxy:
  secretToken: "<RANDOM_HEX>"
```
#The following are Optional and can be added to config.yaml:
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
### JupyterHub Installation:
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

### Current State: 
![alt text](https://github.com/matheesan-CGI/oneClickDeployDSE/edit/main/12_18_20_Current_State.png?raw=true)


#### Extra Information

Littlest JupyterHub Installatoin  --> Doesn't fully support Docker env

Hub runig w elastic kid,
Through yaml, controller, 
Be cloud agnostic
Cluster cost, build up and tear down is going to be annoying
We should learn how to do it in amazon and google
Have ability to build scripts on all 3
If we show them it in Azure
