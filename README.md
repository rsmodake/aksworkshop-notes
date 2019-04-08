# AKS Workshop Notes
Proctor notes for The Azure Kubernetes Workshop, https://aksworkshop.io, supplemental to the workshop documentation.

## 1.1 Prerequisites

### Azure subscription

#### Solution notes

1. The `az login...` command is not required if using Azure Shell. 
1. One can also log in with the Azure account Username/Password provided.

## 1.2 Kubernetes basics

Also point users to:

- https://kubernetes.io/docs/home/
- https://kubernetes.io/docs/tutorials/kubernetes-basics/

## 1.3 Application Overview

Nothing else to add.

## 1.4 Scoring

Optional, at the discretion of the Emcee. More practical when there is one or more proctors, in addition to the Emcee.

## 1.5 Tasks

# 2 Getting up and running

## 2.1 Deploy Kubernetes with Azure Kubernetes Service (AKS)

### Concepts

- Deployment methods, not just CLI and Portal (Azure Shell). Also, PowerShell, ARM, Terraform, Ansible
- K8S versions and AKS cadence update (120 days behind as of 2019-04-05!)
- kubectl, the K8s CLI
- Regions, verify that AKS is available in a given region and what versions are supported
  - `az aks get-versions`
- Service principal Add link to Resources

### Solution notes

> Deploy Kubernetes to Azure, using CLI or Azure portal using the latest Kubernetes version available in AKS

- Visit https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough for step by step
- Note that solution uses Azure CLI. Use Azure Shell or install the Azure CLI. PowerShell is an alternative.

|Step|Method|
|--|--|
|SSH Key|If using your own Azure subscription, you can use your own SSH key, and not use the `--generate-ssh-keys` option in the `az create aks` command. Instead, use the `--ssh-key-value /path` option.|
|Set env variables used in the lab|Resource Group, Location, Cluster name|
|Create RG group|`az group create --name $RGNAME --location $LOCATION`|
|Get V8s versions for my region|az aks get-versions --location $LOCATION -o table|
|Create AKS cluster|`az aks create --resource-group $RGNAME --name  $CLUSTERNAME --enable-addons monitoring --kubernetes-version 1.12.6 --generate-ssh-keys --location $LOCATION --service-principal $APPID --client-secret $CLIENTSECRET`|
|Get Cluster creds|`az aks get-credentials -g $RGNAME -n $CLUSTERNAME`|
|Verify cluster access|`kubectl get nodes`|

> Cluster creds are stored in `~/.kube/config`. Open the files with `code ~/.kube/config`. Look for current context.

### Tips

1. Create a `aksworkshop` directory in your home dir (`~/`). Store code files here, e.g. K8s manifests.
1. Add env vars to `.bashrc`.
1. Add `alias k=kubectl` to `.bashrc`.

## 2.2. Deploy MongoDB

### Concepts
- Helm
 -ConfigMap as in
https://github.com/helm/charts/blob/master/stable/mongodb/templates/configmap.yaml
 - Secrets as in 
https://github.com/helm/charts/blob/master/stable/mongodb/templates/secrets.yaml 

### In Hints

2.2. Deploy MongoDB

> _Be careful with the authentication settings when creating MongoDB. It is recommended that you create a standalone username/password and database._

1. Why _Be careful_?
2. How does creating a standalone username/password and database help?
Note, if a database does not exist, MongoDB creates the database when you first store data for that database.
3. Why refer to this as a “standalone”? Why not use the language from the values file, “MongoDB custom user and database”. standalone is used in the helm chart, but not specifically where a custom database is defined. (Search on mongodbDatabase in https://github.com/helm/charts/blob/master/stable/mongodb/templates/deployment-standalone.yaml.)
4. Why use **stable.mongo** and not **mongodb-replicaset** as the Helm chart?

### Tips

1. To Check if a cluster is RBAC enabled, visit https://resources.azure.com/ and drill down, e.g.  
https://resources.azure.com/subscriptions/[SUBSCRIPTIONID]/resourceGroups/[RGNAME]/providers/Microsoft.ContainerService/managedClusters/[CLUSTERNAME]

    In my case, I see:

    ```
    "nodeResourceGroup": "MC_rgmichael28849_aksmichael28849_eastus",
        "enableRBAC": **true**,
        "networkProfile": {
        "networkPlugin": "kubenet",
    ```

1. to verify `helm init`, run `helm version`, e.g.
    ```
    michael@Azure:~$ helm version
    Client: &version.Version{SemVer:"v2.13.1", GitCommit:"618447cbf203d147601b4b9bd7f8c37a5d39fbb4", GitTreeState:"clean"}
    Server: &version.Version{SemVer:"v2.13.1", GitCommit:"618447cbf203d147601b4b9bd7f8c37a5d39fbb4", GitTreeState:"clean"}
    ```

1. If helm is already installed, and you get error messages like:
   
    ```
    Error: release orders-mongo failed: namespaces "default" is forbidden: User "system:serviceaccount:kube-system:default" cannot get resource "namespaces" in API group "" in the namespace "default"
    ```
    Then run: `helm init --service-account tiller --upgrade`

1. If you get error:
    ```
    Error: could not find a ready tiller pod
    ```
    Then, the tiller pod is not ready. Give it some more time. Check with `helm version`.

1. Verify Mongo by: 
- Get a shell to the running Container, e.g. 
```
kubectl exec -it orders-mongo-mongodb-6684cbf59f-5rwq2 -- /bin/bash
```
Once in the shell, run `service mongod status`

### Notes

1. In https://github.com/helm/charts/blob/master/stable/mongodb/values.yaml, note this code that is commented out:
    ```
    ## MongoDB custom user and database
    ## ref: https://github.com/bitnami/bitnami-docker-mongodb/blob/master/README.md#creating-a-user-and-database-on-first-run
    ##
    # mongodbUsername: username
    # mongodbPassword: password
    # mongodbDatabase: database
    ```

    These values are consumed in https://github.com/helm/charts/blob/master/stable/mongodb/templates/deployment-standalone.yaml. 

    ... Which in turn are passed on docker run during helm chart installation. (See https://github.com/bitnami/bitnami-docker-mongodb/blob/master/README.md#creating-a-user-and-database-on-first-run) 

    This is why they are passed on the command line.

    Alternatively, can a secret be used? How would that be done?

## 2.3 Deploy the Order Capture API

### Concepts

- Health checks in K8S - Liveness and readiness probes:
https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/

- Services: Types, labels, selector (see Resource link)
    
### Tasks

- Provision the captureorder deployment and expose a public endpoint
    - Straight forward
- Ensure orders are successfully written to MongoDB
    - The curl solution is not obvious. Let’s break it down 


    ```
    curl -d '{"EmailAddress": "mike@snp.com", "Product": "prod-1", "Total": 100}' -H "Content-Type: application/json" -X POST http://40.121.XXX.XXX/v1/order
    ```

    -d: (HTTP) Sends the specified data in a POST request to the HTTP server.
    
    -H: (HTTP) Extra header to include in the request when sending HTTP to a server.

    -X: (HTTP) Specifies a custom request method to use when communicating with the HTTP server. Default is GET. We are using POST

    POST http://40.121.XXX.XXX/v1/order

- Access the Swagger UI at http://[host]/swagger.
For me: http://40.121.XXX.XXX/swagger/

- With regard to the hint:

    > The Order Capture API exposes the following endpoint for health-checks: http://[PublicEndpoint]:[port]/healthz”...

    …note the /healthz path. This is coded in the application. Refer to:
https://github.com/Azure/azch-captureorder/blob/1a4b34528882d9fb85a3d88246ddc53b74801f79/tests/default_test.go

### Tips

1. `k get svc`

## 2.4 Deploy the frontend using Ingress

### Concepts

- Ingress
- HTTP application routing
https://docs.microsoft.com/en-us/azure/aks/http-application-routing 

### Tasks

- Deployment of app is straight forward
- Use of HTTP Application Routing add-on for Ingress
  - Application of K8s resource is straight forward
  - Why this approach if not recommended for production use? 

> Retrieve your cluster specific DNS zone name by running the command below”

```
michael@Azure:~/aksworkshop$ az aks show -g $RGNAME  -n $CLUSTERNAME --query addonProfiles.httpApplicationRouting.config.HTTPApplicationRoutingZoneName -o table
Result
-------------------------------------
0c2e4fa855404271be9c.eastus.aksapp.io
```
Open the front end app via:
<http://frontend.0c2e4fa855404271be9c.eastus.aksapp.io/>

## 2.5 Monitoring

### Concepts

- Azure Monitor

### Tasks

Straight forward

### Tips

Coming soon

### Resources

- Another lab: <https://github.com/Azure/kubernetes-hackfest/blob/master/labs/monitoring-logging/README.md>
- <https://kubernetes.io/docs/tasks/debug-application-cluster/resource-usage-monitoring/>

## 2.5 Scaling

### Concepts

- Azure Container Instances

### Tasks

Straight forward. My az command: 

```
az container create -g $RGNAME -n loadtest --image azch/loadtest --restart-policy Never -e SERVICE_IP=40.121.XXX.XXX
```
```
az container logs -g $RGNAME -n loadtest
```
```
az container delete -g $RGNAME -n loadtest
```


### Tips

Coming soon

### Resources


## Misc Notes

1. On trying to review live container logs for frontend container:
    ```
    Kube API Response
    pods "frontend-86bbcd8448-m7jfw" is forbidden: User "clusterUser" cannot get resource "pods/log" in API group "" in the namespace "default"
    ```
    I am able to see logs in logs analytics.
    No issues apparent from `k logs  frontend-86bbcd8448-m7jfw`.
    
2. I experienced a timeout error on Monday, April 8:
    ```
    michael@Azure:~$ k logs captureorder-69fd98b57b-bcnw2
    Error from server: Get https://aks-nodepool1-30479232-1:10250/containerLogs/default/captureorder-69fd98b57b-bcnw2/captureorder: dial tcp 10.240.0.4:10250: i/o timeout
    ```
    May be related to hot fix pushed on 4/4.
    
    Get list of upgrades:
    ```
    az aks get-upgrades -g $RGNAME -n $CLUSTERNAME  --output table
    Name     ResourceGroup    MasterVersion    NodePoolVersion    Upgrades
    default  rgmike20190405   1.12.6           1.12.6             1.12.7
    ```
    ```
    michael@Azure:~$ az aks upgrade -g $RGNAME -n $CLUSTERNAME --kubernetes-version 1.12.7
    Kubernetes may be unavailable during cluster upgrades.
    Are you sure you want to perform this operation? (y/n): y
    - Running ..
    ```



