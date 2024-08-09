# Tips

- Use `--dry-run=client` to create a pod template as the starting point. Example:
`k run webapp --image=my-webapp --dry-run=client -o yaml > webapp.yaml`
    
    This will create `webapp.yaml` with the following contents:
    
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
    	creationTimestamp: null
    	labels:
    		run: webapp
    	name: webapp
    spec:
    	containers:
    		- image: my-webapp
    			name: webapp
    			resources: {}
    	dnsPolicy: ClusterFirst
    	restartPolicy: Always
    status: {}
    ```
    
- Use the following tool to quickly switch between contexts and namespaces: https://github.com/ahmetb/kubectx
    - Run `brew install kubectx` to install both `kubectx` and `kubens`
    - `kubens` - switch to a namespace in a dropdown (requires `fzf` to be installed)
- We can install plugins from [Installing Addons | Kubernetes](https://kubernetes.io/docs/concepts/cluster-administration/addons/) to add functionalities that are not natively provided by K8s. These include networking solutions, service discovery etc.


# How declarative commands work?

The declarative `kubectl apply` command checks the **local config file** (stored on the local system), the **live config** (stored in K8s memory) and the **last applied config** (stored in the live config as an annotation in JSON format), to decide what changes are to be made to take the system to the desired state. 

![API Version](Images/k8_intro/k8_declarative_commands1.png)

Last applied config is required to find out if something has been removed in the local config file.
![API Version](Images/k8_intro/k8_declarative_commands2.png)
# CKA

- Kubectl Reference - [Kubectl Reference Docs (kubernetes.io)](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands)

[Wrong questions](https://www.notion.so/Wrong-questions-a479bc3a193b4d7c95a6c22cc53f643b?pvs=21)

- Pod to print something at regular intervals
    
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      creationTimestamp: null
      labels:
        run: looper-cka16-arch
      name: looper-cka16-arch
    spec:
      containers:
      - image: busybox
        name: looper-cka16-arch
        command: 
          - "sh"
          - "-c"
          - "while true; do echo hello; sleep 10; done"
    ```
    
- PVC cannot be edited, it must be deleted and recreated
- Create ClusterRoleBinding for a ServiceAccount
    
    ```bash
    kubectl create clusterrolebinding <clusterrolebinding-name> --clusterrole=<clusterrole-name> --serviceaccount=<namespace>:<serviceaccount-name>
    ```
    
- SC, PV and PVC with full options
    
    ```yaml
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: orange-stc-cka07-str
    provisioner: kubernetes.io/no-provisioner
    volumeBindingMode: WaitForFirstConsumer
    ---
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: orange-pv-cka07-str
    spec:
      capacity:
        storage: 150Mi
      volumeMode: Filesystem
      accessModes:
        - ReadWriteOnce
      persistentVolumeReclaimPolicy: Retain
      storageClassName: orange-stc-cka07-str
      local:
        path: /opt/orange-data-cka07-str
      nodeAffinity:
        required:
          nodeSelectorTerms:
          - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
              - cluster1-controlplane
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: orange-pvc-cka07-str
    spec:
      accessModes:
        - ReadWriteOnce
      volumeMode: Filesystem
      resources:
        requests:
          storage: 128Mi
      storageClassName: orange-stc-cka07-str
      volumeName: orange-pv-cka07-str
    ```
    
- Using `jsonpath` to get the value of a key in YAML
    
    ```bash
    kubectl --context cluster1 get pod <podname> -o jsonpath='{.metadata.labels.stack}'
    ```
    
- Take ETCD backup using `etcdctl`
    
    ```bash
    ETCDCTL_API=3 etcdctl --endpoints=https://[127.0.0.1]:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key snapshot save /opt/cluster1_backup.db
    ```
    
- DNS name of a pod - `<pod-ip-separated-by-hyphens>.<namespace>.pod`
- Check if a user has access to perform an operation (`User` is not namespace bound)
    
    ```yaml
    k auth can-i get deployments --as <user-name>
    ```
    
- Check if a service account has access to perform an operation
    
    ```yaml
    k auth can-i get deployments --as system:serviceaccount:<namespace>:<serviceaccount-name>
    ```
    
- If a PVC is not getting bounded to a PV, check if the PV has `claimRef` referring to a specific version of the PVC.
- Display the name and IP address of all the pods in a namespace
    
    ```yaml
    kubectl get pods -n <namespace> -o=custom-columns='POD_NAME:metadata.name,IP_ADDR:status.podIP' --sort-by=status.podIP
    ```
    
- Ingress resource with SSL redirect turned off
    
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: nginx-ingress-cka04-svcn
      annotations:
        nginx.ingress.kubernetes.io/ssl-redirect: "false"
    spec:
      rules:
      - http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-service-cka04-svcn
                port:
                  number: 80
    ```
    
- When mounting a config file as a configMap type volume, mount the volume at the directory containing the config file (not the full path of the config file).
- Config files for `kubelet` are present under `/var/lib/kubelet` on every node.
- Endpoint object to send traffic to an external web server
    
    ```yaml
    apiVersion: v1
    kind: Endpoints
    metadata:
      # the name here should match the name of the Service
      name: external-webserver-cka03-svcn
    subsets:
      - addresses:
          - ip: <node-ip>
        ports:
          - port: 9999
    ```
    
- `apiGroups: ""` (core) for namespace as resource in a cluster role.
- If `kube-apiserver` is not reachable or not running, grep into the kubelet logs using `journalctl -u kubelet | grep` command and look closely for error logs.
- `spec.volumeName: <pv-name>` to make a PVC request storage from a specific PV
- Check the details like image name of a deployment revision - `k rollout history deployment <deployment-name> --revision <revision-number>`
- If pods are not able to resolve a hostname like `example.com`, it’s likely because CoreDNS containers in `kube-system` namespace is down. In this case if you see no CoreDNS containers, it’s possible that the `coredns` deployment in `kube-system` namespace is scaled down to `0`. The solution in this case would be to scale the `coredns` deployment to `2`.
- Allowing volume readonly permission to the sidecar container
    
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: olive-app-cka10-str
    spec:
      replicas: 1
      template:
        metadata:
          labels:
            app: olive-app-cka10-str
        spec:
          containers:
          - name: python
            ...
            volumeMounts:
            - name: python-data
              mountPath: /usr/share/
          - name: busybox
            ...
            volumeMounts:
              - name: python-data
                mountPath: "/usr/src"
                **readOnly: true**
    ```
    
- Logs for stopped containers like `kube-apiserver` can be found at `/var/log/pods`. This is specially useful when `kube-apiserver` is down and we need it’s container logs.
- If several pods in the `kube-system` namespace are crashing repeatedly after some time, a possible cause can be `kube-apiserver` pod crashing due to wrong health check configuration.
- Having `spec.ingressClassName` in `Ingress` is important. If it’s not specified in the question, find the ingress class and specify it.
- To match all the pods, set `spec.podSelector: {}`
- `NetworkPolicy` to only allow pods in `space1` to reach pods in `space2`
    
    By default, all the TCP ports are open in a network policy. We need to explicitly allow port 53 on UDP so that pods in `space1` can resolve DNS queries.
    
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: np
      namespace: space1
    spec:
      podSelector: {}
      policyTypes:
        - Egress
      egress:
        - to:
            - namespaceSelector:
                matchLabels:
                  kubernetes.io/metadata.name: space2
        - ports:
            - protocol: UDP
              port: 53
    ---
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: np
      namespace: space2
    spec:
      podSelector: {}
      policyTypes:
        - Ingress
      ingress:
        - from:
            - namespaceSelector:
                matchLabels:
                  kubernetes.io/metadata.name: space1
    ```
    
- RBAC Rules
    
    A `*ClusterRole`/`Role`* defines a set of permissions and where it is available, in the whole cluster or just a single namespace.
    
    A `*ClusterRoleBinding`/`RoleBinding`* connects a set of permissions with an account and defines where it is applied, in the whole cluster or just a single namespace.
    
    - `*Role*` + `*RoleBinding*` (available in single namespace, applied in single namespace)
    - `*ClusterRole*` + `*ClusterRoleBinding*` (available cluster-wide, applied cluster-wide)
    - `*ClusterRole*` + `*RoleBinding*` (available cluster-wide, applied in single namespace when creating `RoleBinding` object)
    - `*Role*` + `*ClusterRoleBinding*` (**NOT POSSIBLE**)
- Pod scheduling priority is present in `spec`. Higher the number, higher the priority.
