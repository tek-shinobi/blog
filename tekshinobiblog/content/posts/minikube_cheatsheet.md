---
title: "Kubernetes tutorial + Minikube:cheatsheet"
date: 2021-09-15T09:59:30+03:00
draft: false 
categories: ["kubernetes"]
tags: ["minikube"]
---

To create a virtual machine:
```shell
minikube start --driver=virtualbox  --no-vtx-check
```

you can open Virtualbox and see that a new virtual machine is now up and running

check if cluster is ok and everything installed correctly:
```shell
minikube status
```
the output is like this:
```shell
❯ minikube status
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

from here: https://minikube.sigs.k8s.io/docs/start/
now check that __kubectl__ commands are working:

check nodes:
```shell
kubectl get nodes
```
output:
```shell
❯ kubectl get nodes
NAME       STATUS   ROLES           AGE     VERSION
minikube   Ready    control-plane   5m44s   v1.26.3
```
so we see that there is one node and its in ready state, and its running 1.26.3 version of kubernetes


so let's create some deployments using `create deployment`:
```shell
kubectl create deployment hello-minikube --image=kicbase/echo-server:1.0
```

you can list the deploymets like so:
```shell
kubectl get deployments
```
output:
```shell
❯ kubectl get deployments
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
hello-minikube   1/1     1            1           28s
❯ 
```

now expose this deployment as a service:
```shell
kubectl expose deployment hello-minikube --type=NodePort --port=8080
```

check the url of the service:
```shell
minikube service hello-minikube --url
```
it should return something like this:
```shell
❯ minikube service hello-minikube --url
http://192.168.59.100:30673
```

the service will be running at that url and can be visited on url

so setup works

lets cleanup the stuff:
```shell
kubectl delete services hello-minikube
kubectl delete deployment hello-minikube
```

With the above demo, we know that kubernetes cluster is working properly

to install an application in a new pod on a kubernetes cluster:
```shell
kubectl run nginx --image=nginx
```

here application will create a new pod named __nginx__  is installed in it and the source of its download is specfied by `--image` , in this case, the image named  __nginx__ on dockerhub will be downloaded and installed in a new pod. You can also add tags to the docker image name.

You can configure kubernetes to pull the image from the public dockerhub or a private repo

to list all running pods:
```shell
kubectl get pods
```
OR use the `wide` option to get some more details like node where pod is running and theinternal IP address of thepod within the kubernetes cluster
```shell
kubectl get pods -o wide
```

the output under `Ready` will be something like `1/2`.. here it means `total number of containers in ready state in the pod / total number of containers in the pod`.. so `1/2` here would mean that the pod contains a total of 2 containers out of which 1 is in ready state

to get more info on a pod named `nginx`:
```shell
kubectl describe pod nginx
```

To create a deployment using imperative command, use **kubectl create:**

```bash
$ kubectl create deployment nginx --image=nginx 
```


In practise however, we will create pods uing yaml definition file.

## Kubernetes definition file
kubernetes definition file always has 4 top level fields (also called root level properties):
- apiVersion
- kind
- metadata
- spec

there is a relationship between apiVersion value and kind value
| Kind       | Version
| :---------------- | :------: 
| POD        |   v1
| Service           |   v1   
| ReplicaSet    |  apps/v1   
| Deployment |  apps/v1   

example metadata. Imaging that in a pod definition file (`pod-definition.yaml`) like so:
```yaml
apiVersion: v1
kind: Pod
metadata:
    name: myapp-pod
    labels:
        app: myapp
        type: front-end
spec:
    containers:
        -   name: nginx-container
            image: nginx
```
`labels` field above helps in filtering the pods. For example, if we have thousands of pods running with labels.type as `front-end`, `back-end` etc, we will be able to filter all front-end pods using this label. `name` refers to the name of the object created using this yaml file, in this case a pod.
Important: under metadata, you can only add names, lables or any other property that kubernetes expects as metadata. However, under labels, you can have any key-value pair as you see fit. 

`spec` specifies the container or image in the pod.
`spec` contains a property called `containers` which is an array as a pod can have multiple containers inside it, though most of the time, it only has one.
`image` key has the value `nginx` which is the name of the docker image in the dockerhub repository. If you are using an image repository other than dockerhub, then puth the full path to the image here instead of just `nginx`

now, to create the pod from this definition file, 
```shell
kubectl create -f pod-definition.yaml
```
__Note__: `kubectl create` and `kubectl apply` are the same. You can use either `create` or `apply`

once you have created pod, to see it:
```shell
kubectl get pods
```

to see all the pods running

to see pod details:
```shell
kubectl describe pod myapp-pod
```

__Tip__: To make changes to a running kubernetes pod/instance (for example, the instance is in waiting/error state because a wrong docker image was specified in its manifest.. you can either delete the pod and then update tha manifest and then re-launch the pod... OR.. you can use `kubectl edit` to access the running container's spec and edit them, this will only make changes to this container) `kubectl edit pod <pod_name>` like `kubectl edit pod myapp-pod` ... this will not open the file `pod-definition.yaml` but the file that kubernetes generated in the background (actually a temporary file generated by kubernetes and living in-memory.. and generated to give you access to the kubernetes object. Changes made to this file are directly applied to the instance of the object running in the cluster, as soon as the file is saved.. so you must be very careful with the changes that you are making here) and this will give you access to the pod definition file used internally by the kubernetes. So, now make changes directly to opened file (like fixing the image name shown under `containers` as that was the error we were looking to fix)

__Summary__:
```shell
# show all pods
kubectl get pods
# show pods with some additional details like which node is it running on
kubectl get pods -o wide
# show details for a specific pod by name
kubectl describe pod myapp-pod
# to create a pod from a pod-definition file named pod-definition.yaml
kubectl create -f pod-definition.yaml
# to delete a pod by name
kubectl delete pod myapp-pod
# to edit a running instance of pod
kubectl edit pod myapp-pod
```

## Kubernets Controllers

Controllers are the brains behind kubernetes. They are the processes that monitor kubernetes objects and respond accordingly.

#### Replication Controller
There are different kinds of controllers. Replication controller is one of these.
Replication controller's job is to:
- ensure that the specified number of pods are running all the time. So, even if we are only running a single pod, and that pod dies, replication controller detects this and ensures to create a new pod to replace the one that just died.
- **Load Balancing and Scaling**: share the load across multiple pods. The replication controller can scale the pods across different nodes within the cluster to distribute load.

__Important__: There are two similar terms: `Replication Controller` and `Replica Set`. Both have the same purpose but they are not the same. `Replication Controller` is the older technology that is being replaced by `Replica Set`. `Replica Set` is the new recommended way to setup replication.

#### How to create a replication controller
As for a pod, we start with a replication controller defition file: `rc-definition.yaml`:
```yaml
apiVersion: v1
kind: ReplicationController
metadata:
    name: myapp-rc
    labels:
        app: myapp
        type: front-end
spec:
    -   template:
            metadata:
                name: myapp-pod
                labels:
                    app: myapp
                    type: front-end
            spec:
                containers:
                    -   name: nginx-container
                        image: nginx
    replicas: 3

``` 
Since replication controller is supported in kubernetes API version `v1`, we specify that in `apiVersion`

Also, in this replication controller, we are managing replicas of pod we created in `pod-definition.yaml`, instead of manually managing the pod creating via `pod-definition.yaml`, we just copy the `metadata` and `spec` sections of it and put it under `template` in Replication Controller definition file. We will also specify the number of replicas we want

we can now create the pods like so:
```shell
kubectl create -f rc-definition.yaml
```

to see the replication controller created;
```shell
kubectl get replicationcontroller
```

in the output, `DESIRED` meand desired number of replicas. Current means actual number of replicas at the moment. `READY` means how many of those desired replicas are running

to see the pods created by the replication controller:
```shell
kubectl get pods
```

#### How to create a Replicaset Controller
It is very similar to the replication controller.
`replicaset-definition.yaml`:
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
    name: myapp-replicaset
    labels:
        app: myapp
        type: front-end
spec:
    -   template:
            metadata:
                name: myapp-pod
                labels:
                    app: myapp
                    type: front-end
            spec:
                containers:
                    -   name: nginx-container
                        image: nginx
    replicas: 3
    selector:
        matchLabels:
            type: front-end

``` 
Since replicaset controller is supported in kubernetes API version `apps/v1`, we specify that in `apiVersion`

`selector` is specific to replica-set (is not present in replication controller) and is used to identify which pods fall under the replicaset-controller. Although `selector` is also present in replication-controller, it is an optional field. That's why we had skipped it in `rc-defition.yaml`. In case of replicaSets, it is a required field and also has some addition functionality not supported in replication-controller.

___But why would you need to provode the `selector` to identify what pods fall under it when you have provided the same info in the `template` section?___
It's because replica-set controller can also manage pods that were not part of replica-set creation

For example, if there are pods present before the replica-set creation that have the same labels as in the template section, the replica-set will take those in consideration when creating the replicas.

`selector`: always has the same structure as shown above.

to create replicaSet,
```shell
kubectl create -f replicaset-definition.yaml
```
to get all replicasets running:
```shell
kubectl get replicaset
```

to delete a replicaset,
```shell
kubectl delete replicaset myapp-replicaset
```
(the above delete command will also delete all the underlying pods)

###### Labels and Selectors
___So why do we label our pods and objects in kubernetes?___
Let's see a simple scenario. Let's say that we have 3 instances of our frontend application as 3 pods. We would like to setup a replicaset-controller to ensure that we always have 3 replicas at any time. Yes, this is one of the use-cases of replicasets. You can use them to monitor already created pods as in this example. The role of the replicaset is to monitor the pods and if any one of them were to fail, to deploy a new one. The replicaset is in fact a process that monitors the pods. But how does a replicaset know which pods to monitor?  There could be 100s of other pods in the cluster running different applications. This is where labeling our pods during creation comes in handy. We can now provide label as a filter for the replicaset. Under the `matchLabels` within `selector`, we provide the same labels we used when creating the pods. This way the replica set knows which pods to monitor.

##### How to scale the replicasets

Using the previous `replicaset-definition.yaml` file, we have 3 replicas. If in the future, we wanted to scale up to 6 replicas, how do we update our defintion files? There are multiple ways of doing it. 
1. First one is to update the number of needed replicas in the `replicaset-definition.yaml` itself. Then use `kubectl replace -f replicaset-definition.yaml` to replace the replicaset definition file. This will update the replicaset to have 6 replicas.
2. `kubectl scale --replicaset=6 replicaset-definition.yaml` (either like this, by providing the name of the replicaset definition file) OR `kubectl scale --replicas=6 replicaset myapp-replicaset` (or like this.. by providing the name of the replicaset in the type name format... here type being `replicaset` and name being `myapp-replicaset`) . However, using the file-name as input will not result in updating the number of replicas in the file.
3. There are also options available to automatically scale the replicas based on load.

__tip__: If you changed the pod template inside a replicaset, the changes are not applied to existing pods. If you want changes to be applied, you either need to delete the existing pods manually, or better delete the existing instance of replicaset and create a new one.

___summary___:
```shell
# create
kubectl create -f replicaset-definition.yaml
# list all created/existing replicasets
kubectl get replicaset
# list all created/existing replicasets.... rs is shorthand for replicaset
kubectl get rs
# describe replicaset
kubectl describe replicaset myapp-replicaset
# delete a replicaset by name
kubectl delete replicaset myapp-replicaset
# replace a replicaset 
kubectl replace -f replicaset-definition.yaml
# scale a replicaset
kubectl scale --replicas=6 replicaset myapp-replicaset
# to edit a running instance of replicaset
kubectl edit replicaset myapp-replicaset
```

##### Export
At any time, if you want to export the definition file of any kubernetes object, lets say replicaset in this case, do it like so:
(very handy in scenarios where you don't have a definition file for the running kubernetes objects)
```shell
kubectl get rs myapp-replicaset -o yaml > new-replicaset.yaml
```
this will export the definition file in yaml format for the replicaset object named `myapp-replicaset`

#### Deployment Controller
We generally don't use Replicasets directly. We manage them via Deployments.
A deployment controller manages things like rolling updates (apply new changes to docker image into running instances in progressive manner.. like if 100 pods are running, update one pod at a time instead of updating all pods at same time.. this is to ensure least downtime), rollback of updates, grouping a set of changes and apply them as one update, scaling (think of everything that replicaset offers... since they manage replicasets internally)

In the kubernetes hierarchy, deployment controller comes higher up in the hierarchy, sitting above replicaset controller. Giving ability to apply seamless updates to underlying replicaset instance (which internally manages a whole set of pods), scaling, rolling updates to pods within replicaset instance etc. In fact, you can copy the `spec` section of the replicaset-definition.yaml file and set it as `spec` for `deployment-definition.yaml`

To create a deployment, we create a deployment definiton file:
`deployment-definition.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  labels:
    app: myapp
    type: front-end
spec:
  replicas: 3
  selector:
    matchLabels:
        type: front-end
  template:
    metadata:
      name: myapp-pod
      labels:
        app: myapp
        type: front-end
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.14.2
```

As you see above, the template in a deployment-controller definition has a pod definition inside it.

```shell
# create deployment from the definition file
kubectl create -f deployment-definition.yaml
# to see the newly create deployment
kubectl get deployments
# to get more info of the deployment object
kubectl describe deployment myapp-deployment
# deployment automatically creates a replicaset
# the created rs by default has the same name as the deployment
kubectl get rs
# since rs automatically creates pods
kubectl get pods 
# to see all objects created in the cluster at once
kubectl get all
```

the output of `kubectl get deployments`  contains `READY` which means `actual-pods/desired-pods` and `AVAILABLE` which means how many of the actual pods are available.

##### Rollout and Versioning

When you first create a deployment, it triggers a rollout. A new rollout creates a new deployment revision. Let's call it revision 1. In the future, when the application is updated, meaning when the container version is updated to a new one, a new rollout is triggered and a new deployment revision is created, named revision 2. This helps us keep track of changes to our deployments and to revert back to a previous version.

```shell
# to track the status of rollout
kubectl rollout status deployment/myapp-deployment
# to track deployment history
kubectl rollout history deployment/myapp-deployment
```

there is a `CHANGE-CAUSE` listed under the output of history. This records what command triggered the change. To make sure its recorded, use the create command like so:
```shell
kubectl create -f deployment-definition.yaml --record
```
In fact you can use `--record` with any command that causes a change like `edit` command  or `set image` command and this will make sure that the command that triggered the change is recorded in the revision history.

##### Deployment strategy
There are two types of deployment strategies.

###### Recreate Strategy
Imagine that there are 5 replicas of our web application instance deployed. One way to upgrade all of these is to destroy all of these instances and then create new instances of the web application. Meaning, first destroy 5 running instances and then deploy 5 new instances of the new application version. The problem with this approach is that between the time the instance are destroyed and the time when new deployments are available, the application is down and inaccessible to the users. This strategy is called the `Recreate Strategy`. This is not the default deployment strategy.
###### Rolling Update Strategy
The second strategy is where we do not destroy all running instances at once. Instead we take down the older version and create the newer version one by one. This way, the application never goes down and the upgrade is seamless. This is the default deployment strategy.

__tip__: 
```shell
kubectl describe deployment myapp-deployment
```
 will display the deployment strategy used under `StrategyType`

###### Doing the update
Application update can mean multiple things. Changing the docker image to a new one in the template, changing the number of replicas, changing the labels in the deployment-definition.yaml file etc.

Once we have made the changes, the folowing command applies the changes:
```shell
kubectl apply -f deployment-definition.yaml
```
This command triggers a new rollout and a new revision is created.

If you only wanted to change the image, you could use this command:
```shell
kubectl set image deployment/myapp-deployment nginx=nginx:1.9.1
```
but be careful with this command, as like `edit` command, this only changes the in-memory deployment file and not the actual `deployment-definition.yaml`

###### Upgrades
Let's take a look at upgrades in more detail. Here we will look at what happens when deployments trigger an upgrade.

When a new deployment is created to, say create 5 replicas, it first create a replica-set automatically, which in turn creates the number of pods to meet the number of replicas.

When you now trigger an upgrade, the kubernetes creates a new replicaset under the hood and starts deploying the containers there, at the same time, taking down the pods in the old replicaset, following a rolling upgrade strategy. This can be see with `kubectl get rs` command which will also display old replicaset (with 0 pods) and new replicaset with 5 pods. Now, say the at some point of time we realize that something isn't right with the new version of the application and we want to roll back the upgrade. Kubernetes deployments allow you to rollback to a previous revision.

```shell
# to undo a change
kubectl rollout undo deployment/myapp-deployment  
```

with the abve `undo` command, the deployment will now destroy the pods in the new replicaset and bring the older ones up in the old replicaset. And your application will be back to its older format.

Once again, you will be able to notice this difference when doing the `kubectl get rs` command.

finally:
```shell
kubectl run nginx --image=nginx
```
actually creates a deployment with the mentioned docker image. This is another way of creating a deployment by only using the image name and not a definition file. The required replicasets and the pods are automatically created in the backend. Using a definition file is recommended though as you can keep it in source control.

## Kubernetes Networking 

Let's start with a single node cluster. The node has an IP address, say 192.168.1.2. This will be the IP address used to acccess the kubernetes node, SSH into it etc. On a side note, if for example you are using a minikube single cluster node, then we are talking about the minikube IP address inside the hypervisor (the hypervisor will also be assigned an IP address, say 192.168.1.10). So its important to understand how your VMs are setup.

So, on a single node kubernetes cluster, we have created a pod. As you know, a pod contains a container. Unlike in a docker world, where an IP address is always assigned to a docker container, in the kubernetes world, the IP address is assigned to a pod. Each pod gets its own internal IP address. In this case, its in the range 10.244.*.* series and let's say that the IP assigned to the pod is 10.244.0.2. So, how is it getting this IP address? With kubernetes is initially configured, we create an internal private network with the address 10.244.0.0 and all the pods are attached to it. When you deploy a different pod, they get a different IP assigned from this network. The pods can communicate with each other through this IP. But accessing other pods using this internal IP address is not a good idea as its subject to change when pods are recreated. We will see better ways to establish communication between pods in a while. For now, its important to understand how the internal networking works in kubernetes.

So, its all easy and simply to know when its networking within a single node, but how does it work when you have multiple nodes in your cluster. Let's say that in our case, we have two nodes in the cluster with IP addresses 192.168.1.2 and 192.168.1.3 assigned to them. Note that they are not part of the cluster yet. Each of them have a single pod deployed. These pods are attached to their respective internal networks and they have their IP addresses assigned (say, each one is assigned 10.244.0.2). So, as you see, they are assigned the same IP address on their respective nternal networks. The two internal networks also have the same address of 10.244.0.0. But this is not going to work when the nodes become part of the same cluster. The pods have the same IP address assigned to them and this will lead to IP conflicts in the network. Now that's a problem. Also, when kubernetes creates a cluster, it doen not provide a network to handle these kinds of issues. In fact, kubernetes expects us to provide a network to meet fundamental networking requirements within a cluster. Some of these requirements are:
- All containers /Pods can communicate to one another without having to configure NAT
- All nodes can communicate with all containers and vice-versa without NAT

Kubernetes expects us to provides a networking solution that meets this criteria.

Fortunately, there are multiple pre-built solutions available. Like Cisco's ACI network, cilium, bitcloud fabric, flannel, VMWare NSXt and Calico. Depending on the platform you are deploying the kubernetes cluster on, you may use one of these solutions. So, in my cluster above, once I have provided a networkign solution, it makes sure that each network within the cluster is assigned a unique IP address. THis creates a virtual network of all pods and nodes where they are all assigned a unique IP address.

## Kubernets Services
Kubernets services enable communication between various components within and outside the application. Kubernetes Services helps us connect our application with other applications or other users. For example, our application has groups of pods for different sections. Such as one group of pods for serving frontend to users, another group of pods for running the backend servers, and a third group connecting to an external datasource. It is services that enable connectivity between these groups of pods. Servics enable frontend application to be available to end users, it helps communication between backend and frontend pods and helps in connectivity to an external datasource. Thus services enable loose coupling between microservices in our application.

In the previous section we briefly discussed how pods communicate with eachother through internal networking. Let's discuss some other aspects of networking. Let's start with external communication. So, we deployed our pod running a web application. How do we, as an external user, access the webpage? 

First of all, let us look at the existing setup. The kubernetes node has an IP address, let's say 192.168.1.2. My laptop is on the same network as well, so it has an IP address of 192.168.1.10. The internal pod network is in the range 10.244.0.0 and the pod has an IP of 10.244.0.2. Clearly, I cannot ping the pod 10.244.10.2 as it is a separate network. So what are the options to see the webpage.

We could SSH into the kubernetes node at 192.168.1.2. And then from the node, we could access the webpage by doing a `curl http://10.244.0.2`. But this is fron inside the kubernetes node and this is not what I really want. I want to be able to access the web-page simply by `curl http://192.168.1.2` (just by the kubernetes IP address... without the need of SSH).

So we need something connecting node to our laptop on one side and node and the pod (running the webserver) on the other side. This is where kubernetes service comes into play. Kubernetes service is just an object, just like pods, replicaset, and deployments seen before. One of its use cases is to listen to a port on the node and forward that request to a port on the pod running the web-application. This type of service is called a __NodePort__ service because the service listens to a port on the node and forwards the request to the pod.

There are other kinds of services available which we will now discuss.

#### Service Types
- __NodePort__: This service make an internal pod accesible on a port on the node.
- __ClusterIP__: This service creates a virtual IP inside the cluster to enable communication between different services such as a set of frontend servers to a set of backend servers.
- __LoadBalancer__: This service provisions a loadbalancer for our application in supported cloud providers.

#### NodePort Kubernetes Service
This service helps making a pod accessible by mapping a port on the node to a port on the pod. 

{{< figure src="/kube-service.png" alt="Kubernetes NodePort Service" caption="Kubernetes NodePort Service" >}}

In the above figure, we can see that in this service there are 3 ports involved. 

1. the port on the pod where the actual webserver is running is `80` and it is referred to as `TargetPort` because that is where the service forwards the request to
2. the port on the service itself. It is simply referred to as the `Port` (remember, these terms are from the point of view of the service). The service is in fact like a virtual server inside the node. Inside the cluster, this service has its own IP address and this IP address is called the __ClusterIP__ of the service. In the figure, this is `10.106.1.12`
3. the port on the node itself which we use to access externally and that is known as the `NodePort`. In the figure below, it is `30008`. The port value anything in range 30000 - 32767  

`service-definition.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
    name: myapp-service
spec:
    type: NodePort
    ports:
    -   targetPort: 80
        port: 80
        nodePort: 30008
    selector:
        app: myapp
        type: front-end
```
In the definiton file, the `type` refers to the type of service we are creating. It could be `ClusterIP`, `LoadBalancer` or `NodePort`  

For each array element in `ports`, the only mandatory field is `port`. If you don't provide a `targetPort`, it is assumed to be same as `port`. And, if no nodePort is specified, any available port in the region 30000 - 32767 is assigned as nodePort.

Also, since `ports` is an array, we can have multiple such port mappings within a service.

As you can see here, there could be 100s of pods in a cluster with webservices running on port 80. So, how do we connect the exact pod with this service? Well, what connects this service to the pod is the part under `selector` which is exactly what it is in `label` in pod-definition.yaml (literally copy pasted from `pod-definition.yaml`, the section under `labels`). This is a technique used often in kubernetes definition files, createing assiciation between objects using `labels` from one file and putting them as `selectors` in another.

```shell
# to create the service:
kubectl create -f service-definition.yaml
# to see the services created
kubectl get services
# to see the services created
kubectl get svc
```

Note that when you create a NodePort service, a ClusterIP service is also created. In fact, the `get svc` command will show both.

with the above in place, you will be able to access the webpage using:
```shell
curl http://192.168.1.2:30008
```
where the `192.168.1.2` is the IP address of the worker node.

So, to access the web-page, you need to know the IP address of the worker node. Since we are running this on minikube, we could do

```shell
minikube service myapp-service --url
```
and this will print us the url where the service is available, something like:
```shell
> minikube service myapp-service --url
http://192.168.1.2:30008
```

So, till now we have seen the case of a service mapped to a single pod. But this is not always the case. In real world, we have many instances of the same service in different pods for loadbalancing and fault-tolerance. In this case, we have multiple similar pods running our web application. They all have the same label in pod definition:
```yaml
labels:
    app: myapp
```

and for the matching selector in the service definition file:
```yaml
selector:
    app: myapp
```

So, imagine that we have 3 pods running the same webapplication for load balancing purposes. Each of them has the same label since thay were all created from the same pod-definition. So when as service tries to match with a pod using selector, it finds 3 pods with the same label as its selector. The service then automatically selects all the 3 pods as endpoints to forward the external request coming from the user. You don't have to do any additional configuration to make this happen. It used "Random" algorithm to bakance the load across the 3 different pods. Thus the service acts as a built in load balancer to distribute load across different pods.

And finally, lets see what happens when pods are distrubuted across multiple nodes. In this case, we have the web application on pods on separate node in the cluster. When we create a service, without us having to do any additional configuration, kubernetes automatically creates a service that spans across all the nodes in the cluster and maps the targetPort on the same nodePort on all the nodes in the cluster. This way, you can access the web-application using any node in the cluster and using the same portNumber.

#### ClusterIP Kubernetes Service
{{< figure src="/clusterIP.png" alt="ClusterIP service" caption="ClusterIP service" >}}

A full stack app has different kinds of pods hosting different parts of the application.

You may have a number of pods running a frontend webserver, another set of pods running a backend server and yet another set of pods running a key-value store like redis and another set of pods running a persistent database like mysql.

The web-frontend servers need to communicate with the backend servers and the backend servers need to communicate with the redis and the mysql servies etc. So, what is the right way to establish connectivity between these services or tiers of my application? The pods all have an IP address assigned to them. But thse IPs as we know are not static. These pods can go down anytime and new pods are created all the time. And so you cannot rely on these IP addresses for internal communication between different tiers of the application. 

Also, as in the image above, if front-end server at 10.244.0.3 needs to communicate with the backend server, which of the three pods it goes to and who makes that decision? 

A kubernetes service can help us group the pods together and provide a single interface to access the pods in a group. For example, a service created for the backend pod will help group all the backend pods together and provide a single interface for other pods to access this service. The requests are forwarded to pods under the service randomly. Similarly create additional service for redis and allow the backend pods to access the redis systems through this service. This enables us to easily and effectively deploy a microservices based application on kubernetes cluster. Each layer can scale or move as required without impacting communication between the various services. Each service gets an IP and a name assigned to it inside the cluster and that is the name that should be used by the other pods to access the service. This type of service is called __ClusterIP__ service.
`service-definition.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
    name: back-end
spec:
    type: ClusterIP
    ports:
        -   targetPort: 80
            port: 80
    selector: 
        app: myapp
        type: back-end
```

In fact, ClusterIP is the default type for a service. So, if we had skipped the `type: ClusterIP`, kubernetes would have anyways assumed the definition file to be for ClusterIP service.

## LoadBalancer Kubernetes Service

So we have seen NodePort service make an external facing application available on the port on worker nodes.

LoadBalancer type Service is only supported on specific kubernetes provoders like AWS, GCP and Azure. Instead of using an external LoadBalancer like nginx, HAProxy etc, you could use a supported LoadBalancer service.

## Running microservices on Kubernetes

Let's take an example of the demo application provided by docker https://github.com/dockersamples/example-voting-app
{{< figure src="/microservices.png" alt="Microservices" caption="Microservices" >}}

Here, there is a votiing app implemented across 5 docker containers. 
- `vote` is a python application that enables a user to vote
- `redis` serves as in-memory database for `vote` application
- `worker` is a .NET application that takes in data from `redis` and sends it to `db`
- `db` is a postgres database 
- `result` is a NodeJS application that gets its data from `db` and then displays the reults of the vote in realtime

Without using docker-compose or swarm, we can manually run each application in a docker container and then link them together (`--link`) so that we are able to access the applications via hostname

here `--link` has the format `container_name:hostname`

Note the hostname used in the `try` block in the .NET application code example. We are using `redis` and `db` as hostnames

Note that directly linking docker containers like this is deprecated and this feature might be removed completely in future in favor of docker compose or swarm.

#### Deploying microservices in Kubernetes

In previous section, we saw how to deploy microservices within docker using containers and linking them together. Let's see how to do this on a Kubernetes cluster.

On a kubernetes cluster, we can't deploy a docker container directly. The smallest unit for deployment on a Kubernetes cluster is a pod. So for simplicity sake, we will create a pod-definition file and then use that info to create a deployment-definition and then create Services to make the application accessible. Let's see this in the figure below:

{{< figure src="/services.png" alt="Microservices" caption="Kubernetes Microservices" >}}

So let's take a moment to create a high level strategy on how to start with Kubernetes deployment.

Steps:
1. Deploy Pods for the 5 microservices
1. Note that since only `voting-app`, `result-app`, `redis` and `postgres` pods have arrows coming in, meaning only these pods are accessed either from outside or from other pods, we will only need to configure services for these 4 pods. Since `worker` does not have any arrows coming in, it does not get accessed by anyone, so no need to create a service for this.
1. Since the services `redis` and `postgres` are not to be accessed outside the cluster, they will just be of type `ClusterIP`
1. Now that we have decided that `redis` and `postgres` pods will have services of type `ClusterIP` to make them accessible by other parts of application, we need to decide what name to use for those services. Service name is important because that is how you will access the service.
    - As we see in figure captioned `redis`, since we access redis microservice from code using hostname as `redis` in code (in the connection string), we will name the redis service as `redis`
    - As we see in the figure captioned `postgres`, since we access postgres microservice from code using hostname as `db` in code (in the connection string), we will name the postgres service as `db`
    - Also note that in the figure captioned `postgres`, we see that in the connection string for the database, we also supoly a Username and Password. We must ensure that these credentials must be set accordingly when we setup the database inside the pod.
1. Next task is to enable external access. As we know, service of type `NodePort` does this. 
    - Create a service for `voting-app` and set its type to `NodePort`
    - Create a service for `result-app` and set its type to `NodePort`
    - We can decide on what port to make them available on, it would be a high port, with the port number greater than 30000.

{{< figure src="/redis.png" alt="Redis" caption="redis" >}}
{{< figure src="/postgres.png" alt="postgres" caption="postgres" >}}

__Note__: A service is only needed when the application has some kind of process or database or webservice that needs to be exposed, that needs to be accessible by others. So, for example, in the above example, since `worker` app didn't need this accebility by others, it did not need a service.

Lets get to the code.
We mentioned that step 1 is to write pod definition files for all 5 applications we have. In reality, we don't use pod-definition files or replicaset-definition files. The reason being that doing this way causes us to first take down the instances first before rolling new ones, when we are updating the applications to a new version. Also there are no rolling updates, rollbacks etc as deployments offer you. So, in practise, we write deployment-definition files instead of pod-definition files or replicaset definition files. 

`postgres-deploy.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-deploy
  labels:
    name: postgres-deploy
    app: demo-voting-app
spec:
  replicas: 1
  selector:
    matchLabels:
      name: postgres-pod
      app: demo-voting-app
  template:
    metadata:
      name: postgres-pod
      labels:
        name: postgres-pod
        app: demo-voting-app
    spec:
      containers:
      - name: postgres
        image: postgres
        ports:
        - containerPort: 5432
        env:
          - name: POSTGRES_USER
            value: "postgres"
          - name: POSTGRES_PASSWORD
            value: "postgres"
          - name: POSTGRES_HOST_AUTH_METHOD
            value: trust
```

`postgres-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: db
  labels:
    name: postgres-service
    app: demo-voting-app
spec:
  ports:
  - port: 5432
    targetPort: 5432
  selector:
    name: postgres-pod
    app: demo-voting-app
```

`redis-deploy.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-deploy
  labels:
    name: redis-deploy
    app: demo-voting-app
spec:
  replicas: 1
  selector:
    matchLabels:
      name: redis-pod
      app: demo-voting-app
  template:
    metadata:
      name: redis-pod
      labels:
        name: redis-pod
        app: demo-voting-app
    spec:
      containers:
      - name: redis
        image: redis
        ports:
        - containerPort: 6379
```

`redis-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
  labels:
    name: redis-service
    app: demo-voting-app
spec:
  ports:
  - port: 6379
    targetPort: 6379
  selector:
    name: redis-pod
    app: demo-voting-app
```

`result-app-deploy.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: result-app-deploy
  labels:
    name: result-app-deploy
    app: demo-voting-app
spec:
  replicas: 1
  selector:
    matchLabels:
      name: result-app-pod
      app: demo-voting-app
  template:
    metadata:
      name: result-app-pod
      labels:
        name: result-app-pod
        app: demo-voting-app
    spec:
      containers:
      - name: result-app
        image: docker/example-voting-app-result
        ports:
        - containerPort: 80
```

`result-app-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: result-service
  labels:
    name: result-service
    app: demo-voting-app
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30005
  selector:
    name: result-app-pod
    app: demo-voting-app
```

`voting-app-deploy.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: voting-app-deploy
  labels:
    name: voting-app-deploy
    app: demo-voting-app
spec:
  replicas: 1
  selector:
    matchLabels:
      name: voting-app-pod
      app: demo-voting-app
  template:
    metadata:
      name: voting-app-pod
      labels:
        name: voting-app-pod
        app: demo-voting-app
    spec:
      containers:
      - name: voting-app
        image: docker/example-voting-app-vote
        ports:
        - containerPort: 80
```

`voting-app-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: voting-service
  labels:
    name: voting-service
    app: demo-voting-app
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30004
  selector:
    name: voting-app-pod
    app: demo-voting-app
```

`worker-app-deploy.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: worker-app-deploy
  labels:
    name: worker-app-deploy
    app: demo-voting-app
spec:
  replicas: 1
  selector:
    matchLabels:
      name: worker-app-pod
      app: demo-voting-app
  template:
    metadata:
      name: worker-app-pod
      labels:
        name: worker-app-pod
        app: demo-voting-app
    spec:
      containers:
      - name: worker-app
        image: docker/example-voting-app-worker
```
now we will create all the deployments and services:
```shell
# voting app
kubectl create -f voting-app-deploy.yaml
kubectl create -f voting-app-service.yaml
# do a quick check to make sure that above deplyment is in running state
kubectl get deployments
# postgres
kubectl create -f postgres-deploy.yaml
kubectl create -f postgres-service.yaml
# redis
kubectl create -f redis-deploy.yaml
kubectl create -f redis-service.yaml
# do a quick check to make sure that above deplyment is in running state
kubectl get deployments
# make sure that all pods are up as well as the service
kubectl get pods,svc
# worker
kubectl create -f worker-app-deploy.yaml
# make sure that all pods are up as well as the service
kubectl get pods,svc
# result app
kubectl create -f result-app-deploy.yaml
kubectl create -f result-app-service.yaml
# make sure that all deployments are up as well as the service
kubectl get deployments,svc
# get urls for the two frontend services
minikube service voting-service --url
minikube service result-service --url
```