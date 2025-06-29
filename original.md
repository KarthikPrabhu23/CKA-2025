# CKA-2025

1.	Install cri-dockerd .deb and set system params 
Prepare a Linux system for Kubernetes. Docker is already installed, but you need to configure it for kubeadm.
Task
Complete these tasks to prepare the system for Kubernetes :
Set up cri-dockerd:
Install the Debian package —/cri-dockerd_O.3.9.3-O.ubuntu-jammy_amd64.deb
Debian packages are installed using dpkg.
Enable and start the cri-docker service
Configure these system parameters:
Set net.bridge.bridge-nf-call-iptables to 1
Set net.ipv6.conf.all.forwarding to 1
Set net.ipv4.ip_forward to 1
Set net.netfilter.nf_conntrack_max to 131072

Solution:
```
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.18/cri-dockerd_0.3.18.3-0.ubuntu-jammy_amd64.deb
sudo dpkg -i cri-dockerd_0.3.1.3-0.ubuntu-jammy_amd64.deb
```

Enable and start service:
```
sudo systemctl enable cri-docker.service
sudo systemctl start cri-docker.service
```
To modify the parameters:
`Vim /etc/sysctl.d/cka.conf`
Inside the file, update all required config
```
cat /etc/sysctl.d/cka.conf 
net.bridge.bridge-nf-call-iptables=1
net.ipv6.conf.all.forwarding=1
net.ipv4.ip_forward=1
net.netfilter.nf_conntrack_max=131072
```
(TO apply changes)
`sudo sysctl –system `

2. Verify the cert-manager application which has been deployed in the cluster.
Create a list of all cert-manager Custom Resource Definitions (CRDs) and save it to ~/resources.yaml.
Make sure kubectl's default output format and use kubectl to list CRD's.
Do not set an output format.
Failure to do so will result in a reduced score.
Using kubectl, extract the documentation for the subject specification field of the Certificate Custom Resource and save it to ~/subject.yaml.
You may use any output format that kubectl supports.

Ans:
k get crds | grep cert-manager > ~/resouce.yaml
OR
```
kubectl get crd | grep cert-manager | awk ‘{print $1}’ | xargs -I{} kubectl get crd {} -o yaml >> ~/resources.yaml
k explain certificates.spec.subject > ~/subject.yaml
```
OR
```
kubectl get crd certificates.cert-manager.io -o jsonpath='{.spec.versions[*].schema.openAPIV3Schema.properties.spec.properties.subject}' > subject.yaml
```


3.	TLS
An NGINX Deploy named nginx-static is Running in the nginx-static NS. It is configured using a CfgMap named nginx-config. Update the nginx-config CfgMap to allow only TLSv1.3 connections.re-create, restart, or scale resources as necessary. By 1 using command to test the changes: 
curl --tls-max 1.2 https://web.k8s.local
As TLSV1.2 should not be allowed anymore, the command should fail

Edit ConfigMap to enable TLSv1.2 and make it immutable
```
K get cm 
K get deploy
K get cm -o yaml > configmap.yaml
```
Vim configmap.yaml
Delete the TLSv1.2 from the CM
```
K rollout restart deployment nginx-static
K get deploy nginx-static -o yaml > deploy.yaml
K delete deploy nginx-static
K create -f deploy.yaml
```

4. Create Ingress for a Deployment
Create a new ingress resource named echo in echo-sound namespace
With the following tasks:
Expose the deployment with a service named echo-service on http://example.org/echo using Service port 8080 type=NodePort
The availability of Service echo-service can be checked using the following command which should return 200:

Solution:
```
K expose deployment nginx-deploy –type=NodePort –port=8080 –target-port=8080
```
Service created
Copy ingress.yaml from documentation
Vim ingress.yaml
Add host: {Hostname}
Add service details, earlier created
Port number : 8080
Path: /

4.	Gateway API Migration (Ingress ➝ Gateway + HTTPRoute)
The team from Project r500 wants to replace their Ingress (Networking.k8s.io) with a Gateway Api gateway.networking.k8s.io) solution. The old Ingress is available at /opt/course/13/ingress.yaml. Perform the following in Namespace project-r500 and for the already existing Gateway:
Create a new HTTPRoute named traffic-director which replicates the routes from the old Ingress
Extend the new HTTPRoute with path /auto which redirects to mobile if the User-Agent is exactly mobile and to desktop otherwise
-	Create Gateway
-	Create HTTPRoute
-	Delete Ingress
```yaml
	  - matches:
	    - path:
	        type: PathPrefix
	        value: /
	      header:
	– type: Exact
	  Name: User-Agent
	  Value: Mobile
	    backendRefs:
	    - name: my-service
	      port: 80
```

6. Add Sidecar and Shared Volume
A legacy app needs to be integrated into the Kubernetes built-in logging architecture (i.e. kubectc logs). Adding a streaming co-located container is a good and common way to accomplish this requirement.
Task
Update the existing Deployment synergy-deployment, adding a co-located container named sidecar using the bus box:stable image to the existing Pod.
The new co-located container has. to run the following command: /bin/sh -c "tail -0±1 -f /var/log/synergy-deployment.log"
Use a Volume mounted at /var/log to make the log file synergy-deployment.log available to the co located container.
Do not modify the specification of the existing container other than adding the required.
Hint: Use a shared volume to expose the log file between the main application container and the sidecar
Solution: 
Add a container,
EmptyDir Volume, and 2 volumeMounts to each container
7. PVC + Mount to Deployment
Create a PVC
Edit the deployment to add PVC claim name to the volume inside deployment

8. ArgoCD Install with Helm (No CRDs)
```
Helm repo add argo https://github.com/argoproj/argo-helm
Helm template argocd argo/argo-cd --version 5.51.6
 --set crds.install=false > argocd.yaml
 --skip-crds > argocd.yaml
```

 10. Divide Node Resources (Include Init Containers)
```
K scale deploy wordpress –replicas=0
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
k scale deploy wordpress –replicas=3

10. Least Permissive NetworkPolicy
```yaml
apiVersion: netvorking. k8s. io/vl
kind: NetworkPolicy
metadata :
  name: frontend-from-backend
  namespace: backend
spec:
  podSelector :
    matchLabeis :
      app: backend
  policyTypes :
    - Ingress
      ingress :
      - from:
        - namespaceSeIector:
          matchLabeis :
            app: frontend
        podSelector :
          matchLabeis :
            app: frontend
          ports :
            protocol: TCP
            port: 80
```

11. Install CNI: Flannel vs Calico
Calico: Network policy
```
Curl -sl https:tigera-operator.yaml
K apply -f https:tigera-opearator.yaml 
```
If error, use k create -f 

Flannel:
3.	Curl -sL https:kube-flannel.yaml
4.	K apply -f https://flannel.yaml
5.	Pods will be failing. Do k logs pod-flannel -n kube-flannel
6.	K edit cm kube-flannel-cfg -n kube-flannel
7.	Go to net-conf.json> “Network”: “192.168.0.0/16” (Edit with the IP address present in K Logs)
12. HPA - Horizontal Pod Autoscaler
Autoscaling v2

apiVersion: autoscaling/v2
kind: HorizontalPodAutosca1er
"Etadata :
name: apache-server
namespace: autoscale
spec:
scaleTargetRef :
apiVersion: apps/vl
kind: Deployment
name: apache-deployment
minReplicas : 1
maxReplicas: 5
metrics :
type: Resource
resource :
name: cpu
target :
type: Utilization
averageutilization : 50
behavior:
scaloown :
stabilizatiorwindowseconds: 30
13. Troubleshooting

14. Expose Deployment via NodePort and Fix Port in Deployment
K edit deploy
Inside spec.containers
ports :
-	name: http
containerPort: 80
protocol: TCP

k get deploy –show-labels. Use the labels while creating the service
Copy a Nodeport svc file from doc. Replace the labels with the above labels.
apiVersion: vl
kind: Service
metadata :
nane: nodeport-service
namespace: relative
spec:
type: NodePort
selector:
app: nodeport-deployment
ports :
port: 80
protocol: TCP
targetPort: 80
nodePort: 30080
15. Priority Class
Get PC from documentation
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 100000
globalDefault: false
k edit deploy 
Insert in spec
priorityClassName: high-priority
16. Kubeconfig
Solve this question on: ssh cka9412
outre asked to extract the following information out of
ubeconfig file /opt/course/l/kubeconfig on cka9412 :
1. Write all kubeconfig context names into /opt/course/l/contexts , one per line
2. Write the name of the current context into /opt/course/l/current—context
3. Write the client-key of user account—base64-  decoded into /opt/course/l/cert

Solution: 
kubectl config get-contexts - -kubeconfig /opt/course/l/kubeconfig -o name > /opt/course/l/contexts
kubectl config current-context --kubeconfig /opt/course/l/kubeconfig > /opt/course/l/current-context
3. cat /opt/course/1/kubeconfig 
Copy the client-key-data field 
Echo <client-key-data> | base64 -d > answer-file

17. QOS Pods
kubectl get pods -n project-c13 -o custom-columns="NAME:.metadata.name, QOS:.status .qosClass" I grep "BestEffort" | awk '{print $1}' > /opt/course/4/pods-terminated-first. Txt

18. Kustomize HPA
Solve this question on: ssh cka5774
Previously the application api-gateway used some external autoscaler which should now be replaced with a HorizontalPodAutoscaler (HPA). The application has been deployed to Namespaces api-gateway-staging and api-gateway-prod like this:
kubectl kustomize /opt/course/5/api—gateway/staging
kubectl kustomize /opt/course/5/api—gateway/prod
sign the Kustomize config at /opt/course/5/api-gateway do the following:
1. Remove the ConfigMap horizontal—scaling—config completely
2. Add HPA named api-gateway for the Deployment api-gateway with min 2 and max 4 replicas. It should scale at 5096 average CPU utilisation
3. In prod the HPA should have max 6 replicas
4. Apply your changes for staging and prod so they're reflected in the cluster

Solution: 
Remove the HPA from ConfigMap in 3 directory: Base, staging, prod
1. Remove the ConfigMap horizontal-scaling-config
•	Locate and delete the ConfigMap manifest file (likely named horizontal-scaling-config.yaml) in /opt/course/5/api-gateway/base or wherever it is referenced.
•	Remove its reference from the kustomization.yaml under resources or configMapGenerator.
2. Add HPA for the Deployment
Create a new HPA manifest in the base
Create a file /opt/course/5/api-gateway/base/hpa.yaml with the following content:
YAML
# Code Generated by Sidekick is for learning and experimentation purposes only.
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
Note: The value "5096" for CPU utilization seems like a typo. Kubernetes HPA expects a percentage (e.g., 50 for 50%). If you meant a different value, adjust accordingly.
3. Patch HPA for Production (maxReplicas: 6)
Create a patch file at /opt/course/5/api-gateway/prod/hpa-patch.yaml:
YAML
# Code Generated by Sidekick is for learning and experimentation purposes only.
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-gateway
spec:
  maxReplicas: 6
4. Update kustomization.yaml Files
In /opt/course/5/api-gateway/base/kustomization.yaml:
Add the HPA resource:
YAML
resources:
  - deployment.yaml
  # ... other resources
  - hpa.yaml
Remove the ConfigMap if present:
YAML
# Remove or comment out any line like:
# - horizontal-scaling-config.yaml
In /opt/course/5/api-gateway/prod/kustomization.yaml:
Add the patch:
YAML
patchesStrategicMerge:
  - hpa-patch.yaml
5. Apply the Changes
Run the following commands to apply the changes:
Bash
# Code Generated by Sidekick is for learning and experimentation purposes only.
kubectl apply -k /opt/course/5/api-gateway/staging
kubectl apply -k /opt/course/5/api-gateway/prod
Summary
•	ConfigMap removed from base and kustomization.
•	HPA added in base with min 2, max 4, 50% CPU utilization.
•	Prod overlays patch maxReplicas to 6.
•	Changes applied to both staging and prod.

19. Metrics Server
The metrics-server has been installed in the cluster. Write two bash scripts which use kubectl :
1. Script /opt/course/7/node.sh should show resource usage of Nodes
2. Script /opt/course/7/pod.sh should show resource usage of Pods and their containers

Solution 
# ! /bin/bash
kubectl top node
 
#!/bin/bash
kubectl top pod --containers

20. Kube upgrade version
Solve this question on: ssh cka3962
Your coworker notified you that node cka3962-node1 is running an older Kubernetes version and is not even part of the cluster yet.
1. Update the node's Kubernetes to the exact version of the controlplane
2. Add the node to the cluster using kubeadm 
You can connect to the worker node using ssh cka3962-node1 from cka3962
Solution:
Use documentation to upgrade node version to 1.32
Kubeadm token create –print-join-command 

21. There is ServiceAccount secret-reader in Namespace project-swan . Create a Pod of image nginx:1-alpine named api-contact which uses this ServiceAccount. Exec into the Pod and use curl to manually query all Secrets from the Kubernetes Api. Write the result into file /opt/course/9/result.json .

candidate@cka9412:- $ kubectl run api-contact image=nginx: I-alpine -n project-swan --dry- run-client -o yaml > q9-pod.yaml
Add: 
spec: 
serviceAccountName: secret- reader

kubectl exec -it api-contact  –n project-swan – sh

curt -k https://kubernetes.defautt/api/vl/secrets -H "Authorization: Bearer ${TOKEN}"
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token) 
curt -k https://kubernetes.defautt/api/vl/secrets -H "Authorization: Bearer ${TOKEN}" > results.json
exit
kubectl exec -it api-contact  –n project-swan – cat results.json > solution.json

22. Solve this question on: ssh cka3962
Create a new ServiceAccount processor in Namespace project-hamster. Create a Role and RoleBinding, both named processor as well. These should allow the new SA to only create Secrets and ConfigMaps in that Namespace.

K create sa processor -n project-hamster
Vim from documentaion
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata
namespace: project-hamster
name: processor
rules:
- apiGroups: [“”]
Resources: ["secrets", "configmaps"]
Verbs: ["create”]

Get rolebinding from documentation:
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
name: processor
namespace: project-hamster
subjects:
kind: ServiceAccount
namespace: project-hamster
name: processor  
roleRef:
kind: Role  
name: processor
apiGroup: rbac.authorization.k8s.io

23. Solve this question on: ssh cka7968
Install the MinlO Operator using Helm in Namespace minio . Then configure and create the Tenant CRD:
1. Create Namespace minio
2. Install Helm chart minio/operator into the new Namespace. The Helm Release should be called minio—operator
3. Update the Tenant resource in /opt/course/2/minio-tenant.yaml to include enableSFTP: true under features
4. Create the Tenant resource from /opt/course/2/minio-tenant. Yaml It is not required for MinlO to run properly.
Installing the Helm Chart and the Tenant resource as requested is enough

Solution:
helm install minio-operator minio/operator -n minio
vim /opt/course/2/minio-tenant.yaml
Add in spec.features:= enableSFTP: true
K apply -f minio-tenant.yaml

24. DaemonSet : Solve this question on: ssh cka2556
Use Namespace project-tiger for the following. Create a DaemonSet named ds-important with image httpd:2-aIpine and labels id=ds—important and uuid=18426aØbf59-4e1Ø-923f-cØeØ78e82462 . The Pods it creates should request 10 millicore cpu and 10 mebibyte memory. The Pods of that DaemonSet should run on all nodes, also controlplanes.
Solution: Copy daemonset from Doc
containers
- name: ds-important
image: httpd:2-alpine
resources :
limits:
memory: 10Mi
requests:
cpu: 10m
memory: 10Mi

25. Solve this question on: ssh cka2556
Implement the following in Namespace project-tiger:
Create a Deployment named deploy-important with 3 replicas. The Deployment and its Pods should have label id=very-important. First container named container1 with image nginx: I-alpine. Second container named container2 with image google/pause. There should only ever be one Pod of that Deployment running on one worker node, use topologyKey: kubernetes.io/hostname for this.
Solution:
spec :
replicas: 3
selector:
matchLabels :
id: very-important
strategy:
template :
metadata.
creationTimestamp: null
labels :
id: very-important
spec :
containers :
- image: nginx:1-alpine
name: containerl
resources: { }
- image: google/pause
name: container 2
affinity:
podAntiAffinity :
requiredDuringSchedulingIgnoredDuringExecution :
- labelSelector:
matchExpressions :
- key: id
operator: In
values: 
-	Very-important
topologyKey: “kubernetes.io/hostname”

26. Perform some tasks on cluster certificates:
1. Check how long the kube-apiserver server certificate is valid using openssl or cfssl. Write the expiration date into /opt/course/14/expiration. Run the kubeadm command to list the expiration dates and confirm both methods show the same one.
2. Write the kubeadm command that would renew the kube-apiserver certificate into /opt/course/14/kubeadm-renew-certs.sh
Solution:
openssl x509  -noout -text -in /etc/Kubernetes/pki/apiserver.crt 
Check for “Validity: Not after” field
Kubeadm certs renew apiserver

27. The CoreDNS configuration in the cluster needs to be updated,
1. Make a backup of the existing configuration Yaml and store it at /opt/course/16/coredns_backup.yml. You should be able to fast recover from the backup
2. Update the CoreDNS configuration in the cluster so that DNS resolution for SERVICE.NAMESPACE. custom-domain will work exactly like and in addition to SERVICE. NAMESPACE. cluster. local
Test your configuration for example from a Pod with busybox:1-image. These commands should result in an address:
nslookup kubernetes.default. svc.cluster. local
nslookup kubernetes.default. svc.custom—domain

Solution:  kubectl get cm coredns -n kube-system -o yaml > /opt/course/16/coredns_backup.yaml
Kubectl edit cm codedns -n kube-system 
ready
kubernetes custom-domain cluster.local in-addr.arpa ip6.arpa {
pods insecure
fall through in-addr.arpa ip6.arpa
ttl 30
}

Kubectl rollout restart deployment coredns -n kube-system 

28. Crictl Pod
Solve this question on: Namespace project-tiger create a Pod named tigers-reunite of image httpd:2-alpine with labels od=container and container=pod. Find out on which node the Pod is scheduled. Ssh into that node and find the containerd container belonging to that Pod.
Using command crictl:
1. Write the ID of the container and the info. runtimeType into /opt/course/17/pod-container. txt
2. Write the logs of the container into /opt/course/17/pod-container. Log

Solution:
kubectl tigers-reunite run --image=httpd:2-alpine -n project-tiger  --labels "pod=container, container=pod"

crictl ps | grep tiger-reunite 
crictl inspect <ContainerID> | grep runtimeType

vim inside file: a151ca5ed8861 io.containerd.runc.v2

sudo -i
crictl logs <ContainerID> 

Copy the crictl logs obtained into the solution file

29. ETCD Info
Solve this question on: ssh cka9412
The cluster admin asked you to find out the following formation about etcd running on cka9412 :
• Server private key location
• Server certificate expiration date
• Is client certificate authentication enabled
Write these information into /opt/course/pl/etcd-info. Txt

Solution:

Cd /etc/Kubernetes/manifests
Vim etcd.yaml
“—key-file” “—cert-file” “

Candidate= /etc/kubernetes/manifests$ openssl x509 -noout -text -in /etc/kubernetes/pki/etcd/server.crt

Copy the “Validity – Not after” field into Solution file
Client-cert YES TRUE

Server private key location: /etc/kubernetes/pki/etcd/server. key
Server certificate expiration date: Feb 20 18:24:56 2026 GMT
Is client certificate authentication enabled: yes

30. Create a new named local-kiddie with the provisioner rancher. io/local-path.
set the volumeBindingMode to WaitForFirstConsumer
Configure the StorageClass to default StorageClass
Do not modify any existing Deployment or PersistentVolumeClaim

Solution:
apiversion: storage.Kgs.io/vl
kind: storageclass
metadata: 
name: low-latency
storageclass. kliernetes. io/is-defåult-class :
provisioner: rancher. io/local-path
WaitForFirstConslner
"false"
