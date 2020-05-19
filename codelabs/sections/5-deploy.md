## Deploy to your Kubernetes Cluster

It's time to deploy the frontend and backend to your cluster!

The preferred way to configure Kubernetes resources is to specify them in YAML files.

In the folder [yaml/](https://github.com/pingrid/nrk-kubernetes-intro/blob/master/yaml) you find the YAML files specifying what resources Kubernetes should create.
There are two files for specifying services and two files for specifying deployments. One for the backend application (*backend-service.yaml*) and one for the frontend application (*frontend-service.yaml*).
Same for the deployments.

1. Open the file [yaml/backend-deployment.yaml](https://github.com/pingrid/nrk-kubernetes-intro/blob/master/yaml/backend-deployment.yaml)
2. In the field `spec.template.spec.containers.image` insert the full name of your backend Docker image created in the previous step. 

The name should be on the form `[CONTAINER REGISTRY ID]/azurecr.io/[IMAGE NAME]:VERSION`. 
You can find the correct path of your image by going to [Azure Portal](https://portal.azure.com/) and searching for Container registry. Select your registry, then select *Repositories*. Latest version can be found under repository under Container registry. 
 
There are a few things to notice in the deployment file:
- The number of replicas is set to 3. This is the number of pods we want running at all times
- The container spec has defined port 5000, so the Deployment will open this port on the containers
- The label `app: backend` is defined three places:
  - `metadata` is used by the service, which we will look at later
  - `spec.selector.matchLabels` is how the Deployment knows which Pods to manage
  - `spec.template.metadata` is the label added to the Pods
  
3. Open the file [yaml/frontend-deployment.yaml](https://github.com/pingrid/nrk-kubernetes-intro/blob/master/yaml/frontend-deployment.yaml). 
4. Insert your Frontend Docker image name in the field `spec.template.spec.containers.image`.  

5. Now we need to give Kubernetes access to our container registry. 

Run the script located in yaml/create-service-principal.sh: 

```
sh  yaml/create-service-principal.sh
```

Store the variables printed from the script and generate a secret for accessing your Azure Container Registry: 

```
kubectl create secret docker-registry <secret-name> \
  --namespace <namespace> \
  --docker-server=https://<container-registry-name>.azurecr.io \
  --docker-username=<service-principal-ID> \
  --docker-password=<service-principal-password>
  ```
  
Let secret-name be `acr-docker-secret` and use the principal service-principal-id and service-principal-password be the ones you got by running the scripts above.

Verify that you now have a secret: 
```
kubectl get secret. 
``` 

A secret is only available for resources within the cluster and is a great way to store passwords and tokens. 




*It did not work?*  Alternative ways for accessing your build images [here](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-auth-kubernetes).

5. Create the resources for the backend and frontend (from root folder in the project):
  
  ```
  kubectl apply -f ./yaml/backend-deployment.yaml
  kubectl apply -f ./yaml/frontend-deployment.yaml
  ```

6. Watch the creation of pods:
  
  ```
  watch kubectl get pods
  ```

  If you don't have `watch` installed, you can use this command instead:
  
  ```
  kubectl get pods -w
  ```

  When all pods are running, quit by `ctrl + q` (or `ctrl + c` when on Windows).

## About pods 

Pods are Kubernetes resources that mostly just contains one or more containers,
along with some Kubernetes network stuff and specifications on how to run the container(s).
All of our pods contains only one container. There are several use cases where you might want to specify several
containers in one pod, for instance if you need a proxy in front of your application.

The Pods were created when you applied the specification of the type `Deployment`, which is a controller resource. 
The Deployment specification contains a desired state and the Deployment controller changes the state to achieve this.
When creating the Deployment, it will create ReplicaSet, which it owns.

The ReplicaSet will then create the desired number of pods, and recreate them if the Deployment specification changes,
e.g. if you want another number of pods running or if you update the Docker image to use.
It will do so in a rolling-update manner, which we will explore soon.

The Pods are running on the cluster nodes. 

![Illustration of deployments, replicasets, pods and nodes.](https://storage.googleapis.com/cdn.thenewstack.io/media/2017/11/07751442-deployment.png)



*Did you noticed that the pod names were prefixed with the deployment names and two hashes?* - The first hash is the hash of the ReplicaSet, the second is unique for the Pod.

4. Explore the Deployments:
  
  ```
  kubectl get deployments
  ```

Here you can see the age of the Deployment and how many Pods that are desired in the configuration specification,
the number of running pods, the number of pods that are updated and how many that are available.

5. Explore the ReplicaSets:
  
  ```
  kubectl get replicaset
  ```

The statuses are similar to those of the Deployments, except that the ReplicaSet have no concept of updates.
If you run an update to a Deployment, it will create a new ReplicaSet with the updated specification and
tell the old ReplicaSet to scale number of pods down to zero.

