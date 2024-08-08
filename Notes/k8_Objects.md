# Node

- A physical or virtual machine on which Kubernetes is installed
- **Nodes are cluster scoped. They are not scoped within a namespace.**
- When you install Kubernetes on a node, the following components are installed. Some of them are used in worker nodes and the rest are used in master nodes.
    - API Server
    - `etcd` Service
    - Kubelet Service
    - Container Runtime
    - Controller
    - Scheduler
- A **cluster** is a collection of nodes grouped together

## Worker Nodes

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/452dc72d-c049-4f71-b9fa-8e2025f54e0e/Untitled.png)

- These nodes do the actual work so they need to have more resources
- Each worker node has multiple pods running on it
- 3 processes must be installed on every worker node
    - **Container Runtime** (eg. docker)
    - **Kubelet**
        - process of Kubernetes
        - starts pods and runs containers inside them
        - allocates resources from the node to the container
    - **Kubeproxy**
        - process of Kubernetes
        - forwards the requests to pods intelligently
        - Image
            - Kubeproxy forwards requests to the DB pod running on the same node to minimize network overhead.
                
                ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/316cbcf2-401c-4ec7-87e6-ef199bf84adf/Untitled.png)
                

## Master Nodes

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/781c65f7-6cf4-40bd-8613-10806f875524/Untitled.png)

- Control the cluster state & manage worker nodes
- Need less resources as they don't do the actual work
- Multi-master setup is often used for fault tolerance
- 4 processes run on every master node
    - **API Server**
        - User interacts with the cluster via the API server using a client (Kubernetes Dashboard, CLI, or Kubernetes API)
        - Cluster gateway (acts as the entry point into the cluster)
        - Can be used for authentication
        
        ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/c5940aed-14dd-48c9-b59f-71b1c6c82803/Untitled.png)
        
    - **Scheduler**
        - Decides the node where the new pod should be scheduled and sends a request to the Kubelet to start a pod.
        
        ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/dbc7359e-dd5f-4926-9cd8-b525ad323018/Untitled.png)
        
    - **Controller**
        - Detects state changes like crashing of pods
        - If a pod dies, it requests scheduler to schedule starting up of a new pod
        
        ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/cd8f1316-c16f-4845-bb56-fd4e12e145c8/Untitled.png)
        
    - **etcd**
        - Key-value store of the cluster state (also known as cluster brain)
        - Cluster changes get stored in the etcd
        - In multi-master configuration, etcd is a distributed key-value store
        - Application data is not stored in the etcd
     

# Pod

- Kubernetes doesn’t run containers directly on the nodes. Every container is encapsulated by a pod.
- Smallest unit of Kubernetes
- A pod is a single instance of an application. If another instance of the application needs to be deployed, another pod is deployed with the containerized application running inside it.
- Creates a running environment over the container so that we only interact with the Kubernetes layer. This allows us to replace the container technology like Docker.
- **Each pod gets an internal IP address** for communicating with each other (virtual network created by K8)
- If a pod is restarted (maybe after the application running on it crashed), its IP address may change

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/4c2e9f57-62ed-41da-b452-e004c1235410/Untitled.png)

<aside>
⛔ Sometimes we need to have a helper container for the application container. In that case, we can run both containers inside the same pod. This way both containers share the same storage and network and can reference each other as `localhost`

Without using Pods, making a setup like this would be difficult as we need to manage attaching the helper containers to the application containers and kill them if the application container goes down. 

Although, most use cases of pods revolve around single containers, it provides flexibility to add a helper container in the future as the application evolves.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/aa93d68e-3e1e-47c5-be7c-bcd91909f678/Untitled.png)

</aside>

### Config file for a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    name: frontend
spec:
  containers:
	  - name: httpd
	    image: httpd:2.4-alpine
```

# Restart Policy

The default behavior of K8s is to restart a pod if it terminates. This is desirable for long running containers like web applications or databases. But, this is not desirable for short-lived containers such as a container to process an image or run analytics. 

`restartPolicy` allows us to specify when K8s should restart the pod.

- `Always` - restart the pod if it goes down (default)
- `Never` - never restart the pod
- `OnFailure` - restart the pod only if the container inside failed (returned non zero exit code after execution)

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    name: analytics
spec:
  containers:
	  - name: analytics
	    image: analytics
	restartPolicy: Never
```


# ReplicaSet

- ReplicaSet monitors and maintains the number of replicas of a given pod. It will automatically spawn a new pod if the pod goes down.
- It is needed even if we only have a single pod, because if that pod dies, replica set will spawn a new pod.
- It spans the entire cluster to spawn pods on any node.

<aside>
⛔ Newer and better way to manage replicated pods in K8s than Replication Controllers

</aside>

## Config YAML file

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: httpd-frontend
  labels:
    name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      name: frontend
  template:
    metadata:
      labels:
        name: frontend
    spec:
      containers:
      - name: httpd
        image: httpd:2.4-alpine
```

- `template` → `metadata` and `spec` from the config file for the pod (required to spawn new pods if any of them goes down)
- `replicas` → how many replicas to maintain
- It has an additional required field `selector` which allows the replica set to select pods that match specific labels. This way **the replica set can manage pods that were not created by it.**

## Scaling the number of replicas

- **Recommended**: edit the config file and re-apply - `k apply -f config.yaml`
    - Scaling changes can be easily tracked using Git
- **Not recommended**: using `kubectl` - `k scale replicaset my-replicaset --replicas=2`
    - This will not update the config file, so changes are hard to track


# Deployment

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3b557804-128f-4d2c-afb5-2cd0038d746c/Untitled.png)

- **Provides the capability to upgrade the instances seamlessly using rolling updates, rollback to previous versions seamlessly, undo, pause and resume changes as required.**
- Abstraction over [ReplicaSet](https://www.notion.so/ReplicaSet-df784dc061344ab6a5a83f1f61652f1c?pvs=21)
- **Blueprint for stateless pods** (application layer)
- When a deployment is created, it automatically creates a [ReplicaSet](https://www.notion.so/ReplicaSet-df784dc061344ab6a5a83f1f61652f1c?pvs=21)  which in turn creates pods. If we run `k get all` we can see the resources created by deployment.

# Config YAML file

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd-frontend
  labels:
    name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      name: frontend
  template:
    metadata:
      labels:
        name: frontend
    spec:
      containers:
      - name: httpd
        image: httpd:2.4-alpine
```

<aside>
⛔ Update the `kind` from `Replicaset` to `Deployment` in a `Replicaset` config file.

</aside>

# Deployment Strategy

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/6bea30ab-259f-4ef2-9289-98661435d916/Untitled.png)

There are two deployment strategies:

- **Recreate**: Bring down all the running containers and then bring up the newer version (application downtime)
- **Rolling Update** (default): Bring down a container and then bring up a new container one at a time (no application downtime)

# Rollout and Versioning

- When you first create a deployment, it triggers a rollout which creates the first revision of the deployment. Every subsequent update to the deployment triggers a rollout which creates a new revision of the deployment. This keeps a track of the deployment and helps us rollback to a previous version of the deployment if required.
- **When we upgrade the version of a deployment, a new replica set is created** under the hood where the new pods are spawned while bringing down pods from the old replica set one at a time, following a rolling update strategy. We can see the new and old replica sets by running `k get replicasets`
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/14a3e225-ab17-4160-bad3-df50151875f7/Untitled.png)
    
- `k rollout status deployment <deployment-name>` - view the status of rollout for a deployment
- `k rollout history deployment <deployment-name>` - view the history of rollouts for a deployment
- `k rollout history deployment <deployment-name> --revision=1` - view the status of rollouts for a deployment revision

# Rollback

- When we rollback to the previous version of a deployment, the pods in the current replica set are brought down one at a time while spawning pods in the previous replica set.
- Rollback a deployment - `k rollout undo deployment <deployment-name>`
- Rollback a deployment to a specific revision - `k rollout undo deployment <deployment-name> --to-revision=1`

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/56ce73da-26b2-41de-99e7-ccfcdc1cf864/Untitled.png)


# Service

- Kubernetes services enable communication between various components within and outside of the application. They enable loose coupling between micro-services in our application.
- **Services are static IPs that can be attached to a pod or a group of pods using label selectors. They are not attached to deployments.**
- Services prevent us from using the pod IP addresses for communication which could change when the pod is restarted.
- **Lifecycle of pod and service are not connected.** So even if a pod dies, we can restart it and attach the original service to have the same IP.
- **Every service spans the entire cluster (all the nodes in the cluster)**
- **Every service has a unique IP across the K8s cluster**
- Kubernetes creates a default ClusterIP Service which forwards requests from within the cluster to the Kubernetes master (API Server). So, there is at least 1 service in every Kubernetes cluster.
- K8s services are of three types:
    - NodePort
    - ClusterIP
    - LoadBalancer

# NodePort Service

Consider an application running in a pod on a node which is on the same network as our laptop, we could SSH into the node and then reach the application by its IP on the Kubernetes network (`10.244.0.0/16`). But doing an SSH into the node to access the application is not the right way. 

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/6e5589df-1fa4-4743-82b7-2ffe3f04d293/Untitled.png)

- NodePort service maps a port on the node (Node Port) to a port on the pod (Target Port) running the application. This will allow us to reach the application on the node’s IP address.
- Allowed range for NodePort: 30,000 - 32,767

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/9a6d869a-5911-4521-b9f9-22e8a6c2cadd/Untitled.png)

```yaml
apiVersion: v1
kind: Service
metadata:
	name: myapp-service
spec:
	type: NodePort
	ports:
		- targetPort: 80
			port: 80
			nodePort: 30008
	selector:
		app: myapp
		type: front-end
```

- `selector` is used to select target pods for the service
- `port` - port on which the service would be accessible
- `targetPort` - port on the pod to which the requests would be forwarded
- `nodePort` - port on the node

- If there are multiple target pods on the same node, the service will automatically load balance to these pods.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/d6678804-800e-4b99-bb79-3d1e16320bf1/Untitled.png)

- If the target pods span multiple nodes in the cluster, as the NodePort service will span the entire cluster, it will map the target port on the pods to the same node port on all the nodes in the cluster, even the nodes that don’t have the application pod running in them. This will make the application available on the IP addresses of all of the nodes in the cluster.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/145f1b12-fb16-4baa-b7d1-2d9148cc43e2/Untitled.png)

# ClusterIP Service

- Consider a 3-tier application running on a K8s cluster. How will different tiers communicate with each other? Using IPs to communicate is not good as it can change when the pods are restarted. Also, how can we load balance if we have multiple pods in the same tier.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/701c4854-d3cb-4455-a6ad-d90a84b3113c/Untitled.png)

- ClusterIP Service groups similar pods and provides a single interface to access those pods. We don’t have to access the pods using their IP addresses.
- Enables access to the service from within the K8s cluster (internal)
- It automatically load balances to the target pods.
- **Service name should be used by other pods to communicate with the service.**
- Useful in deploying micro-services architecture on a K8s cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
	name: back-end
spec:
	type: ClusterIP
	ports:
		- targetPort: 80
			port: 80
	selector:
		app: myapp
		type: back-end
```

- `selector` is used to select target pods for the service
- `port` - port on which the service would be accessible
- `targetPort` - port on the pod to which the requests would be forwarded

# LoadBalancer Service

- Consider the case where we have to route the incoming traffic to the front-end of two applications. If the applications are using NodePort service, they can be accessed at different node ports using the IPs of any of the nodes. But, we cannot use higher order ports for our application as they are non standard. Also, how do we load balance to the nodes (the application can be accessed at any of the nodes by using their IP addresses). For these, we need to use a Load Balancer service.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/17ffc263-7cce-4f9a-af51-89e7f0982a5a/Untitled.png)

- LoadBalancer Service leverages the native layer-4 load balancer of the cloud provider to expose the application on a single IP (NLB’s IP) and load balance to the nodes. So, if a node becomes unhealthy, the NLB will redirect the incoming requests to a healthy node.
- Any NodePort service can be converted to use the cloud provider’s load balancer by setting `type: LoadBalancer` in the config.yaml file.
- LoadBalancer uses NodePort service behind the scenes and sets up a layer-4 load balancer of the cloud provider to load balance to the nodes on the high order node port.

```yaml
apiVersion: v1
kind: Service
metadata:
	name: myapp-service
spec:
	type: LoadBalancer
	ports:
		- targetPort: 80
			port: 80
			nodePort: 30008
	selector:
		app: myapp
		type: front-end
```

<aside>
💡 The downside of this is that each service that you expose will require its own public IP (NLB) as NLB cannot redirect to specific application based on URL or path. This makes this approach expensive if we have multiple applications to load balance to. The solution is to use an [Ingress](https://www.notion.so/Ingress-9fe828fdf67b42d09b0da2a4579ad636?pvs=21), which is an ALB present within the cluster with an single NLB present outside to load balance to the nodes.

</aside>

<aside>
⛔ LoadBalancer Service is only supported on some cloud platforms. On non-supported cloud providers or local machines, it will behave as a NodePort service.

</aside>


# Namespaces

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7040c4d8-3fe4-4fb2-8f62-126d11982770/Untitled.png)

- Namespaces isolate resources within a K8s cluster.
- K8s creates a `default` namespace when the cluster is created. This default namespace is used to create resources.
- If the cluster is deployed using KubeAdmin, it also creates a namespace `kube-system` in which all the internal K8s resources are deployed.
- Resource limits can be placed at the namespace level. So, if we are using the same cluster for both `dev` and `prod` namespaces, we can place a resource limit on the `dev` namespace to prevent it from starving the `prod` namespace.

## DNS Resolution

- Resources within a namespace can refer to each other by their names.
- For cross namespace communication, a resource needs to specify the namespace as shown below.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/c5691031-69f8-4fb9-93c7-a68a29a4b591/Untitled.png)

`cluster.local` - domain name for the cluster

`svc` - subdomain for service object

`dev` - namespace

`db-service` - service in the `dev` namespace

# Creating a namespace

- Imperative command: `k create namespace <namespace>`
- Declarative manifest file
    
    ```yaml
    apiVersion: v1
    kind: Namespace
    metadata:
    	name: dev
    ```
    

# Creating resources in a namespace

- Command line: `k apply -f pod.yml --namespace=dev` (untracked)
- Config file (tracked): Use the namespace `property` under the metadata section. This will always create the resource in the specified namespace.
    
    ```yaml
    metadata:
    	namespace: dev
    ```
    

# Set namespace permanently

`k config set-context $(kubectl config current-context) --namespace=dev set-context`

# Specify Resource Quota for a Namespace

Create a K8s `ResourceQuota` and specify the namespace in the `metadata` section.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
	name: compute-quota
	namespace: dev
spec:
	hard:
		pods: "10"
		requests.cpu: "4"
		requests.memory: 5Gi
		limits.cpu: "10"
		limits.memory: 10Gi
```

# Namespace vs Cluster Scope

Some objects in K8s are not scoped under a namespace, but are scoped under the whole cluster. 

### Namespace scoped

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/a5238583-28f0-478a-bede-6bf85fda1ef2/Untitled.png)

### Cluster Scoped

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/c3d3c2e5-619c-4a29-9879-619d7e30efa9/Untitled.png)


NetworkPolicies
Network policies help control the flow of ingress and egress traffic to a group of pods (using label selectors). 
By default, every pod in the cluster can reach out to any other pod within the cluster. This means, there’s no network policy by default.
There’s no need for a network policy to allow the response. If the request is allowed by the network policy, the response will be allowed automatically (stateful behavior).
Network policies are implemented by the networking solution used in the cluster. Currently, Flannel doesn’t support NetworkPolicies.
The resulting network policy for a pod is the union of all the network policies associated with it. The order of rule evaluation does not matter.
Network policies are firewalls applied directly to the matching pods (not through services).
3-Tier Web Application Example
In a 3 tier web application, the users should be able to reach the web service on port 80 or the API service on port 5000. Also, the DB service should only be reachable by the API service. 

These are the following traffic that should be allowed for each pod (service):
Web service
Ingress on port 80 from anywhere
Egress on port 5000
API service
Ingress on port 5000 from anywhere
Egress on port 3306
DB service
Ingress on port 3306 from API pod
Network Policy for DB pod
Label the DB pod as role: db and API pod as role: api. We can use these labels in the NetworkPolicy definition file to allow ingress traffic on port 3306 only from API pods. We don’t need to create an egress rule for the response from the the DB pod to the API pod as it is allowed automatically.

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
	name: db-policy
spec:
	podSelector:
		matchLabels:
			role: db
	policyTypes:
		- Ingress
	ingress:
		- from:
			- podSelector:
					matchLabels:
						role: api
			ports:
				- protocol: TCP
					port: 3306
​
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
	name: db-policy
spec:
	podSelector:
		matchLabels:
			role: db
	policyTypes:
		- Ingress
	ingress:
		- from:
			- podSelector:
					matchLabels:
						role: api
				namespaceSelector:
					matchLabels:
						name: prod
			ports:
				- protocol: TCP
					port: 3306
​
To restrict access to the DB pod to happen within the current namespace, select the namespace using namespaceSelector. In the example, only the API pods of prod namespace can connect to the DB pod in the prod namespace.

Allowing Ingress Traffic from outside the Cluster
If we want to allow a backup server (192.168.5.10) present outside the cluster but within the same private network to pull data from the DB pod to perform backups, we can specify its IP address in the DB pod’s ingress rule. Now, the DB pod allows ingress traffic on port 3306 from both API pod and the backup server.
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
	name: db-policy
spec:
	podSelector:
		matchLabels:
			role: db
	policyTypes:
		- Ingress
	ingress:
		- from:
			- podSelector:
					matchLabels:
						role: api
			- ipBlock:
					cidr: 192.168.5.10/32
			ports:
				- protocol: TCP
					port: 3306
​```
Allowing Egress Traffic to outside the Cluster
If the DB pod needs to push a backup to a backup server (192.168.5.10) present outside the cluster but within the same private network, we can create an egress rule on the DB pod’s NetworkPolicy.
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
	name: db-policy
spec:
	podSelector:
		matchLabels:
			role: db
	policyTypes:
		- Ingress
		- Egress
	ingress:
		- from:
			- podSelector:
					matchLabels:
						role: api
			ports:
				- protocol: TCP
					port: 3306
	egress:
		- to:
			- ipBlock:
					cidr: 192.168.5.10/32
			ports:
				- protocol: TCP
					port: 80
```


ConfigMap
Centrally managed configuration data that can be passed to the containers as environment variables (key-value pairs). 
Storing config data along with the pod/deployment definition file is not a good idea because as the application grows, managing them would become difficult.
Should be used to store parameters that are not secrets
⚠️
The data stored in the ConfigMap, when the container (pod) is created, is used to set the environment variables. If the ConfigMap gets updated later, the pod will continue to use the old values. We need to re-create the pods by performing a rollout (k rollout restart deployment <deployment-name>) on the deployment to make the new pods use the new data.
ConfigMap definition file
apiVersion: v1
kind: ConfigMap
metadata:
	name: app-config
data:
	USERNAME: arkalim
	PASSWORD: 12345
​
Using file as a ConfigMap:
apiVersion: v1
kind: ConfigMap
metadata:
  name: rho-pacs-config
  namespace: 16bit
data:
  orthanc.json: |
    {
      /**
      * General configuration of Orthanc
      **/
    }
​
Using ConfigMap in Pods
Passing the entire ConfigMap of key-value pairs to ENV
apiVersion: v1
kind: Pod
metadata:
  labels:
    name: app
spec:
  containers:
	  - name: nginx
	    image: nginx
			envFrom:
				- configMapRef:
					  name: app-config
​
Passing a single key-value pair from the ConfigMap to ENV
apiVersion: v1
kind: Pod
metadata:
  labels:
    name: app
spec:
  containers:
	  - name: nginx
	    image: nginx
			env:
				- name: USERNAME
					valueFrom:
						configMapKeyRef:
							name: app-config
							key: USERNAME
​
Passing a config file as ConfigMap (eg. nginx.conf) by mounting the ConfigMap as a volume
apiVersion: v1
kind: Pod
metadata:
  labels:
    name: app
spec:
  containers:
	  - name: nginx
	    image: nginx
			volumeMounts:
        - name: nginx-config-volume
          mountPath: /etc/nginx/conf.d/
  volumes:
    - name: nginx-config-volume
      configMap:
        name: nginx-config
​
The mount path must be a directory. If passing a config file, only pass the full path, not the filename.


# Secret

- Just like [ConfigMap](https://www.notion.so/ConfigMap-9f55290c5ef141298140a28d80222c19?pvs=21) but used to store secrets instead of parameters
- Stores the data in `base64` encoded format
    
    To encode a base64 string - `echo -n '<string>' | base64`
    
- Encryption at rest is not enabled by default. See [Encrypting Secret Data at Rest | Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/). Storing secrets in 3rd-party secrets store provided by cloud providers is another good option.

<aside>
⚠️ The data stored in the Secret, when the container (pod) is created, is used to set the environment variables. If the Secret gets updated later, the pod will continue to use the old value. We need to re-create the pods by performing a rollout (`k rollout restart deployment <deployment-name>`) on the deployment to make the new pods use the new data.

</aside>

### Secret definition file

Same as [ConfigMap](https://www.notion.so/ConfigMap-9f55290c5ef141298140a28d80222c19?pvs=21) except the `kind` and the base64 encoded values.

```yaml
apiVersion: v1
kind: Secret
metadata:
	name: app-secret
data:
	USERNAME: adfcfe==
	PASSWORD: asdgfgv==
```

<aside>
💡 To view the secrets along with their encoded values, run
`k get secret <secret-name> -o yaml`

</aside>

### Using Secrets in Pods

- Passing the entire Secret of key-values pairs to ENV
    
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      labels:
        name: app
    spec:
      containers:
    	  - name: httpd
    	    image: httpd:2.4-alpine
    			envFrom:
    				- secretRef:
    					 name: app-secret
    ```
    
- Passing a single key-value pair of the secret to ENV
    
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      labels:
        name: app
    spec:
      containers:
    	  - name: httpd
    	    image: httpd:2.4-alpine
    			env:
    				- name: PASSWORD
    					valueFrom:
    						secretKeyRef:
    							name: app-secret
    							key: PASSWORD
    ```
    
- Passing a file as Secret by mounting the Secret as a volume
    
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      labels:
        name: app
    spec:
      containers:
    	  - name: nginx
    	    image: nginx
    			volumeMounts:
            - name: nginx-secret-volume
              mountPath: /etc/nginx/conf.d/
      volumes:
        - name: nginx-secret-volume
          secret:
            name: nginx-secret
    ```



  # Volume

---

- A volume is a persistent storage which could be created and mounted at a location inside the containers of a pod. This allows the pod to persist the storage at that location even if it is restarted.
- Volume could be:
    - **Local** (on the same node as the pod) - This is not acceptable if the cluster has multiple worker nodes as each node will store different data in their volumes.
    - **Remote** (outside the cluster) - This works with multiple worker nodes as the storage is being managed remotely. The remote storage provider must follow the **Container Storage Interface (CSI)** standards.

## Creating a local volume on the node

The pod definition file below creates a volume at location `/data` on the node and mounts it to the location `/opt` in the container. The volume is created at the pod level and it mounted at the container level.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    name: frontend
spec:
  containers:
	  - name: httpd
	    image: httpd:2.4-alpine
			volumeMounts:
				- name: data-volume
					mountPath: /opt

	volumes:
		- name: data-volume
			hostPath: 
				path: /data
				type: Directory
```

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/850dbd9d-6f10-402f-9ffb-a44f38d98989/Untitled.png)

## Creating a shared remote volume on EBS

The pod definition file below creates a volume on EBS and mounts it to the location `/opt` in the container. Even if the pods are running on multiple nodes, they will still read the same data.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    name: frontend
spec:
  containers:
	  - name: httpd
	    image: httpd:2.4-alpine
			volumeMounts:
				- name: data-volume
					mountPath: /opt

	volumes:
		- name: data-volume
			awsElasticBlockStore: 
				volumeId: <volume-id>
				fsType: ext4
```

<aside>
⚠️ Configuring volumes at the pod level (in every pod definition file) is not the right way. If we want to switch all the volumes from local to remote, we need to update every pod definition file.

</aside>

# Persistent Volumes

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/54ebfd71-77e8-4814-935c-552fd4978045/Untitled.png)

- **Persistent Volumes (PVs) are cluster wide storage volumes configured by the admin.** This allows the volumes to be centrally configured and managed by the admin. The developer creating application (pods) can claim these persistent volumes by creating **Persistent Volume Claims (PVCs).**
- Once the PVCs are created, K8s binds the PVCs with available PVs based on the requests in the PVCs and the properties set on the volumes. **A PVC can bind with a single PV only** (there is a 1:1 relationship between a PV and a PVC). If multiple PVs match a PVC, we can label a PV and select it using label selectors in PVC.
- A smaller PVC can bind to a larger PV if all the other criteria in the PVC match the PV’s properties and there is no better option.
- When a PV is created, it is in **Available** state until a PVC binds to it, after which it goes into **Bound** state. If the PVC is deleted while the reclaim policy was set to `Retain`, the PV goes into **Released** state.
- If no PV matches the given criteria for a PVC, the PVC remains in **Pending** state until a PV is created that matches its criteria. After this, the PVC will be bound to the newly created PV.
- The properties involved in binding between PV and PVC are: Capacity, Access Modes, Volume Modes, Storage Class and Selector.
- List persistent volumes - `k get persistentvolume` or `k get pv`
- List persistent volume claims - `k get persistentvolumeclaim` or `k get pvc`

### PV definition file

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
	name: pv-vol
spec:
	accessModes:
		- ReadWriteOnce
	capacity:
		storage: 1Gi
	hostPath:
		path: /tmp/data
```

- `accessModes` defines how the volume should be mounted on the host. Supported values:
    - `ReadOnlyMany`
    - `ReadWriteOnce`
    - `ReadWriteMany`
- `hostPath` can be replaced with remote options such as `awsElasticBlockStore`

### PVC definition file

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
	name: myclaim
spec:
	accessModes:
		- ReadWriteOnce
	resources:
		requests:
			storage: 500Mi
```

- `requests` specifies the requested properties by the PVC
- This PVC will bind to the above PV if there is no other PV smaller than `1Gi` and at least `500Mi`

### Using PVCs in Pods

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: frontend
      image: nginx
      volumeMounts:
	      - mountPath: "/var/www/html"
	        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim
```

Only the volume defined at the pod level will be modified to reference the PVC. 

<aside>
💡 If we delete a PVC which is being used by a pod, it will be stuck in **Terminating** state. If the pod is deleted afterwards, the PVC will get deleted after the pod’s deletion.

</aside>

### Reclaim Policy

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
	name: pv-vol
spec:
	persistentVolumeReclaimPolicy: Retain
	accessModes:
		- ReadWriteOnce
	capacity:
		storage: 1Gi
	hostPath:
		path: /tmp/data
```

`persistentVolumeReclaimPolicy` governs the behavior of PV when the associated PVC is deleted. Possible values:

- `Retain` - retain the PV until it is manually deleted but it cannot be reused by other PVCs (default)
- `Delete` - delete PV as well
- `Recycle` - erase the data stored in PV and make it available to other PVCs


# Storage Classes

## Static Provisioning

In [Volume](https://www.notion.so/Volume-3afba1ed481249dea86d81f0a522aeed?pvs=21) we discussed how we can create a PV and a PVC to bind to that PV and finally configure a pod to use the PVC to get a persistent volume. 

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/0226e759-29e4-4c2e-8c8a-91b0df1ebe9b/Untitled.png)

The problem with this approach is that we need to manually provision the storage on a cloud provider or storage device before we can create a PV using it. This is called as **static provisioning.**

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/5327cd36-1365-4264-9cf9-0f7e28aa7427/Untitled.png)

## Dynamic Provisioning

In dynamic provisioning, a **provisioner** is created which can automatically provision storage on the cloud or storage device and attach them to the pod when the claim is made. **Dynamic provisioning is achieved by creating a `StorageClass` object.** 

When using storage classes, we don’t need to create PVs manually. When a PVC is created with a storage class, the storage class uses a provisioner to automatically provision storage and create a PV to bind to the PVC.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
	name: gcp-storage
provisioner: kubernetes.io/gce-pd
```

`provisioner` depends on the type of underlying storage being used (EBS, AzureDisk, etc.) 

`provisioner: kubernetes.io/no-provisioner` means dynamic provisioning is disabled.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
	name: myclaim
spec:
	storageClassName: gcp-storage
	accessModes:
		- ReadWriteOnce
	resources:
		requests:
			storage: 500Mi
```

Depending on the `provisioner`, there are some properties that we can specify to control the behavior of the underlying storage. 

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
	name: gcp-storage
provisioner: kubernetes.io/gce-pd
parameters:
	type: pd-ssd
	replication-type: regional-pd
```

Using these properties, we can create classes of storage such as silver, gold & platinum with increasing levels of replication and speed.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3f44768c-d811-47f2-8fff-f5cdbc5a41f4/Untitled.png)

### Reclaim Policy and Volume Binding Mode

`reclaimPolicy` defines the behavior of the PV when the PVC is deleted

- `Delete` - delete the PV when the PVC is deleted
- `Retain` - retain the PV when the PVC is deleted

`volumeBindingMode` defines when the volume should be created and bound to a PVC

- `WaitForFirstConsumer` - wait for a pod to use the PVC
- `Immediate` - immediately create a volume and bind it to the PVC

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
	name: gcp-storage
provisioner: kubernetes.io/gce-pd
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

# StatefulSets

StatefulSet is used instead of [Deployment](https://www.notion.so/Deployment-aaa3756097d1452a9d42cc3e493a36c6?pvs=21) when we want to deploy stateful pods (such as databases) with replication between them. One of the database pods is set up as master and the rest as slaves. 

StatefulSet is similar to [Deployment](https://www.notion.so/Deployment-aaa3756097d1452a9d42cc3e493a36c6?pvs=21). It’s a template to deploy pods. It supports scaling, updates, rollbacks etc. 

StatefulSet deploys pods in a sequential order (ordered, graceful deployment). Only after the first pod is in a running state, the next pod will be deployed. This helps ensure that the master pod is deployed first and only then the slaves are brought up one by one. When scaled in or during deletion of the StatefulSet, the pods are brought down sequentially in the reverse order.

StatefulSets assign an ordinal pod name to each pod as they are brought up. This goes as `<stateful-set-name>-x` where `x` can be 0, 1, 2, 3, and so on. This means the master pod in any StatefulSet will be named `<stateful-set-name>-0`. Using a **headless service** allows us to use these ordinal pod names to form DNS names for these pods. This way, we can configure the database running in the slave pods to reach out to the master database at a predictable hostname.

<aside>
💡 K8s deployment object cannot be used in this scenario since it brings up all the pods at the same time without any fixed order. Also, the pod names generated have a random slug which can change if the pod is restarted. So, the master pod cannot have its pod name fixed. This means the slave pods cannot reach the master pod reliably to setup continuous replication.

</aside>

## SatefulSet definition file

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
  labels:
    name: db
spec:
	serviceName: mysql-h
  replicas: 3
  selector:
    matchLabels:
      name: db
  template:
    metadata:
      labels:
        name: db
    spec:
      containers:
      - name: db
        image: mysql
```

StatefulSet definition file is written the same way a deployment definition file is written. Only the `kind` is changed and a `serviceName` property is added to the `spec` section which points to a headless service. 

The StatefulSet uses the headless service to create unique predictable DNS records to reach a specific pod in the StatefulSet.

# Headless Service

**A headless service creates a predictable DNS entry for each pod in a StatefulSet.** This allows any other pod in the cluster to reach any pod in the StatefulSet by its DNS name. A headless service does not load balance the requests like any other service in K8s. It instead routes the request to a specific pod in the StatefulSet.

In the diagram, green service is load balancing the read requests coming from the web pod to the database pods. The headless service `mysql-h` creates DNS entries for each database pod. This allows the web pod to reach the master database pod `mysql-0` to perform writes.

The DNS names of the pods are `<pod-name>.<headless-service-dns>`

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/34347868-5313-46d3-8a7e-20857cb5009b/Untitled.png)

```yaml
apiVersion: v1
kind: Service
metadata:
	name: mysql-h
spec:
	ports:
		- port: 3306
	selector:
		app: mysql
	clusterIP: None
```

**Setting the `clusterIP: None` in a service definition file makes it headless.** In the example, port 3306 is the port on which the headless service will route the incoming requests to the pod based on the DNS name. The selector is used to select the pods in the StatefulSet and create DNS entries based on the pod name and the cluster domain.

# Storage in StatefulSets

### PV shared between pods

Attaching a PVC (with a storage class configured) to the database pods will provision a PV and mount all the pods to that PV. This means all the pods (instances of the application) will share the same storage volume. 

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/cfd42167-75f2-47ec-a8fd-f029879fa17b/Untitled.png)

Note that reads/writes by multiple instances at the same time is not supported by all the volume types.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
	name: mysql
	labels:
		app: mysql
spec:
	serviceName: mysql-h
	replicas: 3
	selector:
		matchLabels:
			app: mysql
	template:
		metadata:
			labels:
			app: mysql
		spec:
			containers:
				- name: mysql
					image: mysql
					volumeMounts:
						- name: data-volume
							mountPath: /var/lib/mysql
			volumes:
				- name: data-volume
					persistentVolumeClaim:
						claimName: data-volume
```

### Dedicated PV for each Pod

We can configure the StatefulSet such that each database pod creates a PVC (with a storage class configured) to provision a dedicated PV for itself. This will allow us to implement read-replicas at the database layer.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/fe60c621-2d74-4b4c-9c1e-4f3a0ae9bcbd/Untitled.png)

This can be done by moving the PVC definition as a template into the StatefulSet definition file under the StatefulSet `spec` section. We can specify multiple PVC templates under the `volumeClaimTemplates` section.

<aside>
💡 If one of the DB pods is restarted, StatefulSet does not delete the associate PV and create a new one upon the pod recreation. Instead, it reattaches the original PV to the restarted DB pod. Therefore, StatefulSets ensure stable storage for stateful pods.

</aside>

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
	name: mysql
	labels:
		app: mysql
spec:
	serviceName: mysql-h
	replicas: 3
	selector:
		matchLabels:
			app: mysql
	template:
		metadata:
			labels:
				app: mysql
		spec:
			containers:
				- name: mysql
					image: mysql
					volumeMounts:
						- name: data-volume
							mountPath: /var/lib/mysql
			volumes:
				- name: data-volume
					persistentVolumeClaim:
						claimName: data-volume
	volumeClaimTemplates:
		- metadata:
				name: myclaim
			spec:
				storageClassName: gcp-storage
				accessModes:
					- ReadwriteOnce
				resources:
					requests:
						storage: 500Mi
```

# Jobs

K8s Jobs are used to run a set of pods to perform a given task. Unlike [ReplicaSet](https://www.notion.so/ReplicaSet-df784dc061344ab6a5a83f1f61652f1c?pvs=21), they are used to orchestrate short-lived pods (jobs). When a job is run, it creates a pod that should run to completion.

### Job definition file

The below file will create a K8s job to run 3 pods sequentially. If any of the pods fail to complete, it will be terminated and another pod will be spawned in its place such that 3 pods run successfully in total.

`template` defines the pod that should be run for the job execution.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: math-add-job
spec:
  completions: 3
  template:
    spec:
			restartPolicy: Never
      containers:
	      - name: math-add
	        image: ubuntu
					command: ['expr', '3', '+', '2']
```

### Parallelism

The above job can also be executed in parallel using the `parallelism` property which signifies the maximum number of pods that can be run in parallel at any given time. 

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: math-add-job
spec:
  completions: 3
	parallelism: 3
  template:
    spec:
			restartPolicy: Never
      containers:
	      - name: math-add
	        image: ubuntu
					command: ['expr', '3', '+', '2']
```

### Failure Retries & Execution Deadline

Max retries for this job = 5 

Max execution time = 100s

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: very-long-pi
  namespace: ckad-job
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl:5.34.0
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(1024)"]
      restartPolicy: Never
  backoffLimit: 5
  activeDeadlineSeconds: 100
```

### Commands

- `k get jobs` - list jobs
- `k get pods` - list job executions (search for job name in the pods)

# Cron Jobs

Cron jobs are [Jobs](https://www.notion.so/Jobs-b564d9a03e2b4465ae747118ac5b05a1?pvs=21) that can be run on a schedule. Every cronjob creates a Job object on a schedule. Example: running analytics every night.

### CronJob definition file

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: analytics
spec:
  schedule: "* * * * *"
  jobTemplate:
    spec:
      completions: 3
      parallelism: 3
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: analytics
              image: analytics
```

### Commands

- List all cron jobs - `k get cronjobs`
- List jobs created by cron jobs - `k get jobs`
- List job executions by cron jobs - `k get pods`

# Ingress

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/df5a4a71-7b62-4246-8105-0e097cf3c9e1/Untitled.png)

It’a single entry point into the cluster. It’s basically a layer-7 load balancer that is managed within the K8s cluster. It provides features like **SSL termination**, and **request based routing** to different services. 

Ingress uses an existing reverse proxy solution like Nignx or Traefik to run an **Ingress Controller**. Then a set of ingress rules are configured using definition files. These are called as **Ingress Resources**. **A K8s cluster does not have an ingress controller by default.** If you just configure ingress resources, it won’t work. 

**Note: Ingress Controllers are not just regular reverse-proxy solutions. They have additional intelligence built into them to monitor the K8s cluster for new ingress resources and configure themselves accordingly. The ingress controller needs a service account to do this.**

**The ingress controller requires a NodePort Service to be exposed at a node port on the cluster.** Alternatively, **the ingress controller requires a LoadBalancer Service to be exposed as a public IP.** DNS server can then be configured to point to the IP of the cloud-native NLB. 

# Deploying Ingress Controller

- An ingress controller is deployed as just another deployment in K8s.
- The example below uses the build image of ingress controller using Nginx as reverse proxy.
- The program to run the ingress controller is present at `/nginx-ingress-controller` which is passed as `args`.
- Nginx requires some configuration to run as expected. Instead of configuring these in the deployment definition file. These should be decoupled into a ConfigMap `nginx-configuration`. This config file is passed in `args` as well.
- Nginx ingress controller requires two environment variables `POD_NAME` and `POD_NAMESPACE` to be passed. This can be fetched from the metadata.
- Finally, expose the ports on the ingress controller pod to allow the service to reach the pod.

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
	name: nginx-ingress-controller
spec:
	replicas: 1
	selector:
		matchLabels:
			name: nginx-ingress
	template:
		metadata:
			labels:
				name: nginx-ingress
		spec:
			containers:
				- name: nginx-ingress-controller
					image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.21.0
			args:
				- /nginx-ingress-controller
				- --configmap=$(POD_NAMESPACE)/nginx-configuration

			env:
				- name: POD_NAME
					valueFrom:
						fieldRef:
							fieldPath: metadata.name
				- name: POD_NAMESPACE
					valueFrom:
						fieldRef:
							fieldPath: metadata.namespace

			ports:
				- name: http
					containerPort: 80
				- name: https
					containerPort: 443
			
```

A **NodePort service** can then be configured to make the ingress controller accessible at a node port in the cluster. 

```yaml
apiVersion: v1
kind: Service
metadata:
	name: nginx-ingress
spec:
	type: NodePort
	ports:
		- port: 80
			targetPort: 80
			protocol: TCP
			name: http
		- port: 443
			targetPort: 443
			protocol: TCP
			name: https
	selector:
		name: nginx-ingress
```

# Ingress Resource

**Ingress resources are set of rules and configuration applied on the ingress controller.** This includes path based routing, subdomain based routing, etc. The `backend` in the ingress definition file defines the service name and the port at which the application service is running.

**For every hostname or domain name, we need a separate rule. For each rule, we can route traffic based on the path.**

### Ingress to route all traffic to a backend service

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
	name: ingress-wear
spec:
	backend:
		serviceName: wear-service
		servicePort: 80
```

### Path based routing

In the example below, we have a single hostname (1 rule) and 2 paths.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/85c6fa38-9dbd-4fdf-8e08-c4603066296c/Untitled.png)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
	name: ingress-wear
spec:
	rules:
		- http:
				paths:
					- path: /wear
						pathType: Prefix
						backend:
							service:
								name: wear-service
								port: 
									number: 80
					- path: /watch
						pathType: Prefix
						backend:
							service:
								name: watch-service
								port: 
									number: 80
```

If none of the rules or paths match, then the user will be redirected to the default backend service if it is configured in the ingress resource. So, you should deploy a service by that name if you want to display a nice 404 not found message. 

### Hostname based routing

In case of routing to different hostnames, we need separate rules.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/0b54c1e7-8240-49f3-adee-7a5992febc11/Untitled.png)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
	name: ingress-wear
spec:
	rules:
		- host: wear.my-online-store.com
		  http:
				paths:
					- backend:
							service:
								name: wear-service
								port: 
									number: 80
		- host: watch.my-online-store.com
		  http:
				paths:
					- backend:
							service:
								name: wear-service
								port: 
									number: 80
```

### Rewrite target

Rewrite target rewrites the URL by replacing whatever is under `rules->http->paths->path` which happens to be `/pay` in this case with the value in `rewrite-target`. This works just like a search and replace function.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
	name: ingress-wear
	annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
	rules:
		- host: wear.my-online-store.com
		  http:
				paths:
					- backend:
							service:
								name: wear-service
								port: 
									number: 80
```

# Demo using Traefik as Ingress Controller

The first video in the playlist demonstrates setting up MetalLB as the load balancer on a VM to allow the ingress service to be accessed externally at the IP of the VM and setting up Traefik ingress controller. The second video shows examples of ingress resources to route traffic to different backend services.


# Service Account

---

Kubernetes has two types of accounts:

- **User Accounts** (cluster wide) - used by humans (eg. developer, admin, etc.)
- **Service Accounts** (namespace bound) - used by external applications to interact with the K8s cluster
    - Prometheus uses a service account to poll the K8s API for performance metrics
    - Jenkins uses service account to deploy applications on a K8s cluster
    - A custom application to display all the pods in a cluster needs to use a service account to get this information from the K8s API.

### Commands

- Get service accounts - `k get serviceaccount`
- Describe a service account - `k describe serviceaccount <service-account-name>`
- Create service account - `k create serviceaccount <service-account-name>`

### External Application

When a service account is created, it generates a token to be used by the external application to authenticate to the K8s API. It then creates a secret object and stores the token as a secret. The secret object is then linked to the service account. The token can be viewed by describing the secret object. This token can be used as a Bearer token when making calls to the [KubeAPI Server](https://www.notion.so/KubeAPI-Server-5a11ac27599b409a8e432675780d11ee?pvs=21).

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3dfccf86-fa72-450d-bd04-e400600e9726/Untitled.png)

### Internal Application

If the application accessing the K8s API is a part of the K8s cluster itself, the process of sharing the token with the application can be made simpler by mounting the secret object as a volume in the pod of the application. That way the token is available to the application inside the pod and we don’t have to provide it manually. So, any process within the pod can access the token to query the K8s API.

### Default Service Account

For every namespace, a `default` service account is created automatically. When a pod is created in a namespace, the default service account is automatically associated with the pod and its token (secret object) is automatically mounted to the pod as a volume mount at location `/var/run/secrets/kubernetes.io/serviceaccount`. This can be viewed by describing the pod. 

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/2933248f-637b-4318-b261-aed75ddf16fb/Untitled.png)

The secret is mounted as 3 separate files out of which token contains the access token in plain text format.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/835b2eb7-0eaa-44b8-9247-36a51b7f45e9/Untitled.png)

<aside>
💡 The default service account only has permissions to run basic K8s API queries.

</aside>

### Using Custom Service Accounts

Service account can be specified in the definition file. 

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: web-pod
spec:
	serviceAccountName: monitor-sa
	containers:
		- name: nginx
			image: nginx
```

<aside>
💡 The service account of a pod cannot be updated, the pod must be deleted and re-created with a different service account. However, the service account in a deployment can be updated as the deployment takes care of deleting and recreating the pods.

</aside>

### Don’t auto mount Default Service Account Token

This will prevent the `default` service account token (secret) from being auto-mounted to the pod.

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: web-pod
spec:
	automountServiceAccountToken: false
	containers:
		- name: nginx
			image: nginx
```

### Latest Changes

From v1.24 onwards, K8s has stopped auto-creating tokens for service accounts. Each of these tokens needed a separate secret object (hard to scale) and they were non-expiring (less secure). Now, `TokenRequestAPI` is used to provision tokens in a secure manner. These tokens are audience bound, time bound and object bound. Hence, they are more secure.

To generate a token for a service account, run the command `k create token <service-account-name>`. This token has a default validity of 1 hour which can be modified by passing some arguments when creating the token. The token is then mounted to the pod as a projected volume.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/bb1c9649-e459-4caa-b8ee-05326e330bca/Untitled.png)

## Access Control for Service Accounts

[Role Based Access Control (RBAC)](https://www.notion.so/Role-Based-Access-Control-RBAC-56a88e7951364ccea6f164cc5ad0fa74?pvs=21) can be used to limit access to service accounts.

```yaml
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups:
  - ''
  resources:
  - pods
  verbs:
  - get
  - watch
  - list

---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: ServiceAccount
  name: dashboard-sa
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

<aside>
🚫 No `apiGroup` should be specified when subject kind is `ServiceAccount`

</aside>

# Custom Resource Definition (CRD)

**We can define custom K8s resources (objects) using CRDs.** 

Let’s consider an example of creating a `FlightTicket` object using the definition file below.  Creating a resource using the definition file below will throw an error as `FlightTicket` object is not yet defined in K8s. We first need to create a CRD for it.

```yaml
apiVersion: flights.com/v1
kind: FlightTicket
metadata:
	name: my-flight-ticket
spec:
	from: Mumbai
	to: London
	count: 2
```

The CRD to create `FlightTicket` object in K8s:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
	name: flighttickets.flights.com
spec:
	scope: Namespaced
	group: flights.com
	names:
		kind: FlightTicket
		singular: flightticket
		plural: flighttickets
		shortNames:
			- ft
	versions:
		- name: v1
			served: true
			storage: true
			schema:
				openAPIV3Schema:
					type: object
					properties:
						spec:
							type: object
							properties:
								from:
									type: string
								to:
									type: string
								count:
									type: integer
									minimum: 1
									maximum: 10
```

- `scope: Namespaced` signifies that this resource will be scoped within the Namespace and not the whole cluster. Custom resources can also be cluster scoped.
- `group: flights.com` refers to the API group in which the custom resource will be created
- `names` configures the kind of the resource, its singular, plural and short names.
- `versions` specifies the supported API versions for the resource. `served: true` signifies that the version is being served through the `kube-apiserver`. Only one of the versions can be the storage version and have `storage: true`.
- `schema` defines the the properties to expect in the `spec` section of the resource. For integer fields we can specify validations such as minimum, maximum etc.
- Note that schema is defined under the version.

# Custom [Controllers](https://www.notion.so/Controllers-292c3061762044b39fc3dcfa76221fc1?pvs=21)

Creating a customer resource in K8s such as `FlightTicket` doesn’t do much. It’s just a configuration saved in the `etcd` store. To actually book a flight ticket when a `FlightTicket` object is created, we need to write a custom controller which will continuously monitor the `FlightTicket` objects in the `etcd` store and make API calls to a flight booking service.

# DaemonSet

**DaemonSet automatically schedules a single replica of a pod on all the nodes in the cluster.** It can be though of as a way to run a daemon process on all the nodes in the cluster. 

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/d0cf7ec6-a9f4-415e-8414-bcb0fc099ed5/Untitled.png)

KubeProxy can be deployed as a DaemonSet in the cluster so that the `kube-proxy` process runs as a single pod on all of the nodes. Networking solutions, log collectors and monitoring agents are often deployed as DaemonSets on the cluster.

## Definition file

Exactly like [Deployment](https://www.notion.so/Deployment-aaa3756097d1452a9d42cc3e493a36c6?pvs=21) except for the `replicas` field. To generate a sample definition file, first generate one for a deployment and edit it.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-daemon
  labels:
    name: monitoring-agent
spec:
  selector:
    matchLabels:
      name: monitoring-agent
  template:
    metadata:
      labels:
        name: monitoring-agent
    spec:
      containers:
      - name: monitoring-agent
        image: monitoring-agent
```

## Commands

- Get DaemonSets - `k get daemonsets`
- Describe a DaemonSet - `k describe daemonset <daemonset-name>`

# Static Pods

[Kubelet](https://www.notion.so/Kubelet-4ba7a09077064494a8f74868b6e1eebf?pvs=21) can independently manage pods on worker nodes without relying on other K8s components. Kubelet can be configured to look for k8s manifest files in a directory on the node. It can then automatically create, update and manage pods on the node based on the manifests files present in the directory. These pods are called **static pods**. 

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/b8bfe925-b64a-4d42-89a5-76bf768e2c94/Untitled.png)

If any static pod crashes, Kubelet will attempt to restart it. To delete a static pod, delete its manifest file from the directory.

To view the static pods running on a worker node, run `sudo crictl ps` on that node. This is because we don’t have the `kubectl` utility as we don’t have the `kube-api` server available on the node.

<aside>
💡 **Only pods can be created in a static manner.** Other K8s objects like ReplicaSets and Deployments depend on additional k8s components.

</aside>

## Configure pod manifest path

To configure the pod manifest path in the `kubelet` service, use the below highlighted configuration in the `kubelet` service. This can be viewed for a running `kubelet` service by running `ps -aux | grep kubelet`. 

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/9a8b951c-3301-4693-b037-902a68370a77/Untitled.png)

Another option is to refer `staticPodPath` from the kubelet config file (`--config` option) in the `kubelet` service.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/67ab2dc2-7a52-4633-9cd7-fa7cc43826b5/Untitled.png)

## Static Pods in a Cluster

Even if the node is a part of the cluster, we can create static pods by configuring the manifest directory and adding pod definition files in it. When a static pod is created in a node which is a part of the cluster, a mirror (read-only) object is also created in the KubeAPI server. This way, the [KubeAPI Server](https://www.notion.so/KubeAPI-Server-5a11ac27599b409a8e432675780d11ee?pvs=21) is aware of the static pods created in the cluster.

**Static pods running on a node are handled exclusively by the Kubelet running on that node**. [Kube Scheduler](https://www.notion.so/Kube-Scheduler-2ad8b62f2911478190431df9c1464dc9?pvs=21) has no control over these pods. 

Static pods that are a part of the cluster can be viewed using the `k get pods` command. They have the node name appended to their name.

## Setting up the control plane using static pods

Since static pods don’t depend on the control plane, we can use them to deploy the components of the control plane as pods on a node. 

Let’s say we are setting up a multi-master cluster. Start by installing the `kubelet` service on all of the nodes. Then, place the K8s manifests of the remaining control plane components in the `staticPodPath` in every node. Kubelet will bring up all the pods and if any of them fails, it will be restarted by Kubelet automatically.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/e409ed6e-2e7b-433f-9259-de71c17a5f21/Untitled.png)

<aside>
💡 **KubeAdm** uses this approach to set up the control plane.

</aside>

# Operators

- Operators automate managing the entire lifecycle of **distributed stateful applications** deployed in the cluster. K8s cannot do this natively as it requires application specific knowledge to do so.
- **Operators are specific to the stateful application they manage.** This is because different stateful applications like MySQL, ElasticSearch, etc. have different ways of setting up a distributed DB.
- **Operators use CRDs to extend the K8s API.** So, we have operators for MySQL, Prometheus, Postgres, etc.
- Operators monitor the stateful resources like a controller and makes the necessary changes if required.
- Operator for a stateful application is built by experts in that application. The operator contains information like:
    - How to setup a multi-instance cluster?
    - How to run the cluster?
    - How to sync data between instances?
    - How to update the application instances?
- Operators can be looked up on [OperatorHub](https://operatorhub.io/). These are built by the community.

---

[Kubernetes Operator simply explained in 10 mins](https://www.youtube.com/watch?v=ha3LjlD6g7g)
