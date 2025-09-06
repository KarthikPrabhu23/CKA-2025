# CKA-2025

## 1. Install cri-dockerd and Set System Parameters

Prepare a Linux system for Kubernetes. Docker is already installed, but you need to configure it for `kubeadm`.

**Tasks:**

- Install the `.deb` package
- Install the Debian package —/cri-dockerd_O.3.9.3-O.ubuntu-jammy_amd64.deb
- Debian packages are installed using dpkg.
- Enable and start the cri-docker service
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

Verify the changes with `sudo sysctl -p`
Troubleshoot:
If the parameter value keeps changing after apply,
Try to load your file at the last, name it with a high number:

`/etc/sysctl.d/99-zcka.conf`

OR
To search for the file inside /etc/sysctl.d/ that contains the string net.ipv4.ip_forward, run:
```
grep -rl "net.ipv4.ip_forward" /etc/sysctl.d/ /usr/lib/sysctl.d/ /etc/sysctl.conf
```

## 2. Verify cert-manager CRD

Create a list of all cert-manager Custom Resource Definitions (CRDs) and save it to ~/resources.yaml.
Make sure kubectl's default output format and use kubectl to list CRD's.
Do not set an output format.
Failure to do so will result in a reduced score.
Using kubectl, extract the documentation for the subject specification field of the Certificate Custom Resource and save it to ~/subject.yaml.
You may use any output format that kubectl supports.

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

An NGINX Deploy named nginx-static is Running in the nginx-static NS. It is configured using a CfgMap named nginx-config. Update the nginx-config CfgMap to allow only TLSv1.3 connections.re-create, restart, or scale resources as necessary. By 1 using command to test the changes: 
curl --tls-max 1.2 https://web.k8s.local
As TLSV1.2 should not be allowed anymore, the command should fail

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
Delete the TLSv1.2 from the CM

```bash
# Edit the configmap.yaml to remove TLSv1.2
kubectl apply -f configmap.yaml
kubectl rollout restart deployment nginx-static -n nginx-static

K get deploy nginx-static -o yaml > deploy.yaml
K delete deploy nginx-static

K create -f deploy.yaml
```

## 4. Create Ingress for Deployment

Create a new ingress resource named echo in echo-sound namespace
With the following tasks:
Expose the deployment with a service named echo-service on http://example.org/echo using Service port 8080 type=NodePort
The availability of Service echo-service can be checked using the following command which should return 200:

**Tasks:**
- Create service named `echo-service` with `type=NodePort` on port 8080
- Create ingress resource named `echo` in namespace `echo-sound`

**Solution:**

```bash
kubectl expose deployment nginx-deploy --name=echo-service --type=NodePort --port=8080 --target-port=8080 -n echo-sound
```
`service/echo-service` should be created.

Copy `ingress.yaml` from documentation
Vim ingress.yaml
Add host: {Hostname}
Add service details, earlier created
Port number : 8080
Path: /

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

## 5. Gateway API Migration
You have an existing web application deployed in a Kubernetes cluster using an Ingress resource named web. You must migrate the existing Ingress configuration to the new Kubernetes Gateway API, maintaining the existing HTTPS access configuration.
Tasks :
- Create a Gateway resource named web-gateway with hostname gateway.web.k8s.local that maintains the existing TLS and listener configuration from the existing Ingress resource named web.
- Create an resource named web-route with hostname gateway.web.k8s.local that maintains the existing routing rules from the current Ingress resource named web.
Note:
	A named nginx-class is already installed in the cluster.

```bash
k get secret
k describe secret web-tls

k describe ingress web

k get svc
```
get `gateway.yaml` from documentation
Replace values from the info from `k describe ingress`
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
      port: 80 // Check from k get svc
```

-------------------------------------
The team from Project r500 wants to replace their Ingress (Networking.k8s.io) with a Gateway Api gateway.networking.k8s.io) solution. The old Ingress is available at `/opt/course/13/ingress.yaml`. Perform the following in Namespace `project-r500` and for the already existing Gateway:
Create a new HTTPRoute named traffic-director which replicates the routes from the old Ingress
Extend the new HTTPRoute with path /auto which redirects to mobile if the User-Agent is exactly mobile and to desktop otherwise

**Tasks:**
- Replace Ingress with Gateway + HTTPRoute in `project-r500`

**Example Route Snippet:**

```yaml
matches:
- path:
    type: PathPrefix
    value: /
  header:
    - type: Exact
      name: User-Agent
      value: Mobile
backendRefs:
- name: my-service
  port: 80
```

## 6. Add Sidecar and Shared Volume

Update the existing deployment wordpress, adding a sidecar container named sidecar using the busybox:stable image to the existing pod.
The new sidecar container has to run the following command : "/bin/sh -c "tail -f /var/log/wordpress.log" use a volume mounted at /var/log to make the log file wordpress.log available to the co-located container

`k edit deployment wordpress`

copy sidecar-deployment.yaml file from the documentation

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
-------------------------------
A legacy app needs to be integrated into the Kubernetes built-in logging architecture (i.e. kubectc logs). Adding a streaming co-located container is a good and common way to accomplish this requirement.
Task
- Update the existing Deployment synergy-deployment, adding a co-located container named sidecar using the bus box:stable image to the existing Pod.
- The new co-located container has. to run the following command: /bin/sh -c "tail -0±1 -f /var/log/synergy-deployment.log"
- Use a Volume mounted at /var/log to make the log file synergy-deployment.log available to the co located container.
- Do not modify the specification of the existing container other than adding the required.
Hint: Use a shared volume to expose the log file between the main application container and the sidecar

**Tasks:**
- Update `synergy-deployment` with sidecar using `busybox:stable`
- Share `/var/log/synergy-deployment.log` via `emptyDir` volume

**Solution:**

Add a container,
EmptyDir Volume, and 2 volumeMounts to each container

- Add volume to pod spec
- Mount `/var/log` in both main and sidecar containers

Step 2: Take a Backup (Recommended)
Before editing the deployment, take a backup of the current spec:

```bash
kubectl get deploy neokloud-deployment -o yaml > /root/deploy-backup.yaml
```

Step 3: Edit the Deployment to Add Sidecar
Edit the deployment using the following command:

```bash
kubectl edit deploy neokloud-deployment
```

Changes to do

Inside the editor:
Add this sidecar container under `spec.template.spec.containers`:
```yaml
- name: sidecar
  image: busybox:stable
  command: ["/bin/sh", "-c", "tail -f /var/log/neokloud.log"]
  volumeMounts:
    - name: shared-logs
      mountPath: /var/log
```  
Add this volumeMounts block to the existing container (e.g., monitor):
```yaml
volumeMounts:
  - name: shared-logs
    mountPath: /var/log
```
Add this volumes block under `spec.template.spec`:
```yaml
volumes:
  - name: shared-logs
    emptyDir: {}
```
Save and exit from the editor.

Step 4: Verify Your Changes

Check if the updated pod is running:
```bash
kubectl get pods
```

Get the exact pod name, then:

```bash
kubectl describe pod <pod-name>
kubectl logs -f <pod-name> -c sidecar
```

You should now see the sidecar container tailing /var/log/neokloud.log.

## 7. PVC + Mount to Deployment

**Tasks:**
- Create a PVC
- Mount it inside existing Deployment

A Persistent Volume already exists and is retained for reuse.
Create a PVC named MariaDB in the mariadb namespace as follows
- Access mode ReadWriteOnce
- Storage capacity 250Mi
Edit the maria-deployment in the file located at maria.deploy.yaml to use the newly created PVC.
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

## 8. ArgoCD Install with Helm (No CRDs)

```bash
helm repo add argo https://github.com/argoproj/argo-helm

helm template argocd argo/argo-cd --version 5.51.6 \ --namespace argo
  --set crds.install=false > argocd.yaml
  
kubectl apply -f argocd.yaml
```

OR 

Step 1: Add Helm Repo
```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
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
kubectl create namespace argocd
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

## 10. Divide Node Resources (With Init Containers)

```bash
K scale deploy wordpress --replicas=0

K describe node node01
```

Check cpu, memory in Allocatable section
Expr memory number / 1024
Check the Memory requests in describe node. Expr the result – (sum of memory request)
Expr result * 0.10 | bc. 
Expr earlier result – new_result
Expr result / 3 will give the allocateable memory for each of the 3 pods
Do it similarly for CPU
The calculation values are the ‘requests’. Limits should be declared slightly higher
K edit deploy wordpress

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

---
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


```bash
k scale deployment wordpress -n namespace --replicas=0
```

```bash
k edit deployment wordpress`
```

Edit inside both the containers
Edit in `spec.template.spec.containers.resources.requests`


```bash
controlplane:~$ kubectl describe node I grep -A5 "Allocatable"
Capacity:
	cpu: 1
	ephemeral-storage: 19221248Ki
	hugepages-2Mi : 0
	memory: 2015360Ki
	pods: 110
```

---

## 11. Least Permissive NetworkPolicy

There are 2 deployments. Frontend in frontend namespace, Backend in backend namespace. Create a Network Policy to have interaction between frontend & backend deployments. 
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
      port: 8080 // check the service
```

## 12. Install CNI: Flannel vs Calico

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

## 12. HPA with Autoscaling v2

Create a new HorizontalPodAutoscaler [HPA] named apache-server in the autoscale namespace.
Tasks :
- This HPA must target the existing deployment called aaachg-deployment in the namespace.
- Set the HPA to target for CPU usage per Pod.
- Configure the HPA to have a minimum of 1 pod and maximum of 4 pods. Have to set the downscale stabilization window to 30 seconds.

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


## 13. Troubleshooting

_(Details not provided)_

## 14. Expose Deployment via NodePort and Fix Port

There is a deployment named `nodeport-deployment` in the relative namespace.
Tasks:
- Configure the deployment so it can be exposed using port 80 and protocol TCP name http.
- Create a new service named `nodeport-service` exposing the container port 80 and TCP
- Configure the new Service to also expose the individual pods using Nodeport.

**Tasks:**
- Edit deployment to expose container port 80
- Create NodePort service using same label

K edit deploy
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

Copy a Nodeport service file from doc. Replace the labels with the above labels.

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


## 15. PriorityClass

You're working in a Kubernetes cluster with an existing Deployment named busybox-logger running in a namespace called priority.
The cluster already has at least one user-defined Priority Class
Perform the following tasks:
1. Create a new Priority Class named high-priority for user workloads. The value of this Priority Class should be exactly one less than the highest existing user-defined Priority Class value.
2. Patch the existing Deployment busybox-logger in the priority namespace to use the newly created high-priority Priority Class

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
```bash
kubectl patch deploy -n priority busybox-logger -p '{"spec":{"template":{"spec":{"priorityClassName":"high-priority"}}}}'
deployment.apps/busybox-logger patched
```
OR edit the deployment:-

In Deployment:
`k edit deploy `

Add the below field in `spec.template.spec`
```
	spec:
	  priorityClassName: high-priority
```

## 16. Kubeconfig Extraction

Youre asked to extract the following information out of kubeconfig file /opt/course/l/kubeconfig on cka9412 :
 Write all kubeconfig context names into /opt/course/l/contexts , one per line
 Write the name of the current context into /opt/course/l/current-context
 Write the client-key of user account—base64-  decoded into /opt/course/l/cert

**Tasks:**

- Extract context names, current context, and client-key (base64 decoded)

**Solution:**

```bash
kubectl config get-contexts --kubeconfig /opt/course/l/kubeconfig -o name > /opt/course/l/contexts

kubectl config current-context --kubeconfig /opt/course/l/kubeconfig > /opt/course/l/current-context
```

```bash
# Extract client-key-data:
cat /opt/course/l/kubeconfig

Copy the client-key-data field 

# Then decode:
echo <client-key-data> | base64 -d > /opt/course/l/cert
```

## 17. QOS Pods (BestEffort)


```bash
kubectl get pods -n project-c13 -o custom-columns="NAME:.metadata.name,QOS:.status.qosClass" | grep "BestEffort" | awk '{print $1}' > /opt/course/4/pods-terminated-first.txt
```

The file will contain the Pod name

## 18. Kustomize HPA Migration

Previously the application api-gateway used some external autoscaler which should now be replaced with a HorizontalPodAutoscaler (HPA). The application has been deployed to Namespaces api-gateway-staging and api-gateway-prod like this:
```
kubectl kustomize /opt/course/5/api—gateway/staging
kubectl kustomize /opt/course/5/api—gateway/prod
```

sign the Kustomize config at /opt/course/5/api-gateway do the following:
Remove the ConfigMap horizontal—scaling—config completely
Add HPA named api-gateway for the Deployment api-gateway with min 2 and max 4 replicas. It should scale at 5096 average CPU utilisation
In prod the HPA should have max 6 replicas
Apply your changes for staging and prod so they're reflected in the cluster

**Tasks:**

- Remove horizontal-scaling-config ConfigMap
- Add new HPA to base
- Patch HPA maxReplicas in prod
- Apply using `kubectl kustomize`


Remove the HPA from ConfigMap in 3 directory: Base, staging, prod
1. Remove the ConfigMap horizontal-scaling-config
	•	Locate and delete the ConfigMap manifest file (likely named horizontal-scaling-config.yaml) in /opt/course/5/api-gateway/base or wherever it is referenced.
	•	Remove its reference from the kustomization.yaml under resources or configMapGenerator.
2. Add HPA for the Deployment
	Create a new HPA manifest in the base
	Create a file /opt/course/5/api-gateway/base/hpa.yaml with the following content:

**Base HPA Example:**

```
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
```

**Prod Patch:**

```
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-gateway
spec:
  maxReplicas: 6
```

4. Update kustomization.yaml Files
In /opt/course/5/api-gateway/base/kustomization.yaml:
Add the HPA resource:

```yaml
resources:
  - deployment.yaml
  # ... other resources
  - hpa.yaml
```  
**Apply Commands:**

```
kubectl apply -k /opt/course/5/api-gateway/staging
kubectl apply -k /opt/course/5/api-gateway/prod
```


## 19. Metrics Server Bash Scripts

The metrics-server has been installed in the cluster. Write two bash scripts which use kubectl :
- Script /opt/course/7/node.sh should show resource usage of Nodes
- Script /opt/course/7/pod.sh should show resource usage of Pods and their containers

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

## 20. Kubernetes Node Upgrade and Join

**Tasks:**
- Upgrade Kubernetes version on worker node `cka3962-node1`
- Join node to cluster using `kubeadm`

**Solution:**

Use documentation to upgrade node version to 1.32

```bash
# SSH into node and follow version upgrade docs
kubeadm token create --print-join-command
```

---

## 21. Use ServiceAccount to Access Secrets via API

There is ServiceAccount secret-reader in Namespace project-swan . Create a Pod of image nginx:1-alpine named api-contact which uses this ServiceAccount. Exec into the Pod and use curl to manually query all Secrets from the Kubernetes Api. Write the result into file /opt/course/9/result.json .

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
kubectl exec -it api-contact -n project-swan -- sh

curl -k https://kubernetes.defautt/api/vl/secrets -H "Authorization: Bearer ${TOKEN}"

TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token) 

curt -k https://kubernetes.defautt/api/vl/secrets -H "Authorization: Bearer ${TOKEN}" > results.json

exit

kubectl exec -it api-contact  –n project-swan – cat results.json > solution.json
```

---

## 22. Create Role and RoleBinding for SA

Create a new ServiceAccount processor in Namespace project-hamster. Create a Role and RoleBinding, both named processor as well. These should allow the new SA to only create Secrets and ConfigMaps in that Namespace.

**Namespace:** `project-hamster`

**Solution:**

```bash
K create sa processor -n project-hamster
```

Vim get `Role` from Doc
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: processor
  namespace: project-hamster
rules:
- apiGroups: [""]
  resources: ["secrets", "configmaps"]
  verbs: ["create"]
```

Vim get `RoleBinding` from Doc
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: processor
  namespace: project-hamster
subjects:
- kind: ServiceAccount
  name: processor
  namespace: project-hamster
roleRef:
  kind: Role
  name: processor
  apiGroup: rbac.authorization.k8s.io
```

---

## 23. MinIO Operator Install with Tenant CRD

Install the MinlO Operator using Helm in Namespace minio . Then configure and create the Tenant CRD:
- Create Namespace minio
- Install Helm chart minio/operator into the new Namespace. The Helm Release should be called minio—operator
- Update the Tenant resource in /opt/course/2/minio-tenant.yaml to include enableSFTP: true under features
- Create the Tenant resource from /opt/course/2/minio-tenant. Yaml It is not required for MinlO to run properly.
Installing the Helm Chart and the Tenant resource as requested is enough

**Namespace:** `minio`

**Solution:**

```bash
helm install minio-operator minio/operator -n minio

vim /opt/course/2/minio-tenant.yaml

# Add under spec.features
enableSFTP: true 

kubectl apply -f /opt/course/2/minio-tenant.yaml
```

---

## 24. DaemonSet with Resource Limits

Use Namespace project-tiger for the following. Create a DaemonSet named ds-important with image httpd:2-aIpine and labels id=ds-important and uuid=18426aØbf59-4e1Ø-923f-cØeØ78e82462 . The Pods it creates should request 10 millicore cpu and 10 mebibyte memory. The Pods of that DaemonSet should run on all nodes, also controlplanes.

**Namespace:** `project-tiger`

**Solution:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ds-important
  namespace: project-tiger
  labels:
    id: ds-important
    uuid: 18426a0bf59-4e10-923f-c0e078e82462
spec:
  selector:
    matchLabels:
      id: ds-important
  template:
    metadata:
      labels:
        id: ds-important
    spec:
      containers:
      - name: ds-important
        image: httpd:2-alpine
        resources:
          requests:
            cpu: 10m
            memory: 10Mi
          limits:
            memory: 10Mi
```

---

## 25. Deployment with Anti-Affinity and Multiple Containers

Create a Deployment named deploy-important with 3 replicas. The Deployment and its Pods should have label id=very-important. First container named container1 with image nginx: I-alpine. Second container named container2 with image google/pause. There should only ever be one Pod of that Deployment running on one worker node, use topologyKey: kubernetes.io/hostname for this.

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

---

## 26. Check and Renew kube-apiserver Certificate

Check how long the kube-apiserver server certificate is valid using openssl or cfssl. 
- Write the expiration date into `/opt/course/14/expiration`. Run the kubeadm command to list the expiration dates and confirm both methods show the same one.
- Write the kubeadm command that would renew the `kube-apiserver` certificate into `/opt/course/14/kubeadm-renew-certs.sh`

**Solution:**

```bash
openssl x509 -noout -text -in /etc/kubernetes/pki/apiserver.crt 

Check for “Validity: Not after” field

kubeadm certs renew apiserver
```

Store expiration in:

```bash
/opt/course/14/expiration
```

Store renew command in:

```bash
/opt/course/14/kubeadm-renew-certs.sh
```

---

## 27. CoreDNS Update for custom-domain

The CoreDNS configuration in the cluster needs to be updated,
- Make a backup of the existing configuration Yaml and store it at `/opt/course/16/coredns_backup.yml`. You should be able to fast recover from the backup
- Update the CoreDNS configuration in the cluster so that DNS resolution for SERVICE.NAMESPACE. custom-domain will work exactly like and in addition to SERVICE. NAMESPACE.cluster.local
Test your configuration for example from a Pod with `busybox:1-image`. These commands should result in an address:
nslookup kubernetes.default.svc.cluster.local
nslookup kubernetes.default.svc.custom-domain

**Solution:**

```bash
kubectl get cm coredns -n kube-system -o yaml > /opt/course/16/coredns_backup.yaml

kubectl edit cm coredns -n kube-system
```

Update `Corefile`:
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

---

## 28. Crictl Container Info and Logs

Solve this question on: Namespace project-tiger create a Pod named tigers-reunite of image httpd:2-alpine with labels pod=container and container=pod. Find out on which node the Pod is scheduled. SSH into that node and find the containerd container belonging to that Pod.

**Tasks:**
- Find container ID and runtimeType
- Store to `/opt/course/17/pod-container.txt`
- Save logs to `/opt/course/17/pod-container.log`

**Solution:**

```bash
kubectl run tigers-reunite --image=httpd:2-alpine -n project-tiger --labels="pod=container,container=pod"

# Find node it's running on, SSH there:

crictl ps | grep tigers-reunite

crictl inspect <containerID> | grep runtimeType > /opt/course/17/pod-container.txt


sudo -i

crictl logs <containerID> > /opt/course/17/pod-container.log
```
Copy the crictl logs obtained into the solution file
vim inside file: a151ca5ed8861 io.containerd.runc.v2


---

## 29. ETCD Info Extraction

The cluster admin asked you to find out the following formation about etcd running on cka9412 :
• Server private key location
• Server certificate expiration date
• Is client certificate authentication enabled
Write these information into /opt/course/pl/etcd-info. Txt

**Tasks:**
- Server private key location
- Certificate expiration
- Client auth enabled?

**Solution:**

```bash
Look into /etc/kubernetes/manifests/etcd.yaml for “—key-file” “—cert-file” 


cd /etc/kubernetes/manifests
# Certificate check:
openssl x509 -noout -text -in /etc/kubernetes/pki/etcd/server.crt 
# Look for client-cert-auth: true in etcd manifest
```
Copy the “Validity – Not after” field into Solution file
Client-cert YES TRUE

```bash
Server private key location: /etc/kubernetes/pki/etcd/server.key
Server certificate expiration date: Feb 20 18:24:56 2026 GMT
Is client certificate authentication enabled: yes
```

Store in:

```bash
/opt/course/pl/etcd-info.txt
```

---

## 30. Create StorageClass local-kiddie

Create a new named local-kiddie with the provisioner rancher. io/local-path.
- Set the volumeBindingMode to WaitForFirstConsumer
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

make `storageclass.kubernetes.io/is-default-class: "false"` for the other StorageClass
Use patch
`k patch sc local-kiddie -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class:"true"}}}'`

----

## 31. ETCD Repair

15. etcd Repair
The cluster configured by kubeadm has been migrated to a new machine. It requires configuration changes to run successfully.
Task
- Repair the single — node cluster damaged during machine migration.
- First, identify the damaged cluster components and investigate the causes of their damage.
Note: The decommissioned cluster uses an external etcd server. Next, repair the configuration of all damaged cluster components.
Note: Ensure that all necessary services and components are restarted for the changes to take effect. Failure to do so may result in a score reduction. Finally, ensure the cluster is running normally. Ensure: Each node and all Pods are in the Ready state.


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

Search for /resources 
Reduce CPU request to 100m

---
