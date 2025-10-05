# CKA-2025

## 1. Install cri-dockerd and Set System Parameters

Prepare a Linux system for Kubernetes. Docker is already installed, but you need to configure it for `kubeadm`.

**Tasks:**

- Install the `.deb` package
- Install the Debian package `-/cri-dockerd_O.3.9.3-O.ubuntu-jammy_amd64.deb`
- Debian packages are installed using `dpkg`.
- Enable and start the `cri-docker` service
- Set the following system parameters:
  - `net.bridge.bridge-nf-call-iptables=1`
  - `net.ipv6.conf.all.forwarding=1`
  - `net.ipv4.ip_forward=1`
  - `net.netfilter.nf_conntrack_max=131072`

**Solution:**

```bash
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.18/cri-dockerd_0.3.18.3-0.ubuntu-jammy_amd64.deb

sudo dpkg -i cri-dockerd_0.3.1.3-0.ubuntu-jammy_amd64.deb
```

Enable and start service
```bash
sudo systemctl enable cri-docker.service
sudo systemctl start cri-docker.service
```

Modify the parameters
```bash
vim /etc/sysctl.d/99-zcka.conf

OR

vim /etc/sysctl.conf
```

```bash
# Inside /etc/sysctl.d/cka.conf
net.bridge.bridge-nf-call-iptables=1
net.ipv6.conf.all.forwarding=1
net.ipv4.ip_forward=1
net.netfilter.nf_conntrack_max=131072

# Apply the changes
sudo sysctl --system
```

Verify the changes with `sudo sysctl -p`.

Troubleshoot:
- If the parameter value keeps changing after apply,
- Try to load your file at the last, rename file with a high number: `/etc/sysctl.d/99-zcka.conf`

OR
To search for the file inside `/etc/sysctl.d/` that contains the string `net.ipv4.ip_forward`, run:
```
grep -rl "net.ipv4.ip_forward" /etc/sysctl.d/ /usr/lib/sysctl.d/ /etc/sysctl.conf
```

## 2. Verify cert-manager CRD

- Create a list of all cert-manager Custom Resource Definitions (CRDs) and save it to `~/resources.yaml`.
- Make sure kubectl's default output format and use kubectl to list CRD's.
- Do not set an output format.
- Failure to do so will result in a reduced score.
- Using kubectl, extract the documentation for the subject specification field of the Certificate Custom Resource and save it to `~/subject.yaml`.
- You may use any output format that kubectl supports.

**Tasks:**

- List cert-manager CRDs and save to `~/resources.yaml`
- Extract documentation for `certificates.spec.subject` to `~/subject.yaml`

**Solution:**

```bash
k get crds | grep cert-manager > ~/resouce.yaml

k explain certificates.spec.subject > ~/subject.yaml
```
OR
```bash
kubectl get crd | grep cert-manager | awk ‘{print $1}’ | xargs -I{} kubectl get crd {} -o yaml >> ~/resources.yaml

kubectl get crd certificates.cert-manager.io -o jsonpath='{.spec.versions[*].schema.openAPIV3Schema.properties.spec.properties.subject}' > subject.yaml
```

## 3. TLS Update in ConfigMap

An NGINX Deploy named `nginx-static` is running in the `nginx-static` NS. It is configured using a CfgMap named `nginx-config`. 
- Update the nginx-config CfgMap to allow only TLSv1.3 connections. Re-create, restart, or scale resources as necessary.
- By using 1 command to test the changes:
	- `curl --tls-max 1.2 https://web.k8s.local`
    - As TLSV1.2 should not be allowed anymore, the command should fail

**Tasks:**

- Update the ConfigMap to allow only TLSv1.3
- Restart resources as needed
- Confirm with `curl --tls-max 1.2 https://web.k8s.local` (should fail)

**Solution:**

```bash
kubectl get cm -n nginx-static

kubectl get deploy -n nginx-static

kubectl get cm nginx-config -n nginx-static -o yaml > configmap.yaml
```

vim `configmap.yaml`
- Delete the TLSv1.2 from the ConfigMap

```bash
# Edit the configmap.yaml to remove TLSv1.2

kubectl apply -f configmap.yaml

kubectl rollout restart deployment nginx-static -n nginx-static
```

Take a backup of deployment and re-deploy.
```bash
K get deploy nginx-static -o yaml > deploy.yaml

K delete deploy nginx-static

K create -f deploy.yaml
```

## 4. Create Ingress for Deployment

Create a new ingress resource named `echo` in `echo-sound` namespace.

With the following tasks:
- Expose the deployment with a service named `echo-service` on `http://example.org/echo` using `Service port 8080 type=NodePort`.
- The availability of Service `echo-service` can be checked using the following command which should return `200`:

**Tasks:**
- Create service named `echo-service` with `type=NodePort` on port 8080
- Create ingress resource named `echo` in namespace `echo-sound`

**Solution:**

```bash
kubectl expose deployment nginx-deploy --name=echo-service --type=NodePort --port=8080 --target-port=8080 -n echo-sound
```
`service/echo-service` should be created.



Copy `ingress.yaml` from documentation

Vim `ingress.yaml`
- Add `host: {Hostname}`
- Add service details, earlier created
`Port number : 8080`
`Path: /`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo
  namespace: echo-sound
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: example.org/echo
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: echo-service
            port:
              number: 8080
```
```bash
# Create ingress.yaml with host `example.org/echo` and path `/`
kubectl apply -f ingress.yaml
```

--- 

## 5. Gateway API Migration
You have an existing web application deployed in a Kubernetes cluster using an Ingress resource named `web`. You must migrate the existing Ingress configuration to the new Kubernetes Gateway API, maintaining the existing HTTPS access configuration.

**Tasks** :
- Create a Gateway resource named `web-gateway` with hostname `gateway.web.k8s.local` that maintains the existing TLS and listener configuration from the existing Ingress resource named `web`.
- Create an resource named `web-route` with hostname `gateway.web.k8s.local` that maintains the existing routing rules from the current Ingress resource named `web`.
  
Note:
	A GatewayClass named `nginx-class` is already installed in the cluster.

```bash
k get secret
k describe secret web-tls

k describe ingress web

k get svc
```
Get `gateway.yaml` from documentation
- Replace values from the info from `k describe ingress`
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: web-gateway
spec:
  gatewayClassName: nginx-class
  listeners:
  - name: https
    protocol: HTTPS
    port: 443
    hostname: "gateway.web.k8s.local"
	tls:
	  mode: Terminate
	  certificateRefs:
		- kind: Secret
		  name: web-tls
```
`k create -f gateway.yaml`

Copy HTTPRoute from documentation
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-route
spec:
  parentRefs:
  - name: web-gateway
  hostnames:
  - "gateway.web.k8s.local"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: web-service
      port: 80 					// Check from k get svc
```

-------------------------------------
The team from Project r500 wants to replace their Ingress (Networking.k8s.io) with a Gateway Api gateway.networking.k8s.io) solution. 

The old Ingress is available at `/opt/course/13/ingress.yaml`. Perform the following in Namespace `project-r500` and for the already existing Gateway:
- Create a new HTTPRoute named `traffic-director` which replicates the routes from the old Ingress.
- Extend the new HTTPRoute with path `/auto` which redirects to mobile if the User-Agent is exactly `mobile` and to desktop otherwise

**Tasks:**
- Replace Ingress with Gateway + HTTPRoute in `project-r500`

**Example Route Snippet:**

vim `httproute.yaml`

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: traffic-director
  namespace: project-r500
spec:
  parentRefs:
    - name: main   		# use the name of the existing Gateway
  hostnames:
    - "r500.gateway"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /desktop
      backendRefs:
        - name: web-desktop
          port: 80
    - matches:
        - path:
            type: PathPrefix
            value: /mobile
      backendRefs:
        - name: web-mobile
          port: 80
    - matches:
        - path:
            type: PathPrefix
            value: /auto
          headers:
          - type: Exact
            name: user-agent
            value: mobile
      backendRefs:
        - name: web-mobile
          port: 80
    - matches:
        - path:
            type: PathPrefix
            value: /auto
      backendRefs:
        - name: web-desktop
          port: 80
```

Our solution should result in this:

```bash
➜ candidate@cka7968:~$ curl -H "User-Agent: mobile" r500.gateway:30080/auto
Web Mobile App

➜ candidate@cka7968:~$ curl -H "User-Agent: something" r500.gateway:30080/auto
Web Desktop App

➜ candidate@cka7968:~$ curl r500.gateway:30080/auto
Web Desktop App
```

-----

## 6. Add Sidecar and Shared Volume

Update the existing deployment wordpress, adding a sidecar container named sidecar using the `busybox:stable` image to the existing pod.

The new sidecar container has to run the following command : `"/bin/sh -c "tail -f /var/log/wordpress.log"` use a volume mounted at `/var/log` to make the log file `wordpress.log` available to the co-located container.

       `OR`
	   
A legacy app needs to be integrated into the Kubernetes built-in logging architecture (i.e. kubectc logs). Adding a streaming co-located container is a good and common way to accomplish this requirement.

**Task**
- Update the existing Deployment synergy-deployment, adding a co-located container named sidecar using the `busybox:stable` image to the existing Pod.
- The new co-located container has. to run the following command: `/bin/sh -c "tail -f /var/log/synergy-deployment.log"`
- Use a Volume mounted at `/var/log` to make the log file `synergy-deployment.log` available to the co-located container.
- Do not modify the specification of the existing container other than adding the required.
Hint: Use a shared volume to expose the log file between the main application container and the sidecar.

**Tasks:**
- Update `synergy-deployment` with sidecar using `busybox:stable`.
- Share `/var/log/synergy-deployment.log` via `emptyDir` volume.

**Solution:**
1. Add a sidecar container,
2. EmptyDir Volume,
3. 2 volumeMounts to each container

- Mount `/var/log` in both main and sidecar containers

Take a Backup (Recommended)
- Before editing the deployment, take a backup of the current spec:

```bash
kubectl get deploy neokloud-deployment -o yaml > /root/deploy-backup.yaml

OR

cp source_file copy_file
```

- Edit the Deployment to Add Sidecar
- Edit the deployment using the following command:

```bash
kubectl edit deploy neokloud-deployment
```

**Changes to do**:

### 1. Add a sidecar container under `spec.template.spec.containers`:
```yaml
- name: sidecar
  image: busybox:stable
  command: ["/bin/sh", "-c", "tail -f /var/log/neokloud.log"]
  volumeMounts:
    - name: shared-logs
      mountPath: /var/log
```
### 2. Add this volumes block under `spec.template.spec`:
```yaml
volumes:
  - name: shared-logs
    emptyDir: {}
```

### 3. Add this volumeMounts block to the existing container (e.g., monitor):
```yaml
volumeMounts:
  - name: shared-logs
    mountPath: /var/log
```

Save and exit from the editor.

### 4: Verify Your Changes
Final vim deploy:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-deployment
  labels:
    app: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: busybox:stable
          command: ['sh', '-c', 'while true; do echo "logging" >> /opt/logs.txt; sleep 1; done']
          volumeMounts:
            - name: sidecar
              mountPath: /var/log
      initContainers:
        - name: logshipper
          image: busybox:stable
          restartPolicy: Always
          command: ["/bin/sh","-c","tail -f /var/log/wordpress.log"]
          volumeMounts:
            - name: sidecar
              mountPath: /var/log
      volumes:
        - name: sidecar
          emptyDir: {}
```

Check if the updated pod is running:
```bash
kubectl get pods
```

Get the exact pod name, then:

```bash
kubectl describe pod <pod-name>
kubectl logs -f <pod-name> -c sidecar
```

You should now see the sidecar container tailing `/var/log/neokloud.log`.

## 7. PVC + Mount to Deployment

A Persistent Volume already exists and is retained for reuse.

Create a PVC named `MariaDB` in the `mariadb` namespace as follows
- Access mode `ReadWriteOnce`
- Storage capacity `250Mi`

**Solution:**

**Tasks:**
- Create a PVC
- Mount it inside existing Deployment
- Edit the `maria-deployment` in the file located at `maria-deploy.yaml` to use the newly created PVC.

Verify that the deployment is running and is stable.

Copy a `pvc.yaml` from documentation
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mariadb
  namespace: mariadb
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 250Mi
```
`k edit deploy mariadb`

Add the PVC Claim name under `spec.template.spec.volumes.persistentVolumeClaim.claimName`

`k get pvc -n mariadb` should show the status as `Bound`

---

## 8. ArgoCD Install with Helm (No CRDs)

Step 1: Add Helm Repo
```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

kubectl create namespace argocd
```
Step 2: Render the Argo CD Manifest
```
helm template argo-cd argo/argo-cd \
  --version 8.0.17 \
  --namespace argocd \
  --set crds.install=false > argo-template.yaml
```
Step 3: Apply to Cluster
```bash
kubectl apply -f argo-template.yaml
```

```bash
helm install argo-cd argo/argo-cd \
  --version 8.0.17 \
  --namespace argocd \
  --set crds.install=false
```
Note: CRDs must be installed separately if not already present.

Step 4: Verify Argo CD Pods
```bash
kubectl get pods -n argocd
```

## 9. Divide Node Resources (With Init Containers)

### Solution 1:
```bash
K scale deploy wordpress --replicas=0

K describe node node01
```

Check cpu, memory in Allocatable section
- A = memory number / 1024
Check the Memory requests in describe node.
- B = {A} – (sum of memory request)
- C = {B} * 0.10 | bc. 
Expr {B} – {C}
Expr {B - C} / 3 will give the allocateable memory for each of the 3 pods

 (((Allocatable Mem / 1024) – (sum of memory request)) - (B * 0.10))/ 3

Do it similarly for CPU
The calculation values are the ‘requests’. Limits should be declared slightly higher

`K edit deploy wordpress`

```yaml
initContainers:
- name: init
  image: busybox
  resources:
    requests:
      cpu: "100m"
      memory: "128Mi"
    limits:
      cpu: "100m"
      memory: "128Mi"

containers:
- name: app
  resources:
    requests:
      cpu: "500m"
      memory: "256Mi"
    limits:
      cpu: "500m"
      memory: "256Mi"
```

```bash
k scale deploy wordpress --replicas=3
```

### Solution 2:

`k describe node | grep -A5 "Allocatable"`

Calculation Breakdown:
- Reserve Overhead:
1. Let's reserve about 15% for node/system overhead:
	- CPU overhead: {Actual CPU} * 0.15 = {A-Cpu} Reserve cores
	- Memory overhead: {Actual Memory} * 0.15 = {A-Mem} Reserve
2. Usable for Pods:
	- CPU:  {Actual CPU} - {A-Cpu} Reserve  = {B-CPU}
	- Memory:  {Actual Memory} - {A-Mem} Reserve = {B-Mem}
3. Divide by 3 Pods:
	- CPU per Pod:  {B-CPU} / 3 = {C-CPU} and round to off (conservative & stable)
	- Memory per Pod:  {B-Mem} / 3 = {C-Mem} round it off
4. Final Calculation:
    - CPU = ((Allocatable CPU - (Allocatable CPU x 0.15))/3)
    - Mem = ((Allocatable Mem - (Allocatable Mem x 0.15))/3)


```bash
k scale deployment wordpress -n namespace --replicas=0
```

```bash
k edit deployment wordpress
```

The calculation values are the `requests`. Limits should be declared slightly higher.

- Edit inside both the containers.
- Edit in `spec.template.spec.containers.resources.requests`


```bash
controlplane:~$ kubectl describe node | grep -A5 "Allocatable"
Capacity:
	cpu: 1
	ephemeral-storage: 19221248Ki
	hugepages-2Mi : 0
	memory: 2015360Ki
	pods: 110
```

---

## 10. Least Permissive NetworkPolicy

There are 2 deployments. Frontend in `frontend` namespace, Backend in `backend` namespace. Create a Network Policy to have interaction between frontend & backend deployments. 

The network policy has to be least permissive.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-from-backend
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          app: frontend
      podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080 		// check the service
```
---

## 11. Install CNI: Flannel vs Calico

### i) **Calico: Network policy**
Dont use `k apply -f`
```bash
curl -sL https://raw.githubusercontent.com/projectcalico/calico/v3.28.3/manifests/tigera-operator.yaml | kubectl create -f -

K create -f tigera-operator.yaml 
```

`k get pods` should show 1 running pod `tigera-operator-******`

```bash
kubectl cluster-info dump | grep -i cluster-cidr
>>>> --cluster-cidr=<COPY-THIS-CIDR>,
```

```bash
wget https://raw.githubusercontent.com/projectcalico/calico/v3.28.3/manifests/custom-resources.yaml
```

`vim custom-resources.yaml` and replace the `cidr:` with the above copied cidr value.

`k create -f custom-resources.yaml` Will take 4-5 minutes to create all the pods.

`k get pods -n calico-system` to check if all the pods are running.

### ii) **Flannel:**

```bash
curl -sL https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml | kubectl apply -f -
```

- Pods will be failing.
- Copy the IP Address from `net-conf.json> “Network”: “192.168.0.0/16”` or from the logs 
  ```bash
   k logs pod-flannel -n kube-flannel
  ```
  
- `k edit cm kube-flannel-cfg -n kube-flannel`
- `k delete pod kube-flannel-**** -n kube-flannel`

- The Daemonset will create new pods of flannel, and it'll work.

--- 
## 12. HPA with Autoscaling v2

Create a new HorizontalPodAutoscaler [HPA] named `apache-server` in the `autoscale` namespace.

**Tasks** :
- This HPA must target the existing deployment called `apache-deployment` in the namespace.
- Set the HPA to target for CPU usage per Pod.
- Configure the HPA to have a minimum of `1` pod and maximum of `4` pods.
- Have to set the downscale stabilization window to 30 seconds.

Copy the 3rd `hpa.yaml` from the documentation

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: apache-server
  namespace: autoscale
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: apache-deployment
  minReplicas: 1
  maxReplicas: 4
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 30
```
---

## 12. Troubleshooting

_(Details not provided)_

---

## 13. Expose Deployment via NodePort and Fix Port

There is a deployment named `nodeport-deployment` in the relative namespace.
Tasks:
- Configure the deployment so it can be exposed using port `80` and protocol `TCP` name `http`.
- Create a new service named `nodeport-service` exposing the container port `80` and TCP
- Configure the new Service to also expose the individual pods using Nodeport.

**Tasks:**
- Edit deployment to expose container port 80
- Create NodePort service using same label

**Solution:**

`K edit deploy`

Inside `spec.template.spec.containers`

```yaml
spec:
  containers
	ports :
	- name: http
	  containerPort: 80
	  protocol: TCP
```

```bash
k get deploy --show-labels #Use the labels while creating the service
```

Copy a Nodeport service file from doc. 
- Replace the labels with the above labels.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nodeport-service
  namespace: relative
spec:
  type: NodePort
  selector:
    app: nodeport-deployment //Labels from the deployment
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
    nodePort: 30080
```
---

## 14. PriorityClass

You're working in a Kubernetes cluster with an existing Deployment named `busybox-logger` running in a namespace called `priority`.
The cluster already has at least one user-defined Priority Class

Perform the following tasks:
1. Create a new Priority Class named `high-priority` for user workloads. The value of this Priority Class should be exactly one less than the highest existing user-defined Priority Class value.
2. Patch the existing Deployment `busybox-logger` in the `priority` namespace to use the newly created `high-priority` Priority Class.

**Solution:**

Get `PC.yaml` from documentation

Do `k get pc` to find the value of highest priority.

The PriorityClass which starts with a prefix of `system-` are not user-defined PriorityClass. Don't consider their value.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 100000
globalDefault: false
```

Patch the deploy with new `PriorityClass`
```bash
kubectl patch deploy -n priority busybox-logger -p '{"spec":{"template":{"spec":{"priorityClassName":"high-priority"}}}}'
deployment.apps/busybox-logger patched
```
OR

Edit the deployment:-

In Deployment:
`k edit deploy `

Add the below field in `spec.template.spec`
```
	spec:
	  priorityClassName: high-priority
```
---

## 15. Deployment with Anti-Affinity and Multiple Containers

Create a Deployment named `deploy-important` with 3 replicas. The Deployment and its Pods should have label `id=very-important`. 
- First container named `container1` with image `nginx:1-alpine`.
- Second container named `container2` with image `google/pause`.
There should only ever be one Pod of that Deployment running on one worker node, use `topologyKey: kubernetes.io/hostname` for this.

**Namespace:** `project-tiger`

**Solution:**

```yaml
spec:
  replicas: 3
  selector:
    matchLabels:
      id: very-important
  template:
    metadata:
      labels:
        id: very-important
    spec:
      containers:
      - name: container1
        image: nginx:1-alpine
      - name: container2
        image: google/pause
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: id
                operator: In
                values:
                - very-important
            topologyKey: "kubernetes.io/hostname"
```
## 15. Deployment on all Nodes - Anti-affinity 
 
Create a Deployment named `deploy-important` with 3 replicas

The Deployment and its Pods should have label `id=very-important`
- First container named `container1` with image `nginx:1-alpine`
- Second container named `container2` with image `google/pause`

There should only ever be one Pod of that Deployment running on one worker node, use `topologyKey: kubernetes.io/hostname` for this
ℹ️ Because there are two worker nodes and the Deployment has three replicas the result should be that the third Pod won't be scheduled. In a way this scenario simulates the behaviour of a DaemonSet, but using a Deployment with a fixed number of replicas

**Answer**:
There are two possible ways, one using `podAntiAffinity` and one using `topologySpreadConstraint`.

### PodAntiAffinity
The idea here is that we create a "Inter-pod anti-affinity" which allows us to say a Pod should only be scheduled on a node where another Pod of a specific label (here the same label) is not already running.

Let's begin by creating the Deployment template:

```yaml
k -n project-tiger create deployment --image=nginx:1-alpine deploy-important --dry-run=client -o yaml > 12.yaml

# cka2556:/home/candidate/12.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    id: very-important                  # change
  name: deploy-important
  namespace: project-tiger              # important
spec:
  replicas: 3                           # change
  selector:
    matchLabels:
      id: very-important                # change
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        id: very-important              # change
    spec:
      containers:
      - image: nginx:1-alpine
        name: container1                # change
        resources: {}
      - image: google/pause             # add
        name: container2                # add
      affinity:                                             # add
        podAntiAffinity:                                    # add
          requiredDuringSchedulingIgnoredDuringExecution:   # add
          - labelSelector:                                  # add
              matchExpressions:                             # add
              - key: id                                     # add
                operator: In                                # add
                values:                                     # add
                - very-important                            # add
            topologyKey: kubernetes.io/hostname             # add
status: {}
Specify a topologyKey, which is a pre-populated Kubernetes label, you can find this by describing a node.
```
 

### TopologySpreadConstraints
We can achieve the same with `topologySpreadConstraints`. Best to try out and play with both.

```yaml
# cka2556:/home/candidate/12.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    id: very-important                  # change
  name: deploy-important
  namespace: project-tiger              # important
spec:
  replicas: 3                           # change
  selector:
    matchLabels:
      id: very-important                # change
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        id: very-important              # change
    spec:
      containers:
      - image: nginx:1-alpine
        name: container1                # change
        resources: {}
      - image: google/pause             # add
        name: container2                # add
      topologySpreadConstraints:                 # add
      - maxSkew: 1                               # add
        topologyKey: kubernetes.io/hostname      # add
        whenUnsatisfiable: DoNotSchedule         # add
        labelSelector:                           # add
          matchLabels:                           # add
            id: very-important                   # add
status: {}
```

Apply and Run

```bash
$ k -f 12.yaml create
deployment.apps/deploy-important created
Then we check the Deployment status where it shows 2/3 ready count:


➜ candidate@cka2556:~$ k get pod -o wide -l id=very-important -n project-tiger
NAME                                READY   STATUS   ...   IP           NODE
deploy-important-78f98b75f9-5s6js   0/2     Pending  ...   <none>       <none>
deploy-important-78f98b75f9-657hx   2/2     Running  ...   10.44.0.33   cka2556-node1
deploy-important-78f98b75f9-9bz8q   2/2     Running  ...   10.36.0.20   cka2556-node2
If we kubectl describe the not scheduled Pod it will show us the reason didn't match pod anti-affinity rules:

Warning  FailedScheduling  119s (x2 over 2m1s)  default-scheduler  0/3 nodes are available: 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }, 2 node(s) didn't match pod anti-affinity rules. preemption: 0/3 nodes are available: 1 Preemption is not helpful for scheduling, 2 No preemption victims found for incoming pod.
Or our topologySpreadConstraints reason didn't match pod topology spread constraints:

Warning  FailedScheduling  20s (x2 over 22s)  default-scheduler  0/3 nodes are available: 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }, 2 node(s) didn't match pod topology spread constraints. preemption: 0/3 nodes are available: 1 Preemption is not helpful for scheduling, 2 No preemption victims found for incoming pod.
 ```

## 16. ETCD Info Extraction

The cluster admin asked you to find out the following formation about etcd running on `cka9412` :
- Server private key location
- Server certificate expiration date
- Is client certificate authentication enabled
  
Write these information into `/opt/course/pl/etcd-info.txt`

**Tasks:**
- Server private key location
- Certificate expiration
- Client auth enabled?

**Solution:**

```bash
sudo - i

Look into /etc/kubernetes/manifests/etcd.yaml for “--key-file” “--cert-file” 
--client-cert-auth=true

cd /etc/kubernetes/manifests

# Certificate check:
openssl x509 -noout -text -in /etc/kubernetes/pki/etcd/server.crt | grep Validity -A2

# Look for client-cert-auth: true in etcd manifest
```
Copy the “Validity – Not after” field into Solution file.
- Client-cert `YES` TRUE

```bash
# /opt/course/pl/etcd-info.txt
Server private key location: /etc/kubernetes/pki/etcd/server.key
Server certificate expiration date: Feb 20 18:24:56 2026 GMT
Is client certificate authentication enabled: yes
```

Store in:

```bash
/opt/course/pl/etcd-info.txt
```

---

## 17. Create StorageClass local-kiddie

Create a new named `local-kiddie` with the provisioner `rancher.io/local-path`.
- Set the volumeBindingMode to `WaitForFirstConsumer`
- Configure the StorageClass to default StorageClass
- Do not modify any existing Deployment or PersistentVolumeClaim

**Solution:**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-kiddie
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path
volumeBindingMode: WaitForFirstConsumer
```

Apply it with:

```bash
kubectl apply -f storageclass.yaml
```

Make `storageclass.kubernetes.io/is-default-class: "false"` for the other StorageClass.

Use patch
`k patch sc local-kiddie -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class:"true"}}}'`

----

## 18. ETCD Repair

The cluster configured by kubeadm has been migrated to a new machine. It requires configuration changes to run successfully.
Task
- Repair the single — node cluster damaged during machine migration.
- First, identify the damaged cluster components and investigate the causes of their damage.
- 
**Note**: The decommissioned cluster uses an external etcd server. Next, repair the configuration of all damaged cluster components.
  
**Note**: Ensure that all necessary services and components are restarted for the changes to take effect. Failure to do so may result in a score reduction. Finally, ensure the cluster is running normally. Ensure: Each node and all Pods are in the Ready state.


`K get pods` (Will not work)

```bash
Vim /etc/kubernetes/manifest/kube-apiserver.yaml
```

Search for and correct the address
`/etcd-servers=https://127.0.0.1:2379`

```bash
Systemctl daemon-reload
Systemctl restart kubelet.service

K get pods -n kube-system
```
Kube-scheduler will be failing

```bash
Vim /etc/kubernetes/manifests/kube-scheduler.yaml
```

- Search for `/resources`
- Reduce CPU request to `100m`

----

## 19. Kube Proxy IP tables

You're asked to confirm that kube-proxy is running correctly. For this perform the following in Namespace `project-hamster`:
- Create Pod `p2-pod` with image `nginx:1-alpine`
- Create Service `p2-service` which exposes the Pod internally in the cluster on port `3000->80`

Write the iptables rules of node cka3962 belonging the created Service `p2-service` into file `/opt/course/p2/iptables.txt`

Delete the Service and confirm that the iptables rules are gone again

**Answer:**

Step 1: Create the Pod
First we create the Pod:

```bash
➜ candidate@cka3962:~$ k -n project-hamster run p2-pod --image=nginx:1-alpine
pod/p2-pod created
```
Step 2: Create the Service
```bash
➜ candidate@cka3962:~$ k -n project-hamster expose pod p2-pod --name p2-service --port 3000 --target-port 80

➜ candidate@cka3962:~$ k -n project-hamster get pod,svc,ep
NAME                 READY   STATUS    RESTARTS   AGE
pod/p2-pod           1/1     Running   0          2m31s

NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/p2-service   ClusterIP   10.105.128.247   <none>        3000/TCP   1s

NAME                   ENDPOINTS       AGE
endpoints/p2-service   10.44.0.31:80   1s
```
We should see that Pods and Services are connected, hence the Service should have Endpoints.

 

(Optional) Confirm kube-proxy is running and is using iptables
The idea here is to find the kube-proxy container and check its logs:
```bash
➜ candidate@cka3962:~$ sudo -i

➜ root@cka3962$ crictl ps | grep kube-proxy
67cccaf8310a1   505d571f5fd56   9 days ago      Running    kube-proxy ...

➜ root@cka3962~# crictl logs 67cccaf8310a1
I1029 14:10:23.984360       1 server_linux.go:66] "Using iptables proxy"
```
This could be repeated on each controlplane and worker node where the result should be the same.

 

Step 3: Check kube-proxy is creating iptables rules
Now we check the iptables rules on every node first manually:
```bash
➜ root@cka3962:~# iptables-save | grep p2-service
-A KUBE-SEP-55IRFJIRWHLCQ6QX -s 10.44.0.31/32 -m comment --comment "project-hamster/p2-service" -j KUBE-MARK-MASQ
-A KUBE-SEP-55IRFJIRWHLCQ6QX -p tcp -m comment --comment "project-hamster/p2-service" -m tcp -j DNAT --to-destination 10.44.0.31:80
-A KUBE-SERVICES -d 10.105.128.247/32 -p tcp -m comment --comment "project-hamster/p2-service cluster IP" -m tcp --dport 3000 -j KUBE-SVC-U5ZRKF27Y7YDAZTN
-A KUBE-SVC-U5ZRKF27Y7YDAZTN ! -s 10.244.0.0/16 -d 10.105.128.247/32 -p tcp -m comment --comment "project-hamster/p2-service cluster IP" -m tcp --dport 3000 -j KUBE-MARK-MASQ
-A KUBE-SVC-U5ZRKF27Y7YDAZTN -m comment --comment "project-hamster/p2-service -> 10.44.0.31:80" -j KUBE-SEP-55IRFJIRWHLCQ6QX
# Warning: iptables-legacy tables present, use iptables-legacy-save to see them
Great. Now let's write these logs into the requested file:

➜ root@cka3962:~# iptables-save | grep p2-service > /opt/course/p2/iptables.txt
 ```

Delete the Service and confirm iptables rules are gone
Delete the Service and confirm the iptables rules are gone::

```bash
➜ root@cka3962:~# k -n project-hamster delete svc p2-service
service "p2-service" deleted

➜ root@cka3962:~# iptables-save | grep p2-service

➜ root@cka3962:~#
Kubernetes Services are implemented using iptables rules (with default config) on all nodes. Every time a Service has been altered, created, deleted or Endpoints of a Service have changed, the kube-apiserver contacts every node's kube-proxy to update the iptables rules according to the current state.

 ```
----- 

## 20. Change Service CIDR
 
- Create a Pod named `check-ip` in Namespace `default` using image `httpd:2-alpine`
- Expose it on port `80` as a ClusterIP Service named `check-ip-service`. Remember to output the IP of that Service
- Change the Service CIDR to `11.96.0.0/12` for the cluster
- Create a second Service named `check-ip-service2` pointing to the same Pod


ℹ️ The second Service should get an IP address from the new CIDR range

 **Answer:**
Let's create the Pod and expose it:
```bash
➜ candidate@cka9412:~$ k run check-ip --image=httpd:2-alpine
pod/check-ip created

➜ candidate@cka9412:~$ k expose pod check-ip --name check-ip-service --port 80
And check the Service IP:

➜ candidate@cka9412:~$ k get svc
NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
check-ip-service   ClusterIP   10.109.84.110   <none>        80/TCP    13s
kubernetes         ClusterIP   10.96.0.1       <none>        443/TCP   9d
```
Now we change the Service CIDR in the kube-apiserver manifest:

```yaml
➜ candidate@cka9412:~$ sudo -i

➜ root@cka9412:~# vim /etc/kubernetes/manifests/kube-apiserver.yaml
# cka9412:/etc/kubernetes/manifests/kube-apiserver.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    component: kube-apiserver
    tier: control-plane
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=192.168.100.21
...
    - --service-account-key-file=/etc/kubernetes/pki/sa.pub
    - --service-cluster-ip-range=11.96.0.0/12             # change
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
...
```

We wait for the kube-apiserver to be restarted, which can take a minute:
```bash
➜ root@cka9412:~# watch crictl ps

➜ root@cka9412:~# kubectl -n kube-system get pod | grep api
kube-apiserver-cka9412            1/1     Running   0             20s
```
Now we do the same for the controller manager:
```bash
➜ root@cka9412:~# vim /etc/kubernetes/manifests/kube-controller-manager.yaml
# /etc/kubernetes/manifests/kube-controller-manager.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    component: kube-controller-manager
    tier: control-plane
  name: kube-controller-manager
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-controller-manager
    - --allocate-node-cidrs=true
    - --authentication-kubeconfig=/etc/kubernetes/controller-manager.conf
    - --authorization-kubeconfig=/etc/kubernetes/controller-manager.conf
    - --bind-address=127.0.0.1
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --cluster-cidr=10.244.0.0/16
    - --cluster-name=kubernetes
    - --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
    - --cluster-signing-key-file=/etc/kubernetes/pki/ca.key
    - --controllers=*,bootstrapsigner,tokencleaner
    - --kubeconfig=/etc/kubernetes/controller-manager.conf
    - --leader-elect=true
    - --node-cidr-mask-size=24
    - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
    - --root-ca-file=/etc/kubernetes/pki/ca.crt
    - --service-account-private-key-file=/etc/kubernetes/pki/sa.key
    - --service-cluster-ip-range=11.96.0.0/12         # change
    - --use-service-account-credentials=true
```
We wait for the kube-controller-manager to be restarted, which can take a minute:
```bash
➜ root@cka9412:~# watch crictl ps

➜ root@cka9412:~# kubectl -n kube-system get pod | grep controller
kube-controller-manager-cka9412   1/1     Running   0               39s
```
Checking our Service again:
```bash
➜ root@cka9412:~$ k get svc
NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
check-ip-service   ClusterIP   10.109.84.110   <none>        80/TCP    5m3s
kubernetes         ClusterIP   10.96.0.1       <none>        443/TCP   9d
Nothing changed so far. Now we create the second Service:

➜ root@cka9412:~$ k expose pod check-ip --name check-ip-service2 --port 80
And check again:

➜ root@cka9412:~$ k get svc
NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/check-ip-service    ClusterIP   10.109.84.110   <none>        80/TCP    5m55s
service/check-ip-service2   ClusterIP   11.105.52.114   <none>        80/TCP    29s
service/kubernetes          ClusterIP   10.96.0.1       <none>        443/TCP   9d

NAME                          ENDPOINTS             AGE
endpoints/check-ip-service    10.44.0.3:80          5m55s
endpoints/check-ip-service2   10.44.0.3:80          29s
endpoints/kubernetes          192.168.100.21:6443   9d
```
There we go, the new Service got an IP of the updated range assigned. We also see that both Services have our Pod as endpoint.


----
# Mock Test - 1


## Question 1 | Kubeconfig Context Extraction

You're asked to extract the following information out of kubeconfig file `/opt/course/l/kubeconfig` on cka9412 :
 - Write all kubeconfig context names into `/opt/course/l/contexts` , one per line
 - Write the name of the current context into `/opt/course/l/current-context`
 - Write the client-key of user account-base64-  decoded into `/opt/course/l/cert`

**Tasks:**

- Extract context names, current context, and client-key (base64 decoded)

**Solution:**
### Extract contexts:
```bash
kubectl config get-contexts --kubeconfig /opt/course/l/kubeconfig -o name > /opt/course/l/contexts
```
### Extract current context:
```bash
kubectl config current-context --kubeconfig /opt/course/l/kubeconfig > /opt/course/l/current-context
```

### Extract client-certificate-data:
```bash
 k config view --kubeconfig /opt/course/1/kubeconfig --raw -o jsonpath="{.users[0].user.client-certificate-data}" | base64 -d > /opt/course/1/cert
```
-- OR --
```bash

cat /opt/course/l/kubeconfig

Copy the `client-certificate-data` field 

# Then decode:
echo <client-certificate-data> | base64 -d > /opt/course/l/cert
```
---

## Question 2 | MinIO Operator, CRD Config, Helm Install

Install the MinlO Operator using Helm in Namespace minio . Then configure and create the Tenant CRD:
- Create Namespace `minio`
- Install Helm chart `minio/operator` into the new Namespace. The Helm Release should be called `minio—operator`
- Update the Tenant resource in `/opt/course/2/minio-tenant.yaml` to include `enableSFTP: true` under features
- Create the Tenant resource from `/opt/course/2/minio-tenant.yaml`. It is not required for MinlO to run properly.
Installing the Helm Chart and the Tenant resource as requested is enough

**Namespace:** `minio`

**Solution:**

```bash

k create ns minio

helm install minio-operator minio/operator -n minio

```

```bash
vim /opt/course/2/minio-tenant.yaml

# Add under `spec.features`
enableSFTP: true 
```
```bash
kubectl apply -f /opt/course/2/minio-tenant.yaml
```
----

## Question 3 | Scaledown Stateful sets

There are two Pods named `o3db-*` in Namespace `project-h800`. The Project H800 management asked you to scale these down to one replica to save resources.

```bash
➜ candidate@cka3962:~ k scale sts o3db --replicas 1 -n project-h800 
statefulset.apps/o3db scaled
```

---

## Question 4 | Find Pods first to be terminated: QOS Pods (BestEffort)

Solve this question on: `ssh cka2556`

Check all available _Pods_ in the _Namespace_ `project-c13` and find the names of those that would probably be terminated first if the nodes run out of resources (cpu or memory).

Write the _Pod_ names into `/opt/course/4/pods-terminated-first.txt`.


**Solution:**

```bash
kubectl get pods -n project-c13 -o custom-columns="NAME:.metadata.name,QOS:.status.qosClass" | grep "BestEffort" | awk '{print $1}' > /opt/course/4/pods-terminated-first.txt
```

The file will contain the Pod name

---

## Question 5 | Kustomize configure HPA Autoscaler

Previously the application api-gateway used some external autoscaler which should now be replaced with a HorizontalPodAutoscaler (HPA). 

The application has been deployed to Namespaces `api-gateway-staging` and `api-gateway-prod` like this:
```bash
kubectl kustomize /opt/course/5/api—gateway/staging
kubectl kustomize /opt/course/5/api—gateway/prod
```

Using the Kustomize config at `/opt/course/5/api-gateway` do the following:
- Remove the ConfigMap `horizontal—scaling—config` completely.
- Add HPA named `api-gateway` for the Deployment `api-gateway` with min 2 and max 4 replicas. It should scale at `50%` average CPU utilisation
- In prod the HPA should have max `6` replicas
- Apply your changes for staging and prod so they're reflected in the cluster

**Tasks:**

- Remove horizontal-scaling-config ConfigMap
- Add new HPA to base
- Patch HPA maxReplicas in prod
- Apply using `kubectl kustomize`


Remove the HPA from ConfigMap in 3 directory: Base, Staging, Prod
1. Remove the ConfigMap horizontal-scaling-config
   - Edit files `base/api-gateway.yaml`, `staging/api-gateway.yaml` and `prod/api-gateway.yaml` and remove the ConfigMap.
   - Locate and delete the ConfigMap manifest file (likely named `horizontal-scaling-config.yaml`) in `/opt/course/5/api-gateway/base` or wherever it is referenced.
   - Remove its reference from the `kustomization.yaml` under resources or configMapGenerator.
2. Add HPA for the Deployment
   - Create a new HPA manifest in the base
   - Create a file `/opt/course/5/api-gateway/base/hpa.yaml` with the following content:


**Base HPA Example:**

Add this in the `base/api-gateway.yaml` file

```yaml
# cka5774:/opt/course/5/api-gateway/base/api-gateway.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-gateway
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-gateway
  minReplicas: 2
  maxReplicas: 4
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: api-gateway
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
spec:
  replicas: 1
  selector:
    matchLabels:
      id: api-gateway
  template:
    metadata:
      labels:
        id: api-gateway
    spec:
      serviceAccountName: api-gateway
      containers:
        - image: httpd:2-alpine
          name: httpd
```
Notice that we don't specify a Namespace here as done also for the other resources. The Namespace will be set by staging and prod overlays automatically.


In prod the HPA should have max replicas set to 6 so we add this to the prod patch:
```yaml
# cka5774:/opt/course/5/api-gateway/prod/api-gateway.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-gateway
spec:
  maxReplicas: 6
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  labels:
    env: prod
```


Apply the changes.
```bash
kubectl apply -k /opt/course/5/api-gateway/staging
kubectl apply -k /opt/course/5/api-gateway/prod
```

Notice that the ConfogMap still exists. Delete them manually
```bash
k delete cm horizontal-scaling-cm -n api-gateway
```
---

## Question 7 | Metrics Server - Node and Pod Resource Usage

The metrics-server has been installed in the cluster. Write two bash scripts which use kubectl :
- Script `/opt/course/7/node.sh` should show resource usage of Nodes
- Script `/opt/course/7/pod.sh` should show resource usage of Pods and their containers

**Tasks:**
- Show node resource usage
- Show pod resource usage with containers

**Solution:**

/opt/course/7/node.sh
```bash
#!/bin/bash
kubectl top node
```

/opt/course/7/pod.sh
```bash
#!/bin/bash
kubectl top pod --containers
```

---

## Question 8 | Update Kubernetes Version and join cluster

**Tasks:**
- Upgrade Kubernetes version on worker node `cka3962-node1`
- Join node to cluster using `kubeadm`

**Solution:**
### Upgrade Node

```bash
controlplane:~$ k get nodes -o wide
NAME           STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
controlplane   Ready    control-plane   25d   v1.34.1   172.30.1.2    <none>        Ubuntu 24.04.1 LTS   6.8.0-51-generic   containerd://1.7.27
node01         Ready    <none>          25d   v1.33.2   172.30.2.2    <none>        Ubuntu 24.04.1 LTS   6.8.0-51-generic   containerd://1.7.27
```
      
ssh node01

Update the directory to point to the desired version
`vim /etc/apt/sources.list.d/kubernetes.list`

Search for kubeadm upgrade
```bash
sudo apt-mark unhold kubeadm && \
sudo apt-get update && sudo apt-get install -y kubeadm='1.34.1-1.1' && \
sudo apt-mark hold kubeadm
```

`sudo kubeadm upgrade plan`

`sudo kubeadm upgrade apply v1.34.1`


Upgrade kubelet and kubectl
```bash
sudo apt-mark unhold kubelet kubectl && \
sudo apt-get update && sudo apt-get install -y kubelet='1.34.x-*' kubectl='1.34.x-*' && \
sudo apt-mark hold kubelet kubectl
```

Restart the kubelet:
```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```


### Join the node to the cluster:
`ssh controlplane`

```bash
kubeadm token create --print-join-command
```

Copy the output and paste it after `ssh new-node`

----

## Question 9 | ServiceAccount to Access Secrets via API | Contact K8s Api from inside Pod

There is ServiceAccount `secret-reader` in Namespace `project-swan`. Create a Pod of image `nginx:1-alpine` named `api-contact` which uses this ServiceAccount. Exec into the Pod and use curl to manually query all Secrets from the Kubernetes Api. Write the result into file `/opt/course/9/result.json`.

**Tasks:**
- Use ServiceAccount `secret-reader` in `project-swan`
- Pod `api-contact` should query secrets and save to `/opt/course/9/result.json`

**Solution:**

```bash
kubectl run api-contact --image=nginx:1-alpine -n project-swan --dry-run=client -o yaml > q9-pod.yaml

# Add:
spec: 
  serviceAccountName: secret-reader

kubectl apply -f q9-pod.yaml
```

```bash
kubectl exec -it api-contact -n project-swan -it -- sh

TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)

curl -k https://kubernetes.defautt/api/vl/secrets -H "Authorization: Bearer ${TOKEN}"

curt -k https://kubernetes.defautt/api/vl/secrets -H "Authorization: Bearer ${TOKEN}" > result.json

exit

# Now in control-plane node

kubectl exec -it api-contact –n project-swan -- cat result.json > /opt/course/9/result.json
```

Connect via HTTPS with correct CA
To connect without curl -k we can specify the CertificateAuthority (CA):

```bash
CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

curl --cacert ${CACERT} https://kubernetes.default/api/v1/secrets -H "Authorization: Bearer ${TOKEN}"
```
---

## Question 10 | RBAC ServiceAccount Role RoleBinding

Create a new ServiceAccount `processor` in Namespace `project-hamster`. Create a Role and RoleBinding, both named processor as well. These should allow the new SA to only create Secrets and ConfigMaps in that Namespace.

**Namespace:** `project-hamster`

**Solution:**

```bash
K create sa processor -n project-hamster
```
Create a Role
```bash
kubectl create role processor --verb=create --resource=secret --resource=configmap -n project-hamster
```

It will create:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: processor
  namespace: project-hamster
rules:
- apiGroups:
  - ""
  resources:
  - secrets
  - configmaps
  verbs:
  - create
```

Create a `RoleBinding`
```bash
kubectl create role processor --verb=create --resource=secret --resource=configmap -n project-hamster
```
It will create:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: processor
  namespace: project-hamster
rules:
- apiGroups:
  - ""
  resources:
  - secrets
  - configmaps
  verbs:
  - create
```

Verify the work:
```bash
➜ candidate@cka3962:~$ k -n project-hamster auth can-i create secret --as system:serviceaccount:project-hamster:processor
yes

➜ candidate@cka3962:~$ k -n project-hamster auth can-i create configmap --as system:serviceaccount:project-hamster:processor
yes

➜ candidate@cka3962:~$ k -n project-hamster auth can-i create pod --as system:serviceaccount:project-hamster:processor
no

➜ candidate@cka3962:~$ k -n project-hamster auth can-i delete secret --as system:serviceaccount:project-hamster:processor
no

➜ candidate@cka3962:~$ k -n project-hamster auth can-i get configmap --as system:serviceaccount:project-hamster:processor
no
```

---

## Question 11 |  DaemonSet on all Nodes with Resource Limits

Use Namespace project-tiger for the following. Create a DaemonSet named `ds-important` with image `httpd:2-alpine` and labels `id=ds-important` and `uuid=18426aØbf59-4e1Ø-923f-cØeØ78e82462`. The Pods it creates should request `10 millicore cpu` and `10 mebibyte memory`. The Pods of that DaemonSet should run on all nodes, also controlplanes.

**Namespace:** `project-tiger`

**Solution:**

```bash
k create deployment ds-important --image=httpd:2.4-alpine -n project-tiger --dry-run=client -o yaml > 11.yaml
```

Replace `Deployment` with `DaemonSet`

```yaml
apiVersion: apps/v1
kind: DaemonSet                                     # change from Deployment to Daemonset
metadata:
  creationTimestamp: null
  labels:                                           # add
    id: ds-important                                # add
    uuid: 18426a0b-5f59-4e10-923f-c0e078e82462      # add
  name: ds-important
  namespace: project-tiger                          # important
spec:
  #replicas: 1                                      # remove
  selector:
    matchLabels:
      id: ds-important                              # add
      uuid: 18426a0b-5f59-4e10-923f-c0e078e82462    # add
  #strategy: {}                                     # remove
  template:
    metadata:
      creationTimestamp: null
      labels:
        id: ds-important                            # add
        uuid: 18426a0b-5f59-4e10-923f-c0e078e82462  # add
    spec:
      containers:
      - image: httpd:2-alpine
        name: ds-important
        resources:
          requests:                                 # add
            cpu: 10m                                # add
            memory: 10Mi                            # add
      tolerations:                                  # add
      - effect: NoSchedule                          # add
        key: node-role.kubernetes.io/control-plane  # add
#status: {}  
```

It was requested that the DaemonSet runs on all nodes, so we need to specify the toleration for this.

---

## Question 14 | Check how long certificates are valid

Check how long the kube-apiserver server certificate is valid using openssl or cfssl. 
- Write the expiration date into `/opt/course/14/expiration`. Run the kubeadm command to list the expiration dates and confirm both methods show the same one.
- Write the kubeadm command that would renew the `kube-apiserver` certificate into `/opt/course/14/kubeadm-renew-certs.sh`

**Solution:**

```bash
openssl x509 -noout -text -in /etc/kubernetes/pki/apiserver.crt 

Check for “Validity: Not after” field
```

```bash
openssl x509 -noout -text -in /etc/kubernetes/pki/apiserver.crt | grep Validity -A2
>>>     Validity
            Not Before: Oct 29 14:14:27 2024 GMT
            Not After : Oct 29 14:19:27 2025 GMT
```

Store expiration in:

```bash
# /opt/course/14/expiration
Oct 29 14:19:27 2025 GMT
```

Store renew command in: `/opt/course/14/kubeadm-renew-certs.sh`

```bash
# /opt/course/14/kubeadm-renew-certs.sh
kubeadm certs renew apiserver
```

---
## Question 15 | NetworkPolicy

There was a security incident where an intruder was able to access the whole cluster from a single hacked backend _Pod_.

To prevent this create a _NetworkPolicy_ called `np-backend` in _Namespace_ `project-snake`. It should allow the `backend-*` _Pods_ only to:
-   Connect to `db1-*` _Pods_ on port `1111`
-   Connect to `db2-*` _Pods_ on port `2222`

Use the `app` _Pod_ labels in your policy.

- ℹ️ All _Pods_ in the _Namespace_ run plain Nginx images. This allows simple connectivity tests like: `k -n project-snake exec POD_NAME -- curl POD_IP:PORT`

- ℹ️ For example, connections from `backend-*` _Pods_ to `vault-*` _Pods_ on port `3333` should no longer work

**Solution**:

```bash
➜ candidate@cka7968:~$ k -n project-snake get pod -L app
NAME        READY   STATUS    RESTARTS   AGE   APP
backend-0   1/1     Running   0          8d    backend
db1-0       1/1     Running   0          8d    db1
db2-0       1/1     Running   0          8d    db2
vault-0     1/1     Running   0          8d    vault
```

Copy NP.yaml from Documentation
```yaml
# cka7968:/home/candidate/15_np.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: np-backend
  namespace: project-snake
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Egress                    # policy is only about Egress
  egress:
    -                           # first rule
      to:                           # first condition "to"
      - podSelector:
          matchLabels:
            app: db1
      ports:                        # second condition "port"
      - protocol: TCP
        port: 1111
    -                           # second rule
      to:                           # first condition "to"
      - podSelector:
          matchLabels:
            app: db2
      ports:                        # second condition "port"
      - protocol: TCP
        port: 2222
```

`k create -f np.yaml`

The NP above has two rules with two conditions each, it can be read as:
```bash
allow outgoing traffic if:
  (destination pod has label app=db1 AND port is 1111)
  OR
  (destination pod has label app=db2 AND port is 2222)
```

---

## Question 16 | Update CoreDNS Configuration

The CoreDNS configuration in the cluster needs to be updated,
- Make a backup of the existing configuration Yaml and store it at `/opt/course/16/coredns_backup.yml`. You should be able to fast recover from the backup
- Update the CoreDNS configuration in the cluster so that DNS resolution for `SERVICE.NAMESPACE.custom-domain` will work exactly like and in addition to `SERVICE.NAMESPACE.cluster.local`
Test your configuration for example from a Pod with `busybox:1-image`. These commands should result in an address:
```bash
nslookup kubernetes.default.svc.cluster.local
nslookup kubernetes.default.svc.custom-domain
```

**Solution:**

```bash
kubectl get cm coredns -n kube-system -o yaml > /opt/course/16/coredns_backup.yaml

kubectl edit cm coredns -n kube-system
```

Update `data.Corefile`:
add `custom-domain`
```bash
kubernetes custom-domain cluster.local in-addr.arpa ip6.arpa {
  pods insecure
  fallthrough in-addr.arpa ip6.arpa
  ttl 30
}
```

Restart:

```bash
kubectl rollout restart deployment coredns -n kube-system
```

Verify:
- Run both the `nslookup` commands from the question.

---

## Question 17 | Find Container of Pod and check info

Solve this question on: Namespace project-tiger create a Pod named `tigers-reunite` of image `httpd:2-alpine` with labels `pod=container` and `container=pod`. Find out on which node the Pod is scheduled. SSH into that node and find the containerd container belonging to that Pod.

**Tasks:**
- Find `container ID` and `runtimeType`
- Write the ID of the container and the `info.runtimeType` into `/opt/course/17/pod-container.txt`
- Write the logs of the container into `/opt/course/17/pod-container.log`

ℹ️ You can connect to a worker node using `ssh cka2556-node1` or `ssh cka2556-node2` from `cka2556`


**Solution:**

```bash

ssh cka2556
kubectl run tigers-reunite --image=httpd:2-alpine -n project-tiger --labels="pod=container,container=pod"

```
Find the node where `tigers-reunite` is scheduled on
`k get pod -o wide -n project-tiger`

```bash
# Find node it's running on, SSH there:
ssh <node-name>

sudo -i

crictl ps | grep tigers-reunite

crictl inspect <containerID> | grep runtimeType > /opt/course/17/pod-container.txt


sudo -i

crictl logs <containerID> > /opt/course/17/pod-container.log
```

---
  
----
# Mock Test - 2

## 1. DNS / FQDN / Headless Service
Question 1 | DNS / FQDN / Headless Service
 
Solve this question on: ssh cka6016

The Deployment controller in Namespace `lima-control` communicates with various cluster internal endpoints by using their DNS FQDN values.
Update the ConfigMap used by the Deployment with the correct FQDN values for:

- DNS_1: Service kubernetes in Namespace `default`
- DNS_2: Headless Service department in Namespace `lima-workload`
- DNS_3: Pod section100 in Namespace `lima-workload`. It should work even if the Pod IP changes
- DNS_4: A Pod with IP `1.2.3.4` in Namespace `kube-system`

Ensure the Deployment works with the updated values.

ℹ️ You can use nslookup inside a Pod of the controller Deployment

Answer:
For this question we need to understand how cluster internal DNS works in Kubernetes. The most common use is `SERVICE.NAMESPACE.svc.cluster.local` which will resolve to the IP address of the Kubernetes Service. Note that we're asked to specify the FQDNs here so short values like `SERVICE.NAMESPACE` are not possible even if they would work.
Let's exec into the Pod for testing:


**Solution**
We should update the ConfigMap:
```bash
➜ candidate@cka6016:~$ k -n lima-control edit cm control-config
apiVersion: v1
data:
  DNS_1: kubernetes.default.svc.cluster.local                  # UPDATE
  DNS_2: department.lima-workload.svc.cluster.local            # UPDATE
  DNS_3: section100.section.lima-workload.svc.cluster.local    # UPDATE
  DNS_4: 1-2-3-4.kube-system.pod.cluster.local                 # UPDATE
kind: ConfigMap
metadata:
  name: control-config
  namespace: lima-control
```
Restart the Deployment:

```bash
➜ candidate@cka6016:~$ kubectl -n lima-control rollout restart deploy controller
deployment.apps/controller restarted
```
 
--- 

## Question 2 | Create a Static Pod and Service

- Create a Static Pod named `my-static-pod` in Namespace `default` on the controlplane node. It should be of image `nginx:1-alpine` and have resource requests for `10m` CPU and `20Mi` memory.
- Create a NodePort Service named `static-pod-service` which exposes that static Pod on port `80`.

ℹ️ For verification check if the new Service has one Endpoint. It should also be possible to access the Pod via the `cka2560` internal IP address, like using `curl 192.168.100.31:NODE_PORT`

Answer:

```bash
➜ candidate@cka2560:~$ sudo -i

➜ root@cka2560:~# cd /etc/kubernetes/manifests/

➜ root@cka2560:~# k run my-static-pod --image=nginx:1-alpine -o yaml --dry-run=client > my-static-pod.yaml
Then edit the my-static-pod.yaml to add the requested resource requests:

# cka2560:/etc/kubernetes/manifests/my-static-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: my-static-pod
  name: my-static-pod
spec:
  containers:
  - image: nginx:1-alpine
    name: my-static-pod
    resources:
      requests:
        cpu: 10m
        memory: 20Mi
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

Now we expose that static Pod:
```yaml
➜ root@cka2560:~# k expose pod my-static-pod-cka2560 --name static-pod-service --type=NodePort --port 80
This will generate a Service yaml like:

apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    run: my-static-pod
  name: static-pod-service
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: my-static-pod
  type: NodePort
status:
  loadBalancer: {}
  
```

Then we check the Service and Endpoints:
```bash
➜ root@cka2560:~# k get svc,ep -l run=my-static-pod
NAME                         TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/static-pod-service   NodePort   10.98.249.240   <none>        80:32699/TCP   34s

NAME                           ENDPOINTS      AGE
endpoints/static-pod-service   10.32.0.4:80   34s
Also we should be able to access that Nginx container, your NodePort might be different than the one used here:

➜ root@cka2560:~# k get node -owide
NAME            STATUS   ROLES           AGE   VERSION   INTERNAL-IP      ...
cka2560         Ready    control-plane   8d    v1.33.1   192.168.100.31   ...

➜ root@cka2560:~# curl 192.168.100.31:32699
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
 
---- 
## Question 3 | Kubelet client/server cert info
 
Solve this question on: `ssh cka5248`

Node `cka5248-node1` has been added to the cluster using kubeadm and TLS bootstrapping.

Find the `Issuer` and `Extended Key Usage` values on `cka5248-node1` for:
- Kubelet Client Certificate, the one used for outgoing connections to the `kube-apiserver`
- Kubelet Server Certificate, the one used for incoming connections from the `kube-apiserver`

Write the information into file `/opt/course/3/certificate-info.txt`.

ℹ️ You can connect to the worker node using `ssh cka5248-node1` from `cka5248`

**Answer**:
First we check the kubelet client certificate:
```bash
➜ ssh cka5248

➜ candidate@cka5248:~$ ssh cka5248-node1

➜ candidate@cka5248-node1:~$ sudo -i

➜ root@cka5248-node1:~# find /var/lib/kubelet/pki
/var/lib/kubelet/pki
/var/lib/kubelet/pki/kubelet-client-2024-10-29-14-24-14.pem
/var/lib/kubelet/pki/kubelet.crt
/var/lib/kubelet/pki/kubelet.key
/var/lib/kubelet/pki/kubelet-client-current.pem

➜ root@cka5248-node1:~# openssl x509 -noout -text -in /var/lib/kubelet/pki/kubelet-client-current.pem | grep Issuer
        Issuer: CN = kubernetes
        
➜ root@cka5248-node1:~# openssl x509 -noout -text -in /var/lib/kubelet/pki/kubelet-client-current.pem | grep "Extended Key Usage" -A1
            X509v3 Extended Key Usage: 
                TLS Web Client Authentication
                
```
Next we check the kubelet server certificate:
```bash
➜ root@cka5248-node1:~# openssl x509 -noout -text -in /var/lib/kubelet/pki/kubelet.crt | grep Issuer
        Issuer: CN = cka5248-node1-ca@1730211854

➜ root@cka5248-node1:~# openssl x509 -noout -text -in /var/lib/kubelet/pki/kubelet.crt | grep "Extended Key Usage" -A1
            X509v3 Extended Key Usage: 
                TLS Web Server Authentication
```
We see that the server certificate was generated on the worker node itself and the client certificate was issued by the Kubernetes api. The Extended Key Usage also shows if it's for client or server authentication.

The solution file should look something like this:
```bash
# cka5248:/opt/course/3/certificate-info.txt
Issuer: CN = kubernetes
X509v3 Extended Key Usage: TLS Web Client Authentication
Issuer: CN = cka5248-node1-ca@1730211854
X509v3 Extended Key Usage: TLS Web Server Authentication
 ```
 
 ----

## Question 4 | Pod Ready if Service is reachable
 
Solve this question on: ssh cka3200

Do the following in Namespace `default`:

- Create a Pod named `ready-if-service-ready` of image `nginx:1-alpine`
- Configure a LivenessProbe which simply executes command `true`
- Configure a ReadinessProbe which does check if the url `http://service-am-i-ready:80` is reachable, you can use `wget -T2 -O- http://service-am-i-ready:80` for this

Start the Pod and confirm it isn't ready because of the ReadinessProbe.

Then:
- Create a second Pod named `am-i-ready` of image `nginx:1-alpine` with label `id: cross-server-ready`.
- The already existing Service `service-am-i-ready` should now have that second Pod as endpoint

Check that the first Pod should be in ready state.

**Answer**:

It's a bit of an anti-pattern for one Pod to check another Pod for being ready using probes, hence the normally available `readinessProbe.httpGet` doesn't work for absolute remote urls. Still the workaround requested in this task should show how probes and Pod<->Service communication works.

First we create the first Pod:
```bash
➜ ssh cka3200

➜ candidate@cka3200:~$ k run ready-if-service-ready --image=nginx:1-alpine --dry-run=client -o yaml > 4_pod1.yaml

# cka3200: vim /home/candidate/4_pod1.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: ready-if-service-ready
  name: ready-if-service-ready
spec:
  containers:
  - image: nginx:1-alpine
    name: ready-if-service-ready
    resources: {}
    livenessProbe:                                      # add from here
      exec:
        command:
        - 'true'
    readinessProbe:
      exec:
        command:
        - sh
        - -c
        - 'wget -T2 -O- http://service-am-i-ready:80'   # to here
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```
Then create the Pod and confirm it's in a non-ready state:
```bash
k -f 4_pod1.yaml create
pod/ready-if-service-ready created
```

We can also check the reason for this using describe:
```bash
➜ candidate@cka3200:~$ k describe pod ready-if-service-ready
...
  Warning  Unhealthy  7s (x4 over 23s)  kubelet            Readiness probe failed: command timed out: "sh -c wget -T2 -O- http://service-am-i-ready:80" timed out after 1s
```

Now we create the second Pod:
```bash
➜ candidate@cka3200:~$ k run am-i-ready --image=nginx:1-alpine --labels="id=cross-server-ready"
pod/am-i-ready created
```
The already existing Service `service-am-i-ready` should now have an Endpoint:
```bash
➜ candidate@cka3200:~$ k describe svc service-am-i-ready
Name:                     service-am-i-ready
Namespace:                default
Labels:                   id=cross-server-ready
Annotations:              <none>
Selector:                 id=cross-server-ready
Type:                     ClusterIP
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.108.19.168
IPs:                      10.108.19.168
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
Endpoints:                10.44.0.30:80
Session Affinity:         None
Internal Traffic Policy:  Cluster
Events:                   <none>

➜ candidate@cka3200:~$ k get ep service-am-i-ready 
NAME                 ENDPOINTS       AGE
service-am-i-ready   10.44.0.30:80   6d19h
```

Which will result in our first Pod being ready, just give it a minute for the Readiness probe to check again:
```bash
➜ candidate@cka3200:~$ k get pod ready-if-service-ready
NAME                     READY   STATUS    RESTARTS   AGE
ready-if-service-ready   1/1     Running   0          2m10s
```
Look at these Pods working together!

 ---

## Question 5 | Kubectl sorting
 
Solve this question on: `ssh cka8448`

Create two bash script files which use kubectl sorting to:
- Write a command into `/opt/course/5/find_pods.sh` which lists all Pods in all Namespaces sorted by their AGE `metadata.creationTimestamp`
- Write a command into `/opt/course/5/find_pods_uid.sh` which lists all Pods in all Namespaces sorted by field `metadata.uid`

 
**Answer**:

### Step 1
```bash
vim /opt/course/5/find_pods.sh
kubectl get pod -A --sort-by=.metadata.creationTimestamp
```

### Step 2
```bash
vim /opt/course/5/find_pods_uid.sh
kubectl get pod -A --sort-by=.metadata.uid
```

 ---
 
## Question 6 | Fix Kubelet

There seems to be an issue with the kubelet on controlplane `node cka1024`, it's not running.

Fix the kubelet and confirm that the node is available in Ready state.
- Create a Pod called success in `default` Namespace of image `nginx:1-alpine`.

 ℹ️ The node has no taints and can schedule Pods without additional tolerations

**Answer**:

Investigate, Check node status:
```bash
➜ ssh cka1024

➜ candidate@cka1024:~$ k get node
E0423 12:27:08.326639   12871 memcache.go:265] "Unhandled Error" err="couldn't get current server API group list: Get \"https://192.168.100.41:6443/api?timeout=32s\": dial tcp 192.168.100.41:6443: connect: connection refused"
E0423 12:27:08.329430   12871 memcache.go:265] "Unhandled Error" err="couldn't get current server API group list: Get \"https://192.168.100.41:6443/api?timeout=32s\": dial tcp 192.168.100.41:6443: connect: connection refused"
E0423 12:27:08.332448   12871 memcache.go:265] "Unhandled Error" err="couldn't get current server API group list: Get \"https://192.168.100.41:6443/api?timeout=32s\": dial tcp 192.168.100.41:6443: connect: connection refused"
E0423 12:27:08.335352   12871 memcache.go:265] "Unhandled Error" err="couldn't get current server API group list: Get \"https://192.168.100.41:6443/api?timeout=32s\": dial tcp 192.168.100.41:6443: connect: connection refused"
E0423 12:27:08.342153   12871 memcache.go:265] "Unhandled Error" err="couldn't get current server API group list: Get \"https://192.168.100.41:6443/api?timeout=32s\": dial tcp 192.168.100.41:6443: connect: connection refused"
The connection to the server 192.168.100.41:6443 was refused - did you specify the right host or port?
```
Okay, this looks very wrong. First we check if the kubelet is running:
```bash
➜ candidate@cka1024:~$ sudo -i

➜ root@cka1024:~# ps aux | grep kubelet
root       12892  0.0  0.1   7076  ...  0:00 grep --color=auto kubelet
```
No kubectl process running, just the grep command itself is displayed. 

We check if the kubelet is configured as service, which is default for a kubeadm installation:
```bash
➜ root@cka1024:~# service kubelet status
○ kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; preset: enabled)
    Drop-In: /usr/lib/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: inactive (dead) since Sun 2025-03-23 08:16:52 UTC; 1 month 0 days ago
   Duration: 2min 46.830s
       Docs: https://kubernetes.io/docs/
   Main PID: 7346 (code=exited, status=0/SUCCESS)
        CPU: 5.956s
```
We can see it's not running (inactive) in this line:

Active: inactive (dead) since Sun 2025-03-23 08:16:52 UTC; 1 month 0 days ago

But the kubelet is configured as a service with config at `/usr/lib/systemd/system/kubelet.service`, let's try to start it:

```bash
➜ root@cka1024:~# service kubelet start

➜ root@cka1024:~# service kubelet status
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; preset: enabled)
    Drop-In: /usr/lib/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: activating (auto-restart) (Result: exit-code) since Wed 2025-04-23 12:31:07 UTC; 2s ago
       Docs: https://kubernetes.io/docs/
    Process: 13014 ExecStart=/usr/local/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EX>
   Main PID: 13014 (code=exited, status=203/EXEC)
        CPU: 10ms

Apr 23 12:31:07 cka1024 systemd[1]: kubelet.service: Failed with result 'exit-code'.
```
Above we see it's trying to execute `/usr/local/bin/kubelet` in this line:

```bash
Process: 13014 ExecStart=/usr/local/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EX>
It does so with some arguments defined in its service config file. A good way to find errors and get more info is to run the command manually:

➜ root@cka1024:~# /usr/local/bin/kubelet
-bash: /usr/local/bin/kubelet: No such file or directory

➜ root@cka1024:~# whereis kubelet
kubelet: /usr/bin/kubelet
That's the issue: wrong path to the kubelet binary.
``` 

Read Logs
Usually we need to dig a bit deeper and check logs using `journalctl -u kubelet or cat /var/log/syslog | grep kubelet`:

```bash
➜  root@cka1024:~# cat /var/log/syslog | grep kubelet
2025-03-23T08:13:26.775366+00:00 ubuntu systemd[1]: Started kubelet.service - kubelet: The Kubernetes Node Agent.
2025-03-23T08:13:26.782571+00:00 ubuntu (kubelet)[6826]: kubelet.service: Referenced but unset environment variable evaluates to an empty string: KUBELET_KUBEADM_ARGS
...
2025-04-23T12:31:48.264234+00:00 ubuntu systemd[1]: kubelet.service: Scheduled restart job, restart counter is at 5.
2025-04-23T12:31:48.272108+00:00 ubuntu systemd[1]: Started kubelet.service - kubelet: The Kubernetes Node Agent.
2025-04-23T12:31:48.284966+00:00 ubuntu systemd[1]: kubelet.service: Main process exited, code=exited, status=203/EXEC
2025-04-23T12:31:48.285487+00:00 ubuntu systemd[1]: kubelet.service: Failed with result 'exit-code'.
If we check logs we should always look at the time, we probably only want the latest ones. Here we see:

kubelet.service: Main process exited, code=exited, status=203/EXEC

```
The logs don't show any error messages from the kubelet itself. Usually if the kubelet is started and exits because of an error, like an unknown argument passed, there will be error logs. But because there is nothing more here it could be a good idea to try to execute the kubelet binary manually.

We already did this above before checking the logs and it showed us that a wrong binary path was used in the service config file.


Fix the Kubelet
We go ahead and correct the path in file `/usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf`:

```bash
➜ root@cka1024:~# vim /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
# cka1024:/usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
```

Note: This dropin only works with kubeadm and kubelet v1.11+
```bash
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"

Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically

EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
EnvironmentFile=-/etc/default/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS
In the very last line we updated the binary path to /usr/bin/kubelet.

Now we reload the service:
```bash
➜ root@cka1024:~# systemctl daemon-reload

➜ root@cka1024:~# service kubelet restart

➜ root@cka1024:~# service kubelet status
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; preset: enabled)
    Drop-In: /usr/lib/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: active (running) since Wed 2025-04-23 12:33:25 UTC; 5s ago
       Docs: https://kubernetes.io/docs/
   Main PID: 13124 (kubelet)
      Tasks: 9 (limit: 1317)
     Memory: 88.3M (peak: 88.6M)
        CPU: 1.093s
     CGroup: /system.slice/kubelet.service
             └─13124 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/ku>
...
```
```bash
➜ root@cka1024:~# ps aux | grep kubelet
root       13124  9.2  7.1 1896084 82432 ?       Ssl  12:33   0:01 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock --pod-infra-container-image=registry.k8s.io/pause:3.10
...
That looks much better. We can wait for the containers to appear, which can take a minute:

➜ root@cka1024:~# watch crictl ps
CONTAINER      ...  CREATED           STATE       NAME                      ...
ccfbd17742b05  ...  25 seconds ago    Running     kube-controller-manager   ...
ff3910e3c8c6c  ...  25 seconds ago    Running     kube-scheduler            ...
9b49473786774  ...  25 seconds ago    Running     kube-apiserver            ...
f5de1f6e11d5c  ...  26 seconds ago    Running     etcd                      ...
```
ℹ️ In this environment `crictl` can be used for container management. In the real exam this could also be docker. Both commands can be used with the same arguments.

Also the node should be available, give it a bit of time though:
```bash
➜ root@cka1024:~# k get node
NAME      STATUS   ROLES           AGE   VERSION
cka1024   Ready    control-plane   31d   v1.33.1
```
ℹ️ It might take some time till k get node doesn't throw any errors after fixing the issue

Finally we create the requested Pod:
```bash
➜ root@cka1024:~# k run success --image nginx:1-alpine
pod/success created

➜ root@cka1024:~# k get pod success -o wide
NAME      READY   STATUS    ...   NODE      NOMINATED NODE   READINESS GATES
success   1/1     Running   ...   cka1024   <none>           <none>
 ```

---

## Question 7 | Etcd Operations
 
Solve this question on: ssh cka2560

You have been tasked to perform the following etcd operations:
- Run etcd --version and store the output at `/opt/course/7/etcd-version`
- Make a snapshot of etcd and save it at `/opt/course/7/etcd-snapshot.db`

**Answer**:
### Step 1: Etcd Version
Here we simply need to execute a command, shouldn't be that hard:

```bash
➜ candidate@cka2560:~$ sudo -i

➜ root@cka2560:~# etcd --version
Command 'etcd' not found, but can be installed with:
apt install etcd-server
Well, etcd is not installed directly on the controlplane but it runs as a Pod instead. So we do:

root@cka2560:~# k -n kube-system exec etcd-cka2560 -- etcd --version
etcd Version: 3.5.21
Git SHA: a17edfd
Go Version: go1.23.7
Go OS/Arch: linux/amd64

root@cka2560:~# k -n kube-system exec etcd-cka2560 -- etcd --version > /opt/course/7/etcd-version
 
```

### Step 2: Etcd Snapshot
First we log into the controlplane and try to create a snapshot of etcd:

```bash
➜ candidate@cka2560:~$ sudo -i

➜ root@cka2560:~# ETCDCTL_API=3 etcdctl snapshot save /opt/course/7/etcd-snapshot.db
{"level":"info","ts":"2024-11-07T14:02:17.746254Z","caller":"snapshot/v3_snapshot.go:65","msg":"created temporary db file","path":"/opt/course/7/etcd-snapshot.db.part"}
^C
But it fails or hangs because we need to authenticate ourselves. For the necessary information we can check the etc manifest:

➜ root@cka2560:~# vim /etc/kubernetes/manifests/etcd.yaml
```

We only check the etcd.yaml for necessary information we don't change it.
```bash
# cka2560:/etc/kubernetes/manifests/etcd.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    component: etcd
    tier: control-plane
  name: etcd
  namespace: kube-system
spec:
  containers:
  - command:
    - etcd
    - --advertise-client-urls=https://192.168.100.31:2379
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt                           # use
    - --client-cert-auth=true
    - --data-dir=/var/lib/etcd
    - --initial-advertise-peer-urls=https://192.168.100.31:2380
    - --initial-cluster=cka2560=https://192.168.100.31:2380
    - --key-file=/etc/kubernetes/pki/etcd/server.key                            # use
    - --listen-client-urls=https://127.0.0.1:2379,https://192.168.100.31:2379   # use
    - --listen-metrics-urls=http://127.0.0.1:2381
    - --listen-peer-urls=https://192.168.100.31:2380
    - --name=cka2560
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
    - --peer-client-cert-auth=true
    - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
    - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt                    # use
    - --snapshot-count=10000
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    image: k8s.gcr.io/etcd:3.3.15-0
    imagePullPolicy: IfNotPresent
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 127.0.0.1
        path: /health
        port: 2381
        scheme: HTTP
      initialDelaySeconds: 15
      timeoutSeconds: 15
    name: etcd
    resources: {}
    volumeMounts:
    - mountPath: /var/lib/etcd
      name: etcd-data
    - mountPath: /etc/kubernetes/pki/etcd
      name: etcd-certs
  hostNetwork: true
  priorityClassName: system-cluster-critical
  volumes:
  - hostPath:
      path: /etc/kubernetes/pki/etcd
      type: DirectoryOrCreate
    name: etcd-certs
  - hostPath:
      path: /var/lib/etcd                                                     # important
      type: DirectoryOrCreate
    name: etcd-data
status: {}
```
But we also know that the api-server is connecting to etcd, so we can check how its manifest is configured:
```bash
➜ root@cka2560:~# cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep etcd
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
    - --etcd-servers=https://127.0.0.1:2379
```
We use the authentication information and pass it to etcdctl:

```bash
ETCDCTL_API=3 etcdctl snapshot save /opt/course/7/etcd-snapshot.db \
--cacert /etc/kubernetes/pki/etcd/ca.crt \
--cert /etc/kubernetes/pki/etcd/server.crt \
--key /etc/kubernetes/pki/etcd/server.key
```
Which should provide successful output:
Snapshot saved at /opt/course/7/etcd-snapshot.db

ℹ️ Don't use snapshot status because it can alter the snapshot file and render it invalid in certain etcd versions

 

(Optional) Etcd Restore
ℹ️ Doing this wrong can leave this cluster broken and will affect this question and also others

We create a Pod in the cluster and wait for it to be running:

```bash
➜ root@cka2560:~# kubectl run test --image=nginx
pod/test created

➜ root@cka2560:~# kubectl get pod -l run=test
NAME   READY   STATUS    RESTARTS   AGE
test   1/1     Running   0          17s
Next we stop all controlplane components:

➜ root@cka2560:~# cd /etc/kubernetes/manifests/

➜ root@cka2560:/etc/kubernetes/manifests# mv * ..

➜ root@cka2560:/etc/kubernetes/manifests# watch crictl ps
```
It's very important to wait for all K8s controlplane containers to be removed before continuing. This can take a minute!

ℹ️ In this environment crictl can be used for container management. In the real exam this could also be docker. Both commands can be used with the same arguments.

Now we restore the snapshot into a specific directory:
```bash
➜ root@cka2560:~# TCDCTL_API=3 etcdctl snapshot restore /opt/course/7/etcd-snapshot.db --data-dir /var/lib/etcd-snapshot --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key
Deprecated: Use `etcdutl snapshot restore` instead.

2025-03-02T13:38:07Z    info    snapshot/v3_snapshot.go:265     restoring snapshot      {"path": "/opt/course/7/etcd-snapshot.db", "wal-dir": "/var/lib/etcd-snapshot/member/wal", "data-dir": "/var/lib/etcd-snapshot", "snap-dir": "/var/lib/etcd-snapshot/member/snap", "initial-memory-map-size": 0}
2025-03-02T13:38:07Z    info    membership/store.go:141 Trimming membership information from the backend...
2025-03-02T13:38:07Z    info    membership/cluster.go:421       added member    {"cluster-id": "cdf818194e3a8c32", "local-member-id": "0", "added-peer-id": "8e9e05c52164694d", "added-peer-peer-urls": ["http://localhost:2380"]}
2025-03-02T13:38:08Z    info    snapshot/v3_snapshot.go:293     restored snapshot       {"path": "/opt/course/7/etcd-snapshot.db", "wal-dir": "/var/lib/etcd-snapshot/member/wal", "data-dir": "/var/lib/etcd-snapshot", "snap-dir": "/var/lib/etcd-snapshot/member/snap", "initial-memory-map-size": 0}
We could specify another host to make the backup from by using etcdctl --endpoints http://IP, but here we just use the default value which is: http://127.0.0.1:2379,http://127.0.0.1:4001.
```
The restored files are located at the new folder `/var/lib/etcd-snapshot`, now we have to tell etcd to use that directory:
```bash
➜ root@cka2560:~# vim /etc/kubernetes/etcd.yaml
# /etc/kubernetes/etcd.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    component: etcd
    tier: control-plane
  name: etcd
  namespace: kube-system
spec:
...
    - mountPath: /etc/kubernetes/pki/etcd
      name: etcd-certs
  hostNetwork: true
  priorityClassName: system-cluster-critical
  volumes:
  - hostPath:
      path: /etc/kubernetes/pki/etcd
      type: DirectoryOrCreate
    name: etcd-certs
  - hostPath:
      path: /var/lib/etcd-snapshot                # change
      type: DirectoryOrCreate
    name: etcd-data
status: {}
```
Now we move all controlplane yaml again into the manifest directory. Give it some time (up to several minutes) for etcd to restart and for the api-server to be reachable again:
```bash
➜ root@cka2560:/etc/kubernetes/manifests# mv ../*.yaml .

➜ root@cka2560:/etc/kubernetes/manifests# watch crictl ps
Then we check again for the Pod:

➜ root@cka2560:~# kubectl get pod -l run=test
No resources found in default namespace.
Awesome, snapshot and restore worked as our Pod is gone.
```
 ---

## Question 8 | Get Controlplane Information
 
Solve this question on: ssh cka8448

- Check how the controlplane components kubelet, kube-apiserver, kube-scheduler, kube-controller-manager and etcd are started/installed on the controlplane node.
- Also find out the name of the DNS application and how it's started/installed in the cluster.
- Write your findings into file `/opt/course/8/controlplane-components.txt`.
- The file should be structured like:
```bash
# /opt/course/8/controlplane-components.txt
kubelet: [TYPE]
kube-apiserver: [TYPE]
kube-scheduler: [TYPE]
kube-controller-manager: [TYPE]
etcd: [TYPE]
dns: [TYPE] [NAME]
Choices of [TYPE] are: not-installed, process, static-pod, pod
``` 

**Answer**:
We could start by finding processes of the requested components, especially the kubelet at first:

```bash
➜ candidate@cka8448:~$ sudo -i

➜ root@cka8448:~# ps aux | grep kubelet
```
We can see which components are controlled via systemd looking at /usr/lib/systemd directory:
```bash
➜ root@cka8448:~# find /usr/lib/systemd | grep kube
/usr/lib/systemd/user/podman-kube@.service
/usr/lib/systemd/system/kubelet.service.d
/usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
/usr/lib/systemd/system/kubelet.service
/usr/lib/systemd/system/podman-kube@.service

➜ root@cka8448:~# service kubelet status
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; preset: enabled)
    Drop-In: /usr/lib/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: active (running) since Sun 2024-12-08 16:10:53 UTC; 1h 6min ago
       Docs: https://kubernetes.io/docs/
   Main PID: 7355 (kubelet)
      Tasks: 11 (limit: 1317)
     Memory: 69.0M (peak: 75.9M)
        CPU: 1min 58.582s
     CGroup: /system.slice/kubelet.service
             └─7355 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet>
...

➜ root@cka8448:~# find /usr/lib/systemd | grep etcd
This shows kubelet is controlled via systemd, but no other service named kube nor etcd. It seems that this cluster has been setup using kubeadm, so we check in the default manifests directory:

➜ root@cka8448:~# find /etc/kubernetes/manifests/
/etc/kubernetes/manifests/
/etc/kubernetes/manifests/kube-controller-manager.yaml
/etc/kubernetes/manifests/etcd.yaml
/etc/kubernetes/manifests/kube-apiserver.yaml
/etc/kubernetes/manifests/kube-scheduler.yaml
The kubelet could also have a different manifests directory specified via a KubeletConfiguration, but the one above is the default one.
```

This means the main 4 controlplane services are setup as static Pods. Actually, let's check all Pods running on in the kube-system Namespace:
```bash
➜ root@cka8448:~# k -n kube-system get pod -o wide
NAME                              ...   NODE      
coredns-6f8b9d9f4b-8z7rb          ...   cka8448   
coredns-6f8b9d9f4b-fg7bt          ...   cka8448   
etcd-cka8448                      ...   cka8448   
kube-apiserver-cka8448            ...   cka8448   
kube-controller-manager-cka8448   ...   cka8448   
kube-proxy-dvv7m                  ...   cka8448   
kube-scheduler-cka8448            ...   cka8448   
weave-net-gjrxh                   ...   cka8448
```
Above we see the 4 static pods, with -cka8448 as suffix.

We also see that the dns application seems to be coredns, but how is it controlled?
```bash
➜ root@cka8448$ kubectl -n kube-system get ds
NAME         DESIRED   ...   NODE SELECTOR            AGE
kube-proxy   1         ...   kubernetes.io/os=linux   67m
weave-net    1         ...   <none>                   67m

➜ root@cka8448$ k -n kube-system get deploy
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
coredns   2/2     2            2           68m
```
Seems like coredns is controlled via a Deployment. We combine our findings in the requested file:

```bash
# /opt/course/8/controlplane-components.txt
kubelet: process
kube-apiserver: static-pod
kube-scheduler: static-pod
kube-controller-manager: static-pod
etcd: static-pod
dns: pod coredns
You should be comfortable investigating a running cluster, know different methods on how a cluster and its services can be setup and be able to troubleshoot and find error sources.
```

---

## Question 9 | Kill Scheduler, Manual Scheduling
 
Solve this question on: `ssh cka5248`

Temporarily stop the kube-scheduler, this means in a way that you can start it again afterwards.
- Create a single Pod named manual-schedule of image `httpd:2-alpine`, confirm it's created but not scheduled on any node.
- Now you're the scheduler and have all its power, manually schedule that Pod on node `cka5248`. Make sure it's running.
- Start the `kube-scheduler` again and confirm it's running correctly by creating a second Pod named `manual-schedule2` of image `httpd:2-alpine` and check if it's running on `cka5248-node1`.

**Answer**:
Stop the Scheduler
First we find the controlplane node:
```bash
➜ candidate@cka5248:~$ k get node
NAME            STATUS   ROLES           AGE     VERSION
cka5248         Ready    control-plane   6d22h   v1.33.1
cka5248-node1   Ready    <none>          6d22h   v1.33.1
```
Then we connect and check if the scheduler is running:

```bash
➜ candidate@cka5248:~$ sudo -i

➜ root@cka5248:~# kubectl -n kube-system get pod | grep schedule
kube-scheduler-cka5248            1/1     Running   0               6d22h
```
Kill the Scheduler (temporarily):

```bash
➜ root@cka5248:~# cd /etc/kubernetes/manifests/

➜ root@cka5248:~# mv kube-scheduler.yaml ..
```
And it should be stopped, we can wait for the container to be removed with watch crictl ps:
```bash
➜ root@cka5248:/etc/kubernetes/manifests# watch crictl ps

➜ root@cka5248:/etc/kubernetes/manifests# kubectl -n kube-system get pod | grep schedule

➜ root@cka5248:/etc/kubernetes/manifests
```
ℹ️ In this environment crictl can be used for container management. In the real exam this could also be docker. Both commands can be used with the same arguments.
 

Create a Pod
Now we create the Pod:
```bash
➜ root@cka5248:~# k run manual-schedule --image=httpd:2-alpine
pod/manual-schedule created
```
And confirm it has no node assigned:
```bash
➜ root@cka5248:~# k get pod manual-schedule -o wide
NAME              READY   STATUS    RESTARTS   AGE   IP       NODE    ...
manual-schedule   0/1     Pending   0          14s   <none>   <none>  ...
 
```
Manually schedule the Pod
Let's play the scheduler now:
```bash
➜ root@cka5248:~# k get pod manual-schedule -o yaml > 9.yaml
# cka5248:/root/9.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2020-09-04T15:51:02Z"
  labels:
    run: manual-schedule
  managedFields:
...
    manager: kubectl-run
    operation: Update
    time: "2020-09-04T15:51:02Z"
  name: manual-schedule
  namespace: default
  resourceVersion: "3515"
  selfLink: /api/v1/namespaces/default/pods/manual-schedule
  uid: 8e9d2532-4779-4e63-b5af-feb82c74a935
spec:
  nodeName: cka5248       # ADD the controlplane node name
  containers:
  - image: httpd:2-alpine
    imagePullPolicy: IfNotPresent
    name: manual-schedule
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-nxnc7
      readOnly: true
  dnsPolicy: ClusterFirst
```

The scheduler sets the nodeName for a Pod declaration. How it finds the correct node to schedule on, that's a very much complicated matter and takes many variables into account.

As we cannot kubectl apply or kubectl edit , in this case we need to delete and create or replace:
```bash
➜ root@cka5248:~# k -f 9.yaml replace --force
How does it look?

➜ root@cka5248:~# k get pod manual-schedule -o wide
NAME              READY   STATUS    ...   NODE            
manual-schedule   1/1     Running   ...   cka5248
```
It looks like our Pod is running on the controlplane now as requested, although no tolerations were specified. Only the scheduler takes taints/tolerations/affinity into account when finding the correct node name. That's why it's still possible to assign Pods manually directly to a controlplane node and skip the scheduler.

 

Start the scheduler again
```bash
➜ root@cka5248:~# cd /etc/kubernetes/manifests/

➜ root@cka5248:/etc/kubernetes/manifests# mv ../kube-scheduler.yaml .
Checks it's running:

➜ root@cka5248:~# kubectl -n kube-system get pod | grep schedule
kube-scheduler-cka5248            1/1     Running   0               13s
Schedule a second test Pod:

➜ root@cka5248:~# k run manual-schedule2 --image=httpd:2-alpine

➜ root@cka5248:~# k get pod -o wide | grep schedule
manual-schedule    1/1     Running   0          95s   10.32.0.2   cka5248
manual-schedule2   1/1     Running   0          9s    10.44.0.3   cka5248-node1
Back to normal.
 ```

 ----

## Question 10 | PV PVC Dynamic Provisioning
 
Solve this question on: ssh cka6016

There is a backup Job which needs to be adjusted to use a PVC to store backups.
- Create a StorageClass named `local-backup` which uses `provisioner: rancher.io/local-path` and `volumeBindingMode: WaitForFirstConsumer`.
- To prevent possible data loss the StorageClass should keep a PV retained even if a bound PVC is deleted.
- Adjust the Job at `/opt/course/10/backup.yaml` to use a PVC which request `50Mi` storage and uses the new StorageClass.
Deploy your changes, verify the Job completed once and the PVC was bound to a newly created PV.

**Answer**:
The StorageClass should use provider `rancher.io/local-path`, which is of the project Local Path Provisioner. This project works with Dynamic Volume Provisioning, but instead of creating actual volumes it uses local storage on the node where the Pod runs, by default at path `/opt/local-path-provisioner`.

Cloud companies like AWS or GCP provide their own StorageClasses and providers, which if used for PVCs create PVs backed by actual volumes in the cloud account. 

Create StorageClass
First we can have a look at existing ones:

```bash
➜ ssh cka6016

➜ candidate@cka6016:~$ k get sc
NAME         PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE       ...
local-path   rancher.io/local-path   Delete          WaitForFirstConsumer    ...
```
The `local-path` is the default one available if the Local Path Provisioner is installed. But we can see it has a reclaimPolicy of Delete. Still we could use this one as template for the one we need to create:

```bash
➜ candidate@cka6016:~$ vim sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-backup
provisioner: rancher.io/local-path
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
```
We need to use reclaimPolicy: Retain because this will cause the PV to not get deleted even after the associated PVC is deleted. It's very easy to delete resources in Kubernetes which can lead to quick data loss. Especially in this case where important data, like from a backup, is in play.

```bash
➜ candidate@cka6016:~$ k -f sc.yaml apply
storageclass.storage.k8s.io/local-backup created

➜ candidate@cka6016:~$ k get sc
NAME           PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ...
local-backup   rancher.io/local-path   Retain          WaitForFirstConsumer   ...
local-path     rancher.io/local-path   Delete          WaitForFirstConsumer   ...
```
This looks like what we want. Now we have the choice between two StorageClasses.

 

Check existing Job
Let's have a look at the existing Job:
```bash
# cka6016:/opt/course/10/backup.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: backup
  namespace: project-bern
spec:
  backoffLimit: 0
  template:
    spec:
      volumes:
        - name: backup
          emptyDir: {}
      containers:
        - name: bash
          image: bash:5
          command:
            - bash
            - -c
            - |
              set -x
              touch /backup/backup-$(date +%Y-%m-%d-%H-%M-%S).tar.gz
              sleep 15
          volumeMounts:
            - name: backup
              mountPath: /backup
      restartPolicy: Never
```
Currently it uses an emptyDir volume which means in only stores data in the temporary filesystem of the Pod. This means once the Pod is deleted the data is deleted as well.

We could go ahead and create it now to see if everything else works:
```bash
➜ candidate@cka6016:~$ k -f /opt/course/10/backup.yaml apply
job.batch/backup created

➜ candidate@cka6016:~$ k -n project-bern get job,pod
NAME               STATUS     COMPLETIONS   DURATION   AGE
job.batch/backup   Complete   1/1           5s         11s

NAME               READY   STATUS      RESTARTS   AGE
pod/backup-pll27   0/1     Completed   0          21s
```
Looks like it completed without errors.

 

Adjust Job template
For this we first need to create a PVC and then use in the Job template:

```bash
➜ candidate@cka6016:~$ cd /opt/course/10

➜ candidate@cka6016:/opt/course/10$ cp backup.yaml backup.yaml_ori

➜ candidate@cka6016:/opt/course/10$ vim backup.yaml
# cka6016:/opt/course/10/backup.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: backup-pvc
  namespace: project-bern            # use same Namespace
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Mi                  # request the required size
  storageClassName: local-backup     # use the new StorageClass
---
apiVersion: batch/v1
kind: Job
metadata:
  name: backup
  namespace: project-bern
spec:
  backoffLimit: 0
  template:
    spec:
      volumes:
        - name: backup
          persistentVolumeClaim:     # CHANGE
            claimName: backup-pvc    # CHANGE
      containers:
        - name: bash
          image: bash:5
          command:
            - bash
            - -c
            - |
              set -x
              touch /backup/backup-$(date +%Y-%m-%d-%H-%M-%S).tar.gz
              sleep 15
          volumeMounts:
            - name: backup
              mountPath: /backup
      restartPolicy: Never
```
We first made a backup of the provided file, which is always a good idea. Then we added the new PVC and referenced the PVC in the Pod volumes: section.


Deploy changes and verify
First we delete the existing Job because we did create it once before without any changes. And then we deploy:
```bash
➜ candidate@cka6016:/opt/course/10$ k delete -f backup.yaml 
job.batch "backup" deleted

➜ candidate@cka6016:/opt/course/10$ k apply -f backup.yaml 
persistentvolumeclaim/backup-pvc created
job.batch/backup created
Then we should see the Job execution created a Pod which used the PVC which created a PV:

➜ candidate@cka6016:/opt/course/10$ k -n project-bern get job,pod,pvc,pv
NAME               STATUS    COMPLETIONS   DURATION   AGE
job.batch/backup   Running   0/1           13s        13s

NAME               READY   STATUS    RESTARTS   AGE
pod/backup-q7dgx   1/1     Running   0          13s

NAME         STATUS   VOLUME                                     CAPACITY   ...
backup-pvc   Bound    pvc-dbccec94-cc31-4e30-b5fe-7cb42a85fe7a   50Mi       ...

NAME          CAPACITY   ...  RECLAIM POLICY  STATUS  CLAIM                     ...
pvc-dbcce...  50Mi       ...  Retain          Bound   project-bern/backup-pvc   ...
 ```

Optional investigation
Because the Local Path Provisioner is used we can actually see the volume represented on the filesystem. And because this cluster only has one node, and we're already on it, we can simply do:

```bash
➜ candidate@cka6016:~$ find /opt/local-path-provisioner
/opt/local-path-provisioner/
/opt/local-path-provisioner/pvc-dbccec94-cc31-4e30-b5fe-7cb42a85fe7a_project-bern_backup-pvc
/opt/local-path-provisioner/pvc-dbccec94-cc31-4e30-b5fe-7cb42a85fe7a_project-bern_backup-pvc/backup-2024-12-30-17-27-51.tar.gz
```
If we run the Job again we should see another backup file:
```bash
➜ candidate@cka6016:~$ k -n project-bern delete job backup 
job.batch "backup" deleted

➜ candidate@cka6016:~$ k apply -f backup.yaml 
persistentvolumeclaim/backup-pvc unchanged
job.batch/backup created

➜ candidate@cka6016:~$ k -n project-bern get job,pod,pvc,pv
NAME               STATUS     COMPLETIONS   DURATION   AGE
job.batch/backup   Complete   1/1           18s        20s

NAME               READY   STATUS      RESTARTS   AGE
pod/backup-jpq2t   0/1     Completed   0          20s

➜ candidate@cka6016:~$ find /opt/local-path-provisioner
/opt/local-path-provisioner/
/opt/local-path-provisioner/pvc-dbccec94-cc31-4e30-b5fe-7cb42a85fe7a_project-bern_backup-pvc
/opt/local-path-provisioner/pvc-dbccec94-cc31-4e30-b5fe-7cb42a85fe7a_project-bern_backup-pvc/backup-2024-12-30-17-27-51.tar.gz
/opt/local-path-provisioner/pvc-dbccec94-cc31-4e30-b5fe-7cb42a85fe7a_project-bern_backup-pvc/backup-2024-12-30-17-34-26.tar.gz
```
And if we delete the PVC we should still see the PV and the files in the volume (filesystem in this case):

ℹ️ Removing the PVC and Job might affect your scoring for this question, so best create them again after testing deletion

```bash
➜ candidate@cka6016:~$ k -n project-bern delete pvc backup-pvc 
persistentvolumeclaim "backup-pvc" deleted

➜ candidate@cka6016:~$ k get pv,pvc -A
NAME          CAPACITY   ...  RECLAIM POLICY   STATUS     CLAIM                     ...
pvc-dbcce...  50Mi       ...  Retain           Released   project-bern/backup-pvc   ...
```
We can no longer see the PVC, but the PV is in status Released. This is because we set the reclaimPolicy: Retain in the StorageClass. Now we could manually export/rescue the data in the volume and afterwards delete the PV manually.

 --- 

## Question 11 | Create Secret and mount into Pod

- Create Namespace secret and implement the following in it:
- Create Pod `secret-pod` with image `busybox:1`. It should be kept running by executing sleep 1d or something similar
- Create the existing Secret `/opt/course/11/secret1.yaml` and mount it readonly into the Pod at `/tmp/secret1`
- Create a new Secret called `secret2` which should contain `user=user1` and `pass=1234`. These entries should be available inside the Pod's container as environment variables `APP_USER` and `APP_PASS`

**Answer**
 
#### Secret 1
To create the existing Secret we need to adjust the Namespace for it:

```bash
k create ns secret
namespace/secret created

➜ candidate@cka2560:~$ cp /opt/course/11/secret1.yaml 11_secret1.yaml
# cka2560:/home/candidate/11_secret1.yaml
apiVersion: v1
data:
  halt: IyEgL2Jpbi9zaAo...
kind: Secret
metadata:
  creationTimestamp: null
  name: secret1
  namespace: secret           # UPDATE
➜ candidate@cka2560:~$ k -f 11_secret1.yaml create
secret/secret1 created
 ```

#### Secret 2
Next we create the second Secret:

```bash
➜ candidate@cka2560:~$ k -n secret create secret generic secret2 --from-literal=user=user1 --from-literal=pass=1234
secret/secret2 created
 ```

Pod
Now we create the Pod template:

```yaml
➜ candidate@cka2560:~$ k -n secret run secret-pod --image=busybox:1 --dry-run=client -o yaml -- sh -c "sleep 1d" > 11.yaml
Then make the necessary changes:

# cka2560:/home/candidate/11.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: secret-pod
  name: secret-pod
  namespace: secret                       # important if not automatically added
spec:
  containers:
  - args:
    - sh
    - -c
    - sleep 1d
    image: busybox:1
    name: secret-pod
    resources: {}
    env:                                  # add
    - name: APP_USER                      # add
      valueFrom:                          # add
        secretKeyRef:                     # add
          name: secret2                   # add
          key: user                       # add
    - name: APP_PASS                      # add
      valueFrom:                          # add
        secretKeyRef:                     # add
          name: secret2                   # add
          key: pass                       # add
    volumeMounts:                         # add
    - name: secret1                       # add
      mountPath: /tmp/secret1             # add
      readOnly: true                      # add
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  volumes:                                # add
  - name: secret1                         # add
    secret:                               # add
      secretName: secret1                 # add
status: {}
And execute:

➜ candidate@cka2560:~$ k -f 11.yaml create
pod/secret-pod created
```
Finally we verify:

```bash
➜ candidate@cka2560:~$ k -n secret exec secret-pod -- env | grep APP
APP_PASS=1234
APP_USER=user1

➜ candidate@cka2560:~$ k -n secret exec secret-pod -- find /tmp/secret1
/tmp/secret1
/tmp/secret1/..data
/tmp/secret1/halt
/tmp/secret1/..2019_12_08_12_15_39.463036797
/tmp/secret1/..2019_12_08_12_15_39.463036797/halt

➜ candidate@cka2560:~$ k -n secret exec secret-pod -- cat /tmp/secret1/halt
#! /bin/sh
### BEGIN INIT INFO
# Provides:          halt
# Required-Start:
# Required-Stop:
# Default-Start:
# Default-Stop:      0
# Short-Description: Execute the halt command.
# Description:
```
 
---

## Question 12 | Schedule Pod on Controlplane Nodes
 
Solve this question on: ssh cka5248

- Create a Pod of image `httpd:2-alpine` in Namespace `default`.
- The Pod should be named `pod1` and the container should be named `pod1-container`.
- This Pod should only be scheduled on controlplane nodes.
- Do not add new labels to any nodes.

**Answer**:
First we find the controlplane node(s) and their taints:

```bash
➜ ssh cka5248

➜ candidate@cka5248:~$ k get node
NAME            STATUS   ROLES           AGE   VERSION
cka5248         Ready    control-plane   90m   v1.33.1
cka5248-node1   Ready    <none>          85m   v1.33.1

➜ candidate@cka5248:~$ k describe node cka5248 | grep Taint -A1
Taints:             node-role.kubernetes.io/control-plane:NoSchedule
Unschedulable:      false

➜ candidate@cka5248:~$ k get node cka5248 --show-labels
NAME      STATUS   ROLES           AGE   VERSION   LABELS
cka5248   Ready    control-plane   91m   v1.33.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=cka5248,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
```

Next we create the Pod yaml: 

Solution using NodeSelector
Use the K8s docs and search for tolerations and nodeSelector to find examples, then update:
```yaml
➜ candidate@cka5248:~$ k run pod1 --image=httpd:2-alpine --dry-run=client -o yaml > 12.yaml

# cka5248:/home/candidate/12.yaml

apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: pod1
  name: pod1
spec:
  containers:
  - image: httpd:2-alpine
    name: pod1-container                       # change
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  tolerations:                                 # add
  - effect: NoSchedule                         # add
    key: node-role.kubernetes.io/control-plane # add
  nodeSelector:                                # add
    node-role.kubernetes.io/control-plane: ""  # add
status: {}
```
ℹ️ The nodeSelector specifies `node-role.kubernetes.io/control-plane` with no value because this is a key-only label and we want to match regardless of the value

Important here to add the toleration for running on controlplane nodes, but also the nodeSelector to make sure it only runs on controlplane nodes. 

If we just specify a toleration the Pod can be scheduled on controlplane or worker nodes.
 
#### Solution using NodeAffinity
We could also use nodeAffinity instead of nodeSelector, although in this case it is more complex and not really suggested:
```bash
# cka5248:/home/candidate/12.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: pod1
  name: pod1
spec:
  containers:
  - image: httpd:2-alpine
    name: pod1-container                       # change
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  tolerations:                                 # add
  - effect: NoSchedule                         # add
    key: node-role.kubernetes.io/control-plane # add
  affinity:                                            # add
    nodeAffinity:                                      # add
      requiredDuringSchedulingIgnoredDuringExecution:  # add
        nodeSelectorTerms:                             # add
        - matchExpressions:                            # add
          - key: node-role.kubernetes.io/control-plane # add
            operator: Exists                           # add
status: {}
```
Using nodeAffinity still requires the toleration.

 

Verify
Now we create the Pod and and check if is scheduled:

```bash
➜ candidate@cka5248:~$ k -f 12.yaml create
pod/pod1 created

➜ candidate@cka5248:~$ k get pod pod1 -o wide
NAME   READY   STATUS    ...   NODE      NOMINATED NODE   READINESS GATES
pod1   1/1     Running   ...   cka5248   <none>           <none>
```

We can see the Pod is scheduled on the controlplane node.

 ---

## Question 13 | Multi Containers and Pod shared Volume

Solve this question on: ssh cka3200

Create a Pod with multiple containers named `multi-container-playground` in Namespace `default`:

It should have a volume attached and mounted into each container. The volume shouldn't be persisted or shared with other Pods
- Container `c1` with image `nginx:1-alpine` should have the name of the node where its Pod is running on available as environment variable `MY_NODE_NAME`
- Container `c2` with image `busybox:1` should write the output of the date command every second in the shared volume into file `date.log`. You can use `while true; do date >> /your/vol/path/date.log; sleep 1; done` for this.
- Container `c3` with image `busybox:1` should constantly write the content of file `date.log` from the shared volume to stdout. You can use `tail -f /your/vol/path/date.log` for this.

ℹ️ Check the logs of container c3 to confirm correct setup

**Answer**:
First we create the Pod template:
```yaml
➜ ssh cka3200

k run multi-container-playground --image=nginx:1-alpine --dry-run=client -o yaml > 13.yaml

# cka3200:/home/candidate/13.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: multi-container-playground
  name: multi-container-playground
spec:
  containers:
  - image: nginx:1-alpine
    name: c1                                                                      # change
    resources: {}
    env:                                                                          # add
    - name: MY_NODE_NAME                                                          # add
      valueFrom:                                                                  # add
        fieldRef:                                                                 # add
          fieldPath: spec.nodeName                                                # add
    volumeMounts:                                                                 # add
    - name: vol                                                                   # add
      mountPath: /vol                                                             # add
  - image: busybox:1                                                              # add
    name: c2                                                                      # add
    command: ["sh", "-c", "while true; do date >> /vol/date.log; sleep 1; done"]  # add
    volumeMounts:                                                                 # add
    - name: vol                                                                   # add
      mountPath: /vol                                                             # add
  - image: busybox:1                                                              # add
    name: c3                                                                      # add
    command: ["sh", "-c", "tail -f /vol/date.log"]                                # add
    volumeMounts:                                                                 # add
    - name: vol                                                                   # add
      mountPath: /vol                                                             # add
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  volumes:                                                                        # add
    - name: vol                                                                   # add
      emptyDir: {}                                                                # add
status: {}
```
Well, there was a lot requested here! We check if everything is good with the Pod:

```bash
➜ candidate@cka3200:~$ k -f 13.yaml create
pod/multi-container-playground created

➜ candidate@cka3200:~$ k get pod multi-container-playground
NAME                         READY   STATUS    RESTARTS   AGE
multi-container-playground   3/3     Running   0          47s
```
Not a bad start. Now we check if container c1 has the requested node name as env variable:

```bash
➜ candidate@cka3200:~$ k exec multi-container-playground -c c1 -- env | grep MY
MY_NODE_NAME=cka3200
```
And finally we check the logging, which means that c2 correctly writes and c3 correctly reads and outputs to stdout:

```bash
➜ candidate@cka3200:~$ k logs multi-container-playground -c c3
Tue Nov  5 13:41:33 UTC 2024
Tue Nov  5 13:41:34 UTC 2024
Tue Nov  5 13:41:35 UTC 2024
Tue Nov  5 13:41:36 UTC 2024
Tue Nov  5 13:41:37 UTC 2024
Tue Nov  5 13:41:38 UTC 2024
```

---
 
## Question 14 | Find out Cluster Information
 
Solve this question on: ssh cka8448

You're ask to find out following information about the cluster:

- How many controlplane nodes are available?
- How many worker nodes (non controlplane nodes) are available?
- What is the Service CIDR?
- Which Networking (or CNI Plugin) is configured and where is its config file?
- Which suffix will static pods have that run on cka8448?
- Write your answers into file `/opt/course/14/cluster-info`, structured like this:

```bash
 /opt/course/14/cluster-info
1: [ANSWER]
2: [ANSWER]
3: [ANSWER]
4: [ANSWER]
5: [ANSWER]
 ```

Answer:
How many controlplane and worker nodes are available?
```
➜ ssh cka8448

➜ candidate@cka8448:~$ k get node
NAME      STATUS   ROLES           AGE   VERSION
cka8448   Ready    control-plane   71m   v1.33.1
```
We see one controlplane and no worker nodes.

 

What is the Service CIDR?
```
➜ candidate@cka8448:~$ sudo -i

➜ root@cka8448:~# cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep range
    - --service-cluster-ip-range=10.96.0.0/12
 ```

Which Networking (or CNI Plugin) is configured and where is its config file?
```bash
➜ root@cka8448:~# find /etc/cni/net.d/
/etc/cni/net.d/
/etc/cni/net.d/.kubernetes-cni-keep
/etc/cni/net.d/10-weave.conflist
/etc/cni/net.d/87-podman-bridge.conflist

➜ root@cka8448:~# cat /etc/cni/net.d/10-weave.conflist
{
    "cniVersion": "0.3.0",
    "name": "weave",
    "plugins": [
        {
            "name": "weave",
            "type": "weave-net",
            "hairpinMode": true
        },
        {
            "type": "portmap",
            "capabilities": {"portMappings": true},
            "snat": true
        }
    ]
}
```
By default the kubelet looks into `/etc/cni/net.d` to discover the CNI plugins. This will be the same on every controlplane and worker nodes.

 
Which suffix will static pods have that run on cka8448?
`The suffix is the node hostname with a leading hyphen.`


**Result**
The resulting `/opt/course/14/cluster-info` could look like:

```bash
# /opt/course/14/cluster-info

# How many controlplane nodes are available?
1: 1

# How many worker nodes (non controlplane nodes) are available?
2: 0

# What is the Service CIDR?
3: 10.96.0.0/12

# Which Networking (or CNI Plugin) is configured and where is its config file?
4: Weave, /etc/cni/net.d/10-weave.conflist

# Which suffix will static pods have that run on cka8448?
5: -cka8448
```
 
---

## Question 15 | Cluster Event Logging
 
Solve this question on: ssh cka6016

- Write a kubectl command into `/opt/course/15/cluster_events.sh` which shows the latest events in the whole cluster, ordered by time (`metadata.creationTimestamp`)
- Delete the kube-proxy Pod and write the events this caused into `/opt/course/15/pod_kill.log` on cka6016
- Manually kill the containerd container of the kube-proxy Pod and write the events into `/opt/course/15/container_kill.log`

**Answer**:
 

### Step 1

```bash
➜ candidate@cka6016:~$ vim /opt/course/15/cluster_events.sh

kubectl get events -A --sort-by=.metadata.creationTimestamp
```
And we can execute it which should show recent events:

```
➜ candidate@cka6016:~$ sh /opt/course/15/cluster_events.sh
NAMESPACE     LAST SEEN   TYPE     REASON           OBJECT                       MESSAGE
...
32.0.2:8181: connect: connection refused
default       19m         Normal    Pulled              pod/team-york-board-7d74f8f86c-fvzw5    Successfully pulled image "httpd:2-alpine" in 4.574s (4.575s including waiting). Image size: 22038396 bytes.
default       19m         Normal    Created             pod/team-york-board-7d74f8f86c-fvzw5    Created container httpd
default       19m         Normal    Pulled              pod/team-york-board-7d74f8f86c-9fg47    Successfully pulled image "httpd:2-alpine" in 425ms (4.976s including waiting). Image size: 22038396 bytes.
default       19m         Normal    Started             pod/team-york-board-7d74f8f86c-fvzw5    Started container httpd
default       19m         Normal    Pulled              pod/team-york-board-7d74f8f86c-xnprt    Successfully pulled image "httpd:2-alpine" in 711ms (5.685s including waiting). Image size: 22038396 bytes.
default       19m         Normal    Created             pod/team-york-board-7d74f8f86c-xnprt    Created container httpd
default       19m         Normal    Created             pod/team-york-board-7d74f8f86c-9fg47    Created container httpd
default       19m         Normal    Started             pod/team-york-board-7d74f8f86c-9fg47    Started container httpd
default       19m         Normal    Started             pod/team-york-board-7d74f8f86c-xnprt    Started container httpd
...
```

### Step 2
We delete the kube-proxy Pod:
```
➜ candidate@cka6016:~$ k -n kube-system get pod -l k8s-app=kube-proxy -owide
NAME               READY   ...     NODE      NOMINATED NODE   READINESS GATES
kube-proxy-lf2fs   1/1     ...     cka6016   <none>           <none>

➜ candidate@cka6016:~$ k -n kube-system delete pod kube-proxy-lf2fs
pod "kube-proxy-lf2fs" deleted
```
Now we can check the events, for example by using the command that we created before:

```
➜ candidate@cka6016:~$ sh /opt/course/15/cluster_events.sh
```
Write the events caused by the deletion into `/opt/course/15/pod_kill.log` on cka6016:
```
# cka6016:/opt/course/15/pod_kill.log
kube-system   12s         Normal    Killing             pod/kube-proxy-lf2fs                    Stopping container kube-proxy
kube-system   12s         Normal    SuccessfulCreate    daemonset/kube-proxy                    Created pod: kube-proxy-wb4tb
kube-system   11s         Normal    Scheduled           pod/kube-proxy-wb4tb                    Successfully assigned kube-system/kube-proxy-wb4tb to cka6016
kube-system   11s         Normal    Pulled              pod/kube-proxy-wb4tb                    Container image "registry.k8s.io/kube-proxy:v1.33.1" already present on machine
kube-system   11s         Normal    Created             pod/kube-proxy-wb4tb                    Created container kube-proxy
kube-system   11s         Normal    Started             pod/kube-proxy-wb4tb                    Started container kube-proxy
default       10s         Normal    Starting            node/cka6016  
 ```

### Step 3
ℹ️ Node cka6016 is already the controlplane and the only node of the cluster. Otherwise we might have to ssh onto the correct worker node where the Pod is running instead

 
Finally we will try to provoke events by killing the container belonging to the container of a `kube-proxy` Pod:

```
➜ candidate@cka6016:~$ sudo -i

➜ root@cka6016:~# crictl ps | grep kube-proxy
2fd052f1fcf78       505d571f5fd56       57 seconds ago      Running             kube-proxy                0                   3455856e0970c       kube-proxy-wb4tb

➜ root@cka6016:~# crictl rm --force 2fd052f1fcf78
2fd052f1fcf78
2fd052f1fcf78

➜ root@cka6016:~# crictl ps | grep kube-proxy
6bee4f36f8410       505d571f5fd56       5 seconds ago       Running             kube-proxy                0                   3455856e0970c       kube-proxy-wb4tb
```
ℹ️ In this environment crictl can be used for container management. In the real exam this could also be docker. Both commands can be used with the same arguments.

We killed the container (2fd052f1fcf78), but also noticed that a new container (6bee4f36f8410) was directly created again. Thanks Kubernetes!

Now we see if this caused events again and we write those into the second file:

```bash
➜ candidate@cka6016:~$ sh /opt/course/15/cluster_events.sh
Write the events caused by the killing into /opt/course/15/container_kill.log on cka6016:

# /opt/course/15/container_kill.log
kube-system   21s         Normal    Created             pod/kube-proxy-wb4tb                    Created container kube-proxy
kube-system   21s         Normal    Started             pod/kube-proxy-wb4tb                    Started container kube-proxy
default       90s         Normal    Starting            node/cka6016                            
default       20s         Normal    Starting            node/cka6016
```     
Comparing the events we see that when we deleted the whole Pod there were more things to be done, hence more events. For example was the DaemonSet in the game to re-create the missing Pod. Where when we manually killed the main container of the Pod, the Pod still exists but only its container needed to be re-created, hence less events.

---
 
## Question 16 | Namespaces and Api Resources

Write the names of all namespaced Kubernetes resources (like Pod, Secret, ConfigMap...) into `/opt/course/16/resources.txt`.

Find the `project-*` Namespace with the highest number of Roles defined in it and write its name and amount of Roles into `/opt/course/16/crowded-namespace.txt`.

**Answer**:
Namespace and Namespaces Resources
We can get a list of all resources:

```bash
➜ ssh cka3200

➜ candidate@cka3200:~$ k api-resources --namespaced -o name > /opt/course/16/resources.txt
```
Which results in the file:

```bash
cka3200:/opt/course/16/resources.txt
bindings
configmaps
endpoints
events
limitranges
persistentvolumeclaims
pods
podtemplates
replicationcontrollers
resourcequotas
secrets
serviceaccounts
services
controllerrevisions.apps
daemonsets.apps
deployments.apps
replicasets.apps
statefulsets.apps
localsubjectaccessreviews.authorization.k8s.io
horizontalpodautoscalers.autoscaling
cronjobs.batch
jobs.batch
leases.coordination.k8s.io
endpointslices.discovery.k8s.io
events.events.k8s.io
ingresses.networking.k8s.io
networkpolicies.networking.k8s.io
poddisruptionbudgets.policy
rolebindings.rbac.authorization.k8s.io
roles.rbac.authorization.k8s.io
csistoragecapacities.storage.k8s.io
 ```

Namespace with most Roles
```bash
➜ candidate@cka3200:~$ k -n project-jinan get role --no-headers | wc -l
No resources found in project-jinan namespace.
0

➜ candidate@cka3200:~$ k -n project-miami get role --no-headers | wc -l
300

➜ candidate@cka3200:~$ k -n project-melbourne get role --no-headers | wc -l
2

➜ candidate@cka3200:~$ k -n project-seoul get role --no-headers | wc -l
10

➜ candidate@cka3200:~$ k -n project-toronto get role --no-headers | wc -l
No resources found in project-toronto namespace.
0
```
Finally we write the name and amount into the file:
```
# cka3200:/opt/course/16/crowded-namespace.txt
project-miami with 300 roles
```
 

 ---
 
## Question 17 | Operator, CRDs, RBAC, Kustomize
 

Solve this question on: ssh cka6016

There is Kustomize config available at `/opt/course/17/operator`. It installs an operator which works with different CRDs. It has been deployed like this:

kubectl kustomize `/opt/course/17/operator/prod | kubectl apply -f -`
Perform the following changes in the Kustomize base config:

The operator needs to list certain CRDs. Check the logs to find out which ones and adjust the permissions for Role `operator-role`

Add a new Student resource called `student4` with any name and description

Deploy your Kustomize config changes to prod.

 
**Answer**:
Kustomize is a standalone tool to manage K8s Yaml files, but it also comes included with kubectl. The common idea is to have a base set of K8s Yaml and then override or extend it for different overlays, like done here for prod:

```bash
➜ ssh cka6016

➜ candidate@cka6016:~$ cd /opt/course/17/operator

➜ candidate@cka6016:/opt/course/17/operator$ ls
base  prod
```

Investigate Base
Let's investigate the base first for better understanding:
```yaml
➜ candidate@cka6016:/opt/course/17/operator$ k kustomize base
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: classes.education.killer.sh
spec:
  group: education.killer.sh
...
---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: students.education.killer.sh
spec:
  group: education.killer.sh
...
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: operator
  namespace: NAMESPACE_REPLACE
...
```
Running kubectl kustomize DIR will build the whole Yaml based on whatever is defined in the `kustomization.yaml`.

In the case above we did build for the base directory, which produces Yaml that is not expected to be deployed just like that. We can see for example that all resources contain `namespace: NAMESPACE_REPLACE` entries which won't be possible to apply because Namespace names need to be lowercase.

But for debugging it can be useful to build the base Yaml.

 

Investigate Prod
```yaml
➜ candidate@cka6016:/opt/course/17/operator$ k kustomize prod
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: classes.education.killer.sh
spec:
  group: education.killer.sh
...
---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: students.education.killer.sh
spec:
  group: education.killer.sh
...
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: operator
  namespace: operator-prod
...
```
We can see that all resources now have `namespace: operator-prod`. Also prod adds the additional label `project_id: prod_7768e94e-88da-4744-9135-f1e7fbb96daf` to the Deployment. The rest is taken from base.

 

Locate Issue
The instructions tell us to check the logs:
```
➜ candidate@cka6016:/opt/course/17/operator$ k -n operator-prod get pod
NAME                        READY   STATUS    RESTARTS   AGE
operator-7f4f58d4d9-v6ftw   1/1     Running   0          6m9s

➜ candidate@cka6016:/opt/course/17/operator$ k -n operator-prod logs operator-7f4f58d4d9-v6ftw
+ true
+ kubectl get students
Error from server (Forbidden): students.education.killer.sh is forbidden: User "system:serviceaccount:operator-prod:operator" cannot list resource "students" in API group "education.killer.sh" in the namespace "operator-prod"
+ kubectl get classes
Error from server (Forbidden): classes.education.killer.sh is forbidden: User "system:serviceaccount:operator-prod:operator" cannot list resource "classes" in API group "education.killer.sh" in the namespace "operator-prod"
+ sleep 10
+ true
```
We can see that the operator tries to list resources students and classes. If we look at the Deployment we can see that it simply runs kubectl commands in a loop:

```yaml
# kubectl -n operator-prod edit deploy operator
apiVersion: apps/v1
kind: Deployment
metadata:
...
  name: operator
  namespace: operator-prod
spec:
...
  template:
...
    spec:
      containers:
      - command: ["/bin/sh","-c"]
        args:
          - |
            set -x
            while true; do
              kubectl get students
              kubectl get classes
              sleep 60
            done
...
 ```

Adjust RBAC
Now we need to adjust the existing Role operator-role. In the Kustomize config directory we find file rbac.yaml which we need to edit. Instead of manually editing the Yaml we could also generate it via command line:

```yaml
➜ candidate@cka6016:/opt/course/17/operator$ k -n operator-prod create role operator-role --verb list --resource student --resource class -oyaml --dry-run=client
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  creationTimestamp: null
  name: operator-role
  namespace: operator-prod
rules:
- apiGroups:
  - education.killer.sh
  resources:
  - students
  - classes
  verbs:
  - list
```
Now we copy&paste it into `rbac.yaml`:

```yaml
➜ candidate@cka6016:/opt/course/17/operator$ vim base/rbac.yaml
# cka6016:/opt/course/17/operator/base/rbac.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: operator-role
  namespace: default
rules:
- apiGroups:
  - education.killer.sh
  resources:
  - students
  - classes
  verbs:
  - list
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: operator-rolebinding
  namespace: default
subjects:
  - kind: ServiceAccount
    name: operator
    namespace: default
roleRef:
  kind: Role
  name: operator-role
  apiGroup: rbac.authorization.k8s.io
```
And we deploy:

```

➜ candidate@cka6016:/opt/course/17/operator$ kubectl kustomize /opt/course/17/operator/prod | kubectl apply -f -
customresourcedefinition.apiextensions.k8s.io/classes.education.killer.sh unchanged
customresourcedefinition.apiextensions.k8s.io/students.education.killer.sh unchanged
serviceaccount/operator unchanged
role.rbac.authorization.k8s.io/operator-role configured
rolebinding.rbac.authorization.k8s.io/operator-rolebinding unchanged
deployment.apps/operator unchanged
class.education.killer.sh/advanced unchanged
student.education.killer.sh/student1 unchanged
student.education.killer.sh/student2 unchanged
student.education.killer.sh/student3 unchanged
```

We can see that only the Role was configured, which is what we want. And the logs are not throwing errors any more:

```
➜ candidate@cka6016:/opt/course/17/operator$ k -n operator-prod logs operator-7f4f58d4d9-v6ftw
+ kubectl get students
NAME       AGE
student1   22m
student2   22m
student3   22m
+ kubectl get classes
NAME       AGE
advanced   20m
 ```

Create new Student resource
Finally we need to create a new Student resource. Here we can simply copy an existing one in `students.yaml`:

```yaml
➜ candidate@cka6016:/opt/course/17/operator$ vim base/students.yaml
# cka6016:/opt/course/17/operator/base/students.yaml
...
apiVersion: education.killer.sh/v1
kind: Student
metadata:
  name: student3
spec:
  name: Carol Williams
  description: A student excelling in container orchestration and management
---
apiVersion: education.killer.sh/v1
kind: Student
metadata:
  name: student4
spec:
  name: Some Name
  description: Some Description
```
And we deploy:

```
➜ candidate@cka6016:/opt/course/17/operator$ kubectl kustomize /opt/course/17/operator/prod | kubectl apply -f -
customresourcedefinition.apiextensions.k8s.io/classes.education.killer.sh unchanged
customresourcedefinition.apiextensions.k8s.io/students.education.killer.sh unchanged
serviceaccount/operator unchanged
role.rbac.authorization.k8s.io/operator-role unchanged
rolebinding.rbac.authorization.k8s.io/operator-rolebinding unchanged
deployment.apps/operator unchanged
class.education.killer.sh/advanced unchanged
student.education.killer.sh/student1 unchanged
student.education.killer.sh/student2 unchanged
student.education.killer.sh/student3 unchanged
student.education.killer.sh/student4 created

➜ candidate@cka6016:/opt/course/17/operator$ k -n operator-prod get student
NAME       AGE
student1   28m
student2   28m
student3   27m
student4   43s
```
Only Student `student4` got created, everything else stayed the same.

