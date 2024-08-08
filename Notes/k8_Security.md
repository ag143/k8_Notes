# KubeConfig

- KubeConfig is an authentication config file. It configures what clusters the user has access to and in what capacity. It also stores the key and certificate for TLS handshake when making calls to the `kube-apiserver`. The URL for the `kube-apiserver` is also configured in the KubeConfig.
- `kubectl` looks for the kube config as a file named `config` at location `~/.kube`. If there are multiple config files at that location or the config file is not named `config`, we need to pass the config file in the `kubectl` command as `kubectl get pods --kubeconfig config.yaml`.
- The changes made to the KubeConfig file don’t need to be applied. They are used when the `kubectl` command is run.
- KubeConfig file has 3 sections:
    - **Clusters** - K8s clusters that the user has access to
    - **Users** - User accounts with which the user has access to the clusters. These user accounts may have different privilege on different clusters.
    - **Contexts** - Define which user account will be used to access which cluster.
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/5291d3fe-98d6-4a0e-8a42-6d8f6ff315bb/Untitled.png)
    
- KubeConfig defines what user accounts have access to what clusters. It is not a configuration to create user accounts or clusters. It just defines which user account will be used by the `kubectl` command to access which cluster. This way we don’t have to specify these parameters in the `kubectl` commands.

# KubeConfig file

The below KubeConfig file uses the user account `admin` to access the cluster `playground`. 

<aside>
💡 Every cluster has the CA certificate specified. This lets the `kubectl` utility verify the certificate of the `kube-apiserver` during the TLS handshake.

</aside>

```yaml
apiversion: v1
kind: Config

clusters:
	- name: playground
		cluster:
			certificate-authority: ca.crt
			server: https://playground:6443

contexts:
	- name: admin@playground
		context:
			cluster: playground
			user: admin
	
users:
	- name: admin
		user:
			client-certificate: admin.crt
			client-key: admin.key
```

If the KubeConfig contains multiple contexts, we need to add a `current-context` field to specify which context to use as default. Also, namespace can be specified in the context. This means switching to a context will switch the user to the specified namespace.

```yaml
apiversion: v1
kind: Config

current-context: developer@playground

clusters:
	- name: playground
		cluster:
			certificate-authority: ca.crt
			server: https://playground:6443

contexts:
	- name: admin@playground
		context:
			cluster: playground
			user: admin
			namespace: accounting
	- name: developer@playground
		context:
			cluster: playground
			user: developer
			namespace: finance
	
users:
	- name: admin
		user:
			client-certificate: admin.crt
			client-key: admin.key
	- name: developer
		user:
			client-certificate: developer.crt
			client-key: developer.key
```

Certificates can also be specified as **base64 encoded text** instead of the `.crt` file.

```yaml
apiversion: v1
kind: Config

clusters:
	- name: playground
		cluster:
			certificate-authority-data: <base64-encoded-certificate>
			server: https://playground:6443

contexts:
	- name: admin@playground
		context:
			cluster: playground
			user: admin
	
users:
	- name: admin
		user:
			client-certificate-data: <base64-encoded-certificate>
			client-key: admin.key
```

# Commands

- View KubeConfig file - `k config view`
- Update the current context in KubeConfig - `k config use-context <context-name>`
- Pass a KubeConfig file in a `kubectl` command - `kubectl get pods --kubeconfig config.yaml`

<aside>
💡 Explore other `k config` commands

</aside>


# Kube Rest API

- All resources within K8s are grouped into different API groups. To find which API group a k8s resource belongs to run `k explain <resource>`.
- The `kube-apiserver` is available at `https://kube-master:6443`

# API Groups

- `/version` - to view the cluster version
- `/metrics` - to get cluster metrics
- `/healthz` - to monitor the cluster health
- `/logs` - to integrate with 3rd party logging applications
- `/api` - used to interact with the cluster (**core group**)
- `/apis` - used to interact with the cluster (**named group**)

## Core API group (`/api`)

All the core functionalities exist in this API group. All the resources (functionalities) are scattered in this group.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/74a4aa63-96fc-41d3-a341-cd8105786475/Untitled.png)

## Named API group (`/apis`)

The named API group is organized into subgroups (resources) based on the category. The newer features in k8s and going forward all the incoming features will be made available in this group. Several actions (verbs) can be performed on the resources.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/c08bd6c6-989e-4af5-98e5-6b9dac81f766/Untitled.png)

# Authenticating to Kube-ApiServer

Most of the API endpoint will require you to authenticate to the `kube-apiserver`. This means passing the login credentials in the curl command. Alternatively, you can setup a proxy client to by running the command `kubectl proxy` which will automatically proxy your API requests and add the credentials from the [KubeConfig](https://www.notion.so/KubeConfig-c99bacd10778413fbcb4f580dd0b9dbe?pvs=21)  file on your local system. So, you no longer need to pass the authentication details in every curl command.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/f458ab4f-eac0-4786-84be-26b66c55bdc2/Untitled.png)

### Examples

- Get version - `curl https://kube-master:6443/version`
- List pods - `curl https://kube-master:6443/api/v1/pods`

# API Versions

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/f2f5dd32-5e1c-4034-b57c-b2cc761bdce6/Untitled.png)

An API group can have multiple versions supported at the same time. Either of these API versions can be used to create a resource. But, when you query the resource it checks for it in the **Preferred API version**. Also, when a resource is created in an API version other than the **Storage API version**, it is converted to the storage API version before storing in the `etcd` database. Usually, the stable API version is the Preferred and Storage API version.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/e75988f7-e91c-4c7d-a5e3-b52912205d1e/Untitled.png)

Since alpha versions are not enabled by default, we can enable them by editing the `kube-apiserver`'s config.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7cb720d3-2dce-49b4-ac23-8a077382d8c8/Untitled.png)

### C**heck the preferred version for an API**

Steps:

- Setup `kubectl proxy` on port `8001` - `k proxy 8001 &`
- Send a GET request to the API - `curl localhost:8001/apis/authorization.k8s.io`

The output will be like this:

```yaml
{
  "kind": "APIGroup",
  "apiVersion": "v1",
  "name": "authorization.k8s.io",
  "versions": [
    {
      "groupVersion": "authorization.k8s.io/v1",
      "version": "v1"
    }
  ],
  "preferredVersion": {
    "groupVersion": "authorization.k8s.io/v1",
    "version": "v1"
  }
}
```

# API Deprecation Policy Rules

### **Rule 1**

> API elements may only be removed by incrementing the version of the API group.
> 

In the example below, the resource `webinar` can only be removed by incrementing the version from `v1alpha1` to `v1alpha2`. But both the versions will be supported because otherwise, all the usage of `v1alpha1` will have to be incremented. But, the preferred/supported version can be `v1alpha2`.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/cf527467-6f74-4710-908a-26891d1a2383/Untitled.png)

### Rule 2

> API objects must be able to round-trip between API versions in a given release without information loss, with the exception of whole REST resources that do not exist in some versions.
> 

In the example below, `v1alpha1` does not have `duration` field but `v1alpha2` does. So we need to add the field `duration` to `v1alpha1` so that when downgrading from `v1alpha2` to `v1alpha1`, both the `v1alpha1` versions are the same.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/bd7506ae-34b1-4f41-a656-b7df61572fb9/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/17c40a1c-524d-4aa8-a478-f540ba5312f1/Untitled.png)

### Rule 3

> An API version in a given track may not be deprecated until a new API version at least as stable is released.
> 

If `v2alpha1` is released, `v1` will not be deprecated until `v2` goes through its complete lifecycle and becomes a stable release.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ee6e5ba7-4570-4d1b-912f-9ec71007dc1e/Untitled.png)

### Rule 4a

> Other than the most recent API versions in each track, older API versions must be supported after their announced deprecation for a duration of no less than:
**GA**: 12 months or 3 releases (whichever is longer)
**Beta**: 9 months or 3 releases (whichever is longer)
**Alpha**: 0 releases
> 

Deprecated (old) alpha versions need not be supported in the next releases.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/49c9a542-fe4b-40aa-a799-e24156a9d176/Untitled.png)

Deprecated beta versions only need to be supported till 3 releases (see `v1beta1`). Also according to Rule 4b, `v1` becomes the preferred version only after the next release and not in its first release.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/227a70c3-5166-4f9f-a21c-f5fb0a7b44e6/Untitled.png)

### Rule 4b

> The "preferred" API version and the "storage version" for a given group may not advance until after a release has been made that supports both the new version and the previous version
> 

When `v1beta2` is released, `v1beta1` is still the preferred version (between the lines) even though it is deprecated. Only in the next release will `v1beta2` become the preferred version.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/70f98e5a-b844-43e9-9837-71c07568bfcb/Untitled.png)

### Upgrading the version of a K8s manifest

`kubectl convert -f nginx-old.yaml --output-version apps/v1 > nginx-new.yaml`   - this requires the installation of `kubectl convert` tool from K8s.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/bbc31921-32e3-4f18-bfc4-ffc2802e066d/Untitled.png)

# Securing Nodes and Cluster

# Securing Nodes

- Password based authentication to the hosts should be disabled
- Root user access should be disabled
- Only SSH key based authentication should be allowed

# Securing the cluster

- `kube-apiserver` is the center of all operation within the k8s cluster. We interact with it to make changes to the cluster. So, the first line of defense is to control access to the `kube-apiserver`. This involves two considerations:
    - **Who can access the cluster (authentication)**
    - **What can they do (authorization)**
- All communications within the k8s cluster between the various processes of k8s is secured by TLS encryption.
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/4a6bbf7b-9560-4c18-b144-3242871a066c/Untitled.png)
    
- By default, every pod can access every other pod in the cluster. We can restrict access between them using [NetworkPolicies](https://www.notion.so/NetworkPolicies-96d8b1f5970542f5a7af579077c2f679?pvs=21).

# Authentication

## Intro

- Authentication defines who can access the K8s cluster.
- A K8s cluster is used by 4 types of users:
    - Admin - manage the cluster
    - Developers - develop and deploy applications on the cluster
    - End users - access the application deployed on the cluster
    - 3rd party applications - access the cluster for integrations
- The security for end users is managed by the application running on Pods. So, the security for them does not need to be managed at the cluster level. Admin and Developers access the cluster through **User Accounts** whereas the bots (3rd party applications) access the cluster through [Service Account](https://www.notion.so/Service-Account-b66435734fd042e6b080e1478f66a519?pvs=21).
- User access is managed by the `kube-apiserver`. It authenticates the request before processing it.
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7d66619d-bc5f-4ef2-9130-4e258dc97879/Untitled.png)
    
- K8s does not manage user accounts natively like it manages service accounts. It relies on external solutions such as:
    - File containing list of usernames and passwords
    - File containing list of usernames and tokens
    - TLS Certificates
    - 3rd party IDP such as LDAP

## Basic Authentication

When implementing basic authentication using a file containing usernames and passwords or token, we need to pass the `basic-auth-file` or `token-auth-file` to the `kube-apiserver` and restart it. 

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/60aa91bf-25c1-4141-9abe-5eeccf7547e3/Untitled.png)

If the `kube-apiserver` is running as a service, update the service config and restart it. On the other hand, if the `kube-apiserver` is deployed as a pod through KubeAdmin, update the pod definition file which will automatically recreate the new pod.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7b585645-f035-468b-9eeb-ab80d8f06d4c/Untitled.png)

The user can then authenticate to the `kube-apiserver` in the curl command as shown below.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/0401dd63-fca9-4fe2-a185-a0daa6bc8a61/Untitled.png)

In case of static token file, the authentication in the curl command happens as a bearer token.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/a4792740-6cb4-4a1e-bc5e-a7870f791aaf/Untitled.png)

We need to use volume mounting to store the password file in a location on the host and pass it to the `kube-apiserver` pod (in case of KubeAdmin setup)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
      - --authorization-mode=Node,RBAC
      <content-hidden>
    - --basic-auth-file=/tmp/users/user-details.csv
    image: k8s.gcr.io/kube-apiserver-amd64:v1.11.3
    name: kube-apiserver
    volumeMounts:
    - mountPath: /tmp/users
      name: usr-details
      readOnly: true
  volumes:
  - hostPath:
      path: /tmp/users
      type: DirectoryOrCreate
    name: usr-details
```

<aside>
⛔ Managing user identities using a plaintext file is not the recommended way.

</aside>

# Authorization

- Authorization defines what can someone do after they have been authenticated.
- The best practice is to provide minimum permissions to every user account or service account.

# Authorization Mechanisms in K8s

### Always Allow

Allow all the requests made to the KubeAPI Server.

### Always Deny

Deny all the requests made to the KubeAPI Server.

### Node **Authorizer**

Used to authorize Kubelet service to send information, about the pods running on the worker nodes, to the KubeAPI server. Kubelet should be part of `system:nodes` group and have name prefixed with `system:node` to be authorized by the node authorizer. Node authorizer performs access control within the cluster.

### **Attribute Based Access Control (ABAC)**

Authorize users by specifying the allowed permissions for every user or group. This is done by creating a policy file (JSON) for each user or group and passing it to the KubeAPIServer. 

Later, if we want to modify the permissions for a set of users, we need to edit the permissions for all those users and restart the KubeAPIServer. Therefore, ABAC is difficult to manage.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3abd2a13-164f-40cc-a6bc-1f735caf8c9e/Untitled.png)

### Role **Based Access Control (RBAC)**

Instead of defining permissions for each user or group as with ABAC, we define roles with the right set of permissions and associate users and groups to these roles accordingly. 

Later, if we want to modify the permissions of a role, we can do it once and it will reflect for all the users who are associated to that role.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3457f9ca-ac95-4c3b-8eca-affa553ab188/Untitled.png)

### Webhook

We can outsource authorization to a 3rd party solution (eg. **Open Policy Agent**) outside the K8s cluster using webhooks. K8s will make a request to the the external authorization server with the information about the user and their access requirements and let the authorization server decide whether or not the user should be allowed.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/350cbe55-5ac4-46d7-9653-eaa0b42d1b1b/Untitled.png)

# Setting Authorization Modes

- Authorization modes are set in the `kube-apiserver` config (pod or service).
- `AlwaysAllow` is the default authorization mode.
- We can use multiple authorization modes by passing them as comma separated values. The access is decided in the given order. Whenever an authorizer denies the request, it is forwarded to the next authorizer in the chain until an authorizer accepts the request.
    
    For example: If a user makes a request, it cannot be processed by the node authorizer, so it denies the request. The request is then forwarded to RBAC which processes the request and allows it. Since the request has been allowed, it will not be forwarded to Webhook.
    

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7a5505de-f4e0-43a5-bcf6-4217dc553d1c/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/2d73e23d-81dd-4cdb-a8af-0637e8aeaed6/Untitled.png)

# Commands

- To check the authorization modes in a cluster, run `k describe pod kube-apiserver-controlplane -n kube-system` and look for `--authorization-mode`.

# Role Based Access Control (RBAC)

## Creating a Role

`Role` is a K8s object that can be created using a definition file.

`name` signifies the name of the role

`apiGroups` refers to the [Kube Rest API](https://www.notion.so/Kube-Rest-API-2b70a5dee31c46e9a1588026f33974c1?pvs=21) group. For core `/api` group, we can leave this to `""`

Role object is bound to a namespace and control access within that namespace. The namespace can be specified in the `metadata` section. If not specified, it takes the `default` namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
	name: developer
rules:
	- apiGroups: [""]
		resources: ["pods"]
		verbs: ["list", "get", "create", "update", "delete"]
	- apiGroups: [""]
		resources: ["ConfigMap"]
		verbs: ["create"]
```

We can also restrict access to resources based on their names.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
	name: developer
rules:
	- apiGroups: [""]
		resources: ["pods"]
		verbs: ["list", "get"]
		resourceNames: ["frontend", "backend"]
```

## Linking a user to a Role

To link a user to a role, we need to create a `RoleBinding` object. 

`subjects` refer to the users who will be bound to the role.

`RoleBinding` object is bound to a namespace and can be used to bind users to roles within that namespace. The namespace can be specified in the `metadata` section. If not specified, it takes the `default` namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
	name: dev-user-developer-role-binding
subjects:
	- kind: User
		name: dev-user
		apiGroup: rbac.authorization.k8s.io
roleRef:
	kind: Role
	name: developer
	apiGroup: rbac.authorization.k8s.io
```

# Commands

- List roles - `k get roles`
- List role bindings - `k get rolebindings`
- Check if you have access to perform an operation in K8s:
    - Create deployments - `k auth can-i create deployment`
    - Delete nodes in dev namespace - `k auth can-i delete nodes -n dev`
- Check if another user has access to perform an operation in K8s:
    - Create deployments - `k auth can-i create deployment --as dev-user`
    - Delete nodes in dev namespace - `k auth can-i delete node -n dev --as dev-user`
 

Cluster-wide RBAC
Roles are used to control access to namespace scoped K8s resources. ClusterRoles are used to control access to cluster-scoped resources. Example:
Providing access to the cluster admin to create or delete nodes in a cluster. 
Providing access to the storage admin to create or delete PVs
Creating a ClusterRole and binding it to a User
The definition file is very similar to that of Role except the kind and the resources.
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
	name: cluster-admin
rules:
	- apiGroups: [""]
		resources: ["nodes"]
		verbs: ["list", "get", "create", "delete"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
	name: cluster-admin-role-binding
subjects:
	- kind: User
		name: cluster-admin-user
		apiGroup: rbac.authorization.k8s.io
roleRef:
	kind: ClusterRole
	name: cluster-admin
	apiGroup: rbac.authorization.k8s.io
​
ClusterRole for Namespace-scoped Resources
ClusterRoles and ClusterRoleBindings can also be used to allow users to access namespace scoped resources. This way, the user can access that resource across the cluster in any namespace. 
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
	name: developer
rules:
	- apiGroups: [""]
		resources: ["pods"]
		verbs: ["list", "get", "create", "delete"]
​
Example ClusterRole and ClusterRoleBinding for Storage Admin
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: storage-admin
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes", "storageclasses"]
    verbs: ["list", "get", "create", "delete"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: arkalim-storage-admin
subjects:
  - kind: User
    name: arkalim
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: storage-admin
  apiGroup: rbac.authorization.k8s.io


# How Auth in K8s works

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/8fd8ed45-0152-44c3-bd71-ab11de5b9cea/Untitled.png)

When we use the `kubectl` command, it internally sends a request to Kube ApiServer which validates the request and persists the change in the `etcd` store for the controller to get invoked and take the right action.

The Kube ApiServer uses certificates configured in the [KubeConfig](https://www.notion.so/KubeConfig-c99bacd10778413fbcb4f580dd0b9dbe?pvs=21) to authenticate the user. Once the user’s identity has been verified, [Role Based Access Control (RBAC)](https://www.notion.so/Role-Based-Access-Control-RBAC-56a88e7951364ccea6f164cc5ad0fa74?pvs=21) is used to determine whether or not the user has access to perform the requested action. Finally, the request goes to [Admission Controllers](https://www.notion.so/Admission-Controllers-40e263e59104496abe9751084cc6dbf0?pvs=21)  for validation.

# Admission Controllers

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/8fd8ed45-0152-44c3-bd71-ab11de5b9cea/Untitled.png)

- **Admission Controllers validate or modify the incoming request before executing them.** Many admission controllers are pre-built in the k8s cluster and are enabled by default.
- Example usage:
    - Validating the request (Validating Admission Controllers)
    - Modifying the request (Mutating Admission Controllers)
    - Performing actions in the backend
- Mutating Admission Controllers are invoked before Validating Admission Controllers so that any change made by the Mutating Admission Controllers are also validated at the end.

### Validating Admission Controller

`NamespaceExists` admission controller rejects a request to create a resource in a namespace that doesn’t exist. This way, it validates the request.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/1ebfb385-3c42-474b-b4a0-9b554c6ec230/Untitled.png)

### Mutating Admission Controller

`NamespaceAutoProvision` is another admission controller which is not enabled by default. It creates the namespace automatically if a request is made to create a resource in that namespace.

`DefaultStorageClass` admission controller observes the creation of PVC objects that do not request any specific storage class and automatically adds a default storage class to them. This way it modifies the request.

**Note:** `NamespaceExists` and `NamespaceAutoProvision` admission controllers have now been deprecated and replaced with `NamespaceLifecycle` admission controller. It makes sure that requests to a non-existent namespace is rejected and that the default namespaces such as `default`, `kube-system` and `kube-public` cannot be deleted.

## Commands

- **View enabled admission controllers:**
    
    `cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep enable-admission-plugins`
    
    or
    
    `kube-apiserver -h | grep enable-admission-plugins` 
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7d2f70af-cdac-436f-aa8f-702a91930b0f/Untitled.png)
    
- **Enable/Disable admission controllers:**
    
    Left side is the setup where Kube ApiServer is run as a service and on the right is in the case of a `kubeadm` setup where Kube ApiServer is run as a pod. Add the admission controllers as comma separated values. 
    
    To disable admission controllers, use `--disable-admission-plugins` flag in the same way.
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/1fad0580-3fa5-4d67-9fad-9ee52960c316/Untitled.png)


  # TLS in Kubernetes

## Intro

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/70d8cde4-f9d2-4d2f-8bf4-e0968456f1e1/Untitled.png)

- The communication between the different nodes in the cluster, between the different components of k8s, between a user and the cluster, etc. must be encrypted using TLS.
- There must be at least one CA in the cluster for signing TLS certificates.
- The CA private key must be stored on a secure server (CA server). Anyone who has access to it can create as many authenticated users as they want. When setting up the cluster using KubeAdmin, it is stored on the master node. So, in this case, the master node is also the CA server.

## TLS Certificates

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/f4be1769-767c-4ba7-bec5-4f7140cc12cb/Untitled.png)

[KubeAPI Server](https://www.notion.so/KubeAPI-Server-5a11ac27599b409a8e432675780d11ee?pvs=21), [ETCD](https://www.notion.so/ETCD-305488fa760a458b975adecab21022d6?pvs=21), and [Kubelet](https://www.notion.so/Kubelet-4ba7a09077064494a8f74868b6e1eebf?pvs=21) act like servers that are contacted by clients. So, they generate server certificates. [Kube Scheduler](https://www.notion.so/Kube-Scheduler-2ad8b62f2911478190431df9c1464dc9?pvs=21), [Kube Controller Manager](https://www.notion.so/Kube-Controller-Manager-987e77b4189748ba9aebffee21b2d7e5?pvs=21) and [Kube Proxy](https://www.notion.so/Kube-Proxy-f90a9d6e9d6342b4a70054815c546817?pvs=21), contact the KubeAPI server. So, they generate client certificates. 

The KubeAPI server also contacts ETCD and Kubelet for which it can either use its server certificates or generate a client certificate for itself. Similarly, the Kubelet service also contacts the KubeAPI server for which it can generate its own client certificate or use its server certificate.

If a user has to access the cluster via the KubeAPI server, they will need to generate their own client-side certificates.

<aside>
💡 Whenever a server and client communicate, they both need a copy of the CA root certificate to verify each other’s certificate as every other certificate is signed by the CA.

</aside>

## Generating TLS Certificates

When setting up the cluster from scratch, you need to generate and configure each certificate manually. When setting up the cluster using **KubeAdmin**, it automatically generates and configures the certificates. All the certificates are present at `/etc/kubernetes/pki`.

### CA Certificate

- Generate CA’s private key - `openssl genrsa -out ca.key 2048`
- Generate a CSR (certificate without the signature) using the private key - `openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr` where `/CN` is the common name field (which component in k8s we are creating certificates for). The command will output a `ca.csr` file.
- Sign the CA’s CSR using its own private key generated in the first step (self-signing) - `openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt`. The command will output the self-signed CA certificate `ca.crt`.

Now that we have the CA private key and the `ca.crt` certificate. They both can be used together to sign other CSRs in the cluster.

### Client Certificate for Admin User

- Generate admin user’s private key - `openssl genrsa -out admin.key 2048`
- Generate a CSR for the admin user - `openssl req -new -key admin.key -subj "/CN=kube-admin/O=system:masters" -out admin.csr`. `/O=system:masters` adds the user to the `system:masters` group to provide them with admin privilege.
- Sign the admin’s CSR using the CA’s private key and certificate - `openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -out admin.crt`

### Client Certificate for K8s Components

Generating client TLS certificates for K8s components follows the same procedure. However, the common name `/CN` field must be prefixed with `system:`. The `/CN` field should be as follows for the following:

- Kube Scheduler - `system:kube-scheduler`
- Kube Controller Manager - `system:kube-controller-manager`
- Kube Proxy - `system:kube-proxy`
- Kubelet - `system:node:<node-name>`
    
    The Kubelet service needs to act as a client to connect with the API server. For that, it needs to generate separate certificates for each node (`kubelet` service runs on every node). For this the certificate needs to be a part of `system:nodes` group.
    
    Eg. to create a CSR for node-1 - `openssl req -new -key node-1.key -subj "/CN=system:node:node-1/O=system:nodes" -out node-1.csr`
    

### Server Certificate for K8s Components

Generating server TLS certificates for k8s components follows the same procedure. Since the servers run continuously, we need to configure the key and certificate along with the CA certificate before starting the server.

The `/CN` field should be as follows for the following:

- ETCD - `etcd-server`
    
    The key, certificate and the CA certificate need to be configured while starting the ETCD server.
    
    Peer certificates need to be configured when ETCD server has multiple instances for high availability.
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/c26c2ac7-e6ef-4af4-84c2-6889d3c35003/Untitled.png)
    
- KubeAPI Server - `kube-apiserver`
    
    The KubeAPI server is referred to by many names. All these names along with the pod’s IP running it must be specified under the `alt_names` section of the `openssl.cnf` file, when generating the CSR.
    
    `openssl req -new -key apiserver. key -subj "/CN=kube-apiserver" -out apiserver.csr -config openssl.cnf`
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/99586d1b-a0be-412b-9687-c56f48401cd4/Untitled.png)
    
    The key, certificate and the CA certificate needs to be configured while starting the KubeAPI server. We also need to specify the key, certificate and the CA certificate when the KubeAPI server acts as a client to connect to the ETCD server and Kubelet service.
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/c4a0992c-92bb-4780-8c4a-f1ea46614f29/Untitled.png)
    
- Kubelet - `<node-name>`
    
    The kubelet service runs on every node. So, separate key-certificate pairs must be created for every node. Since the Kubelet is an HTTP server, the key and certificate must be configured before starting it.
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/0f4d40f3-627b-4e2f-967b-d7095e60ce88/Untitled.png)
    

## Using Client Certificates

When making an API request to the KubeAPI server, the HTTP client (`curl` in this case) needs the admin key and certificate to authenticate the request. It also needs the CA certificate to validate the server’s certificate.

```bash
curl https://kube-apiserver:6443/api/v1/pods \
--key admin.key \
--cert admin.crt \
--cacert ca.crt
```

Otherwise, the certificates and keys can be configured in [KubeConfig](https://www.notion.so/KubeConfig-c99bacd10778413fbcb4f580dd0b9dbe?pvs=21) from where `kubectl` can automatically pick it and attach it with every request.

```yaml
apiVersion: v1
clusters:
- cluster:
		certificate-authority: ca.crt
		server: https://kube-apiserver:6443
	name: kubernetes
kind: Config
users:
- name: kubernetes-admin
	user:
		client-certificate: admin.crt
		client-key: admin.key
```

## View certificate details

Run the `openssl x509 -in certificate.crt -text -noout` command to view the certificate details including the subject, validity, issuer and alternative DNS names and IP addresses.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/09c85acc-e0bb-4347-a5e2-cb770012ddfe/Untitled.png)

## Certificates API

K8s has a built in `certificates` API to easily sign and manage certificates. This is handled by the [Kube Controller Manager](https://www.notion.so/Kube-Controller-Manager-987e77b4189748ba9aebffee21b2d7e5?pvs=21) using the CA certificate and key.

Let’s consider the case where a new user needs access to the cluster. She’ll first create a private key and then a CSR (`CN=<user-name>`) using `openssl` commands. She’ll then send the `.csr` file to the cluster admin. The admin will create a `CertificateSigningRequest` object using the base64 encoded contents of the `.csr` file (generate using `cat filename.csr | base64 -w 0`) along with the info about the `groups` and `usages`.

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
	name: <user-name>
spec:
	signerName: kubernetes.io/kube-apiserver-client
	groups:
		- system:authenticated
	usages:
		- digital signature
		- key encipherment
		- server auth
	request: <base64-encoded-csr>
```

All the `CertificateSigningRequest` objects can be viewed by the cluster admins using `k get csr` command. Then admin can approve any CSR by running `k certificate approve <csr-name>`.
