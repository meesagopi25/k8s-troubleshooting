Below is your complete Markdown / README-style training guide for Kubernetes Pending Pod Troubleshooting.
You can copy/paste this directly into a README.md file and use it for team training or workshops.

🧪 Kubernetes Troubleshooting Lab
Understanding & Fixing Pods Stuck in Pending State
This lab provides hands-on, practical scenarios covering every major reason a Kubernetes Pod may be stuck in the Pending phase.
Each scenario includes:
	• What the issue is
	• How to reproduce it
	• How to troubleshoot it
	• How to fix it
Works on:
	• Minikube
	• KIND
	• OpenShift Local
	• OpenShift Sandbox
	• Any Kubernetes cluster

🧩 Prerequisites
Ensure you can run:
kubectl get nodes
kubectl apply -f file.yaml
kubectl describe pod <pod>
kubectl describe node <node>

📘 Table of Contents
	1. Insufficient CPU
	2. Insufficient Memory
	3. Node Taints
	4. Unbound PVC
	5. Node Selector Mismatch
	6. Anti-Affinity Rules
	7. SCC / PSP Security Restrictions
	8. ResourceQuota Violations
	9. GPU Requests Not Allowed
	10. Ephemeral Storage Violations
	11. Common Troubleshooting Commands

1️⃣ Insufficient CPU
❌ Problem
Pod requests more CPU than available in the cluster.
YAML:
apiVersion: v1
kind: Pod
metadata:
  name: cpu-hog
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: "4000m"
      limits:
        cpu: "20"

Create the pod:
bash-5.1 ~ $ oc apply -f cpu-hog.yml 
Expected error:
Error from server (Forbidden): error when creating "cpu-hog.yml": pods "cpu-hog" is forbidden: exceeded quota: compute-deploy, requested: requests.cpu=4, used: requests.cpu=200m, limited: requests.cpu=3
Fix:
Lower CPU request.

2️⃣ Insufficient Memory
YAML:
apiVersion: v1
kind: Pod
metadata:
  name: memory-hog
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "30Gi"
Create the pod:
bash-5.1 ~ $ oc apply -f cpu-hog.yml 
Expected error:
bash-5.1 ~ $ oc apply -f memory-hog.yml 
The Pod "memory-hog" is invalid: spec.containers[0].resources.requests: Invalid value: "80Gi": must be less than or equal to memory limit of 1000Mi
bash-5.1 ~ $ 
Troubleshooting:
bash-5.1 ~ $ oc get limits -n mg1982-dev
NAME              CREATED AT
resource-limits   2025-11-16T06:42:07Z
bash-5.1 ~ $ oc describe limits resource-limits
Name:       resource-limits
Namespace:  mg1982-dev
Type        Resource  Min  Max  Default Request  Default Limit  Max Limit/Request Ratio
----        --------  ---  ---  ---------------  -------------  -----------------------
Container   cpu       -    -    10m              1              -
Container   memory    -    -    64Mi             1000Mi         -

Perfect — this explains exactly why your Pod failed.
Here is what your LimitRange says:
Container Memory:
  Default Request: 64Mi
  Default Limit: 1000Mi (1Gi)
  Max: none
Even though it shows “Max: -”, the default limit (1000Mi) becomes the effective limit unless you explicitly set your own.
And requests must always be ≤ limits.
Your pod was:
requests:
  memory: 30Gi
limits:
  memory: 30Gi
This violates the rule:
request.memory (30Gi) > default limit.memory (1000Mi)
So Kubernetes rejected it with:
Invalid value: "30Gi": must be less than or equal to memory limit of 1000Mi
✔ This error is NOT from ResourceQuota
✔ It is from LimitRange in your namespace

⭐ Why the request fails even though “Max” is empty?
Your LimitRange:
Default Request: 64Mi
Default Limit: 1000Mi
Max: -
This means:
	• If you do not define a limit, Kubernetes automatically assigns 1000Mi.
	• Then it checks →
request ≤ limit
But your request is 30Gi, which is > 1000Mi → invalid.
	• Even if you set your own limit, if it exceeds the ResourceQuota or policy, it still may fail.

⭐ IMPORTANT RULE
Every container must satisfy:
request.memory ≤ limit.memory

Your request: 30Gi
Default limit: 1000Mi
→ Violation, so pod is rejected.

✅ How to fix the issue
You have two options:

OPTION 1 — Set your own limit that is ≥ request
Example (valid):
resources:
  requests:
    memory: 30Gi
  limits:
    memory: 30Gi
But this will still fail because of ResourceQuota?
NO — your quota allows up to 30Gi.
It fails because you must override the default limit by explicitly defining your own.
But!
⚠ In OpenShift Developer Sandbox, LimitRange usually enforces a maximum pod size
Even though Max is “-”, OpenShift enforces a cluster policy that limits Pods to 1Gi.
So even if you set:
limits:
  memory: 30Gi
OpenShift Sandbox will still block it at admission because Sandbox cluster is intentionally small.

OPTION 2 — Use a smaller request (recommended)
Example of a Pod that will succeed:
resources:
  requests:
    memory: 512Mi
  limits:
    memory: 1Gi

🎯 Conclusion
Your error happens because of this rule:
✔ Request must be ≤ Limit
✔ LimitRange sets default memory limit = 1000Mi
✔ Your request = 30Gi > 1000Mi → INVALID
Even though your ResourceQuota allows 30Gi,
your LimitRange restricts per-container memory to max 1000Mi
(because the cluster enforces a default limit and does not allow raising it).

💬 If you want:
I can explain:
✅ The difference between ResourceQuota vs LimitRange
✅ Why OpenShift Sandbox uses 1Gi limit
✅ How to redesign Pods to fit Sandbox restrictions
Just tell me!

3️⃣ Node Taints
Add taint:
kubectl taint nodes <node> dedicated=backend:NoSchedule
Pod that fails:
apiVersion: v1
kind: Pod
metadata:
  name: fail-on-taint
spec:
  containers:
  - name: app
    image: nginx

Troubleshoot:
kubectl describe pod no-toleration
Expected:
node(s) had taint {dedicated=backend:NoSchedule}, that the pod didn't tolerate
Fix:
Add toleration:
tolerations:
- key: "dedicated"
  operator: "Equal"
  value: "backend"
  effect: "NoSchedule"

apiVersion: v1
kind: Pod
metadata:
  name: tolerate-backend-taint
  labels:
    app: test-app
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "backend"
    effect: "NoSchedule"

  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "200m"
        memory: "256Mi"
Great — here are full examples of Pods that will FORCE scheduling onto a tainted node using:
	• Tolerations
	• NodeSelector
	• NodeAffinity
These examples assume your taint is:
dedicated=backend:NoSchedule
and the target node is:
kafka-cluster-worker2
Before running these examples, ensure the node is tainted:
oc adm taint nodes kafka-cluster-worker2 dedicated=backend:NoSchedule

⭐ Pod that FORCES scheduling onto the tainted node using nodeSelector + toleration
✔ Will schedule ONLY on kafka-cluster-worker2
✔ Will FAIL if the node is down
✔ Toleration allows bypassing the taint
Save as pod-force-nodeselector.yaml
apiVersion: v1
kind: Pod
metadata:
  name: force-nodeselector
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "backend"
    effect: "NoSchedule"
nodeSelector:
    kubernetes.io/hostname: kafka-cluster-worker2
containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
👉 Steps:
	1. Label the node (hostname is usually auto-set):
oc label node kafka-cluster-worker2 kubernetes.io/hostname=kafka-cluster-worker2 --overwrite
	1. Deploy the pod:
oc apply -f pod-force-nodeselector.yaml
	1. Check where it landed:
oc get pod -o wide
You will see:
force-nodeselector  Running  kafka-cluster-worker2

⭐ Pod that FORCES scheduling using NodeAffinity + toleration
✔ More flexible than nodeSelector
✔ Allows matching multiple values
✔ Still forces it onto worker2
Save as pod-force-affinity.yaml
apiVersion: v1
kind: Pod
metadata:
  name: force-affinity
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "backend"
    effect: "NoSchedule"
affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
            - kafka-cluster-worker2
containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
👉 Deploy:
oc apply -f pod-force-affinity.yaml

⭐Pod that FAILS even with NodeSelector (no toleration)
This is useful for testing Pending state.
Save as pod-fail-no-toleration.yaml
apiVersion: v1
kind: Pod
metadata:
  name: fail-no-toleration
spec:
  nodeSelector:
    kubernetes.io/hostname: kafka-cluster-worker2
containers:
  - name: nginx
    image: nginx
👉 Result:
oc describe pod fail-no-toleration
Expected:
node(s) had taint {dedicated=backend:NoSchedule} that the pod didn't tolerate

⭐ Pod that FAILS with Affinity (no toleration)
Save as pod-fail-affinity.yaml
apiVersion: v1
kind: Pod
metadata:
  name: fail-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
            - kafka-cluster-worker2
containers:
  - name: nginx
    image: nginx
👉 Result:
oc describe pod fail-affinity
Expected:
node(s) had taint {dedicated=backend:NoSchedule} that the pod didn't tolerate

✔ Summary Comparison
Pod Type	Has Toleration?	Has NodeSelection?	Expected Result
force-nodeselector	Yes	Yes	Schedules on worker2
force-affinity	Yes	Yes (via affinity)	Schedules on worker2
fail-no-toleration	No	Yes	Fails (Pending)
fail-affinity	No	Yes	Fails (Pending)

--------------------------------------------------------------------------------------------------------------------------------
4️⃣ Unbound PVC
Unbindable PVC:
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: bad-pvc
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 10Gi
  storageClassName: does-not-exist
Pod that fails:
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod
spec:
  containers:
  - name: busy
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: bad-pvc
Troubleshoot:
persistentvolumeclaim "bad-pvc" is not bound
Fix:
Use correct StorageClass or create matching PV.

5️⃣ Node Selector Mismatch
Failing Pod:
apiVersion: v1
kind: Pod
metadata:
  name: node-selector-fail
spec:
  nodeSelector:
    disktype: ssd
  containers:
  - name: app
    image: nginx
Describe:
0 nodes match pod's node selector
Fix:
Label a node:
kubectl label node <node> disktype=ssd

6️⃣ Anti-Affinity Rules
Pod with impossible anti-affinity:
apiVersion: v1
kind: Pod
metadata:
  name: anti-affinity-fail
  labels:
    app: test
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: test
        topologyKey: "kubernetes.io/hostname"
  containers:
  - name: app
    image: nginx
Describe:
pod anti-affinity rules not satisfied
Fix:
Use preferred anti-affinity or reduce strictness.

Few more examples:
Node Affinity and Pod Anti-Affinity are scheduling rules Kubernetes uses to decide which nodes a Pod should or should not run on.
They give you fine-grained control over Pod placement, similar to advanced “nodeSelector”, but much more powerful.

🧩6. 1Node Affinity (Pod prefers/needs specific nodes)
Node Affinity tells Kubernetes:
✔ Which nodes a Pod should run on
✔ Or which nodes a Pod must run on**
✔ Based on node labels (ex: environment=prod, region=us-east, instance=big)
🎯 Types of Node Affinity
Type	Meaning
requiredDuringSchedulingIgnoredDuringExecution	Pod must be scheduled on matching nodes (hard rule).
preferredDuringSchedulingIgnoredDuringExecution	Pod prefers matching nodes but will run elsewhere (soft rule).

⭐ Node Affinity Example – Pod MUST run on nodes labeled env=prod
Step 1 — Label the node:
oc label node kafka-cluster-worker env=prod
Step 2 — Pod with Node Affinity:
apiVersion: v1
kind: Pod
metadata:
  name: pod-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: env
            operator: In
            values:
            - prod
containers:
  - name: nginx
    image: nginx
Result:
✔ Pod runs only on nodes labeled env=prod
❌ If no node has that label → Pod stays Pending
Describe output:
0/3 nodes are available: 3 node(s) didn't match node affinity

🧩 6.2 Pod Anti-Affinity (Pod must avoid certain pods)
Pod Anti-Affinity tells Kubernetes:
✔ Do NOT place this Pod on a node that already runs similar Pods
✔ Useful for high availability, spreading Pods across nodes
Example use cases:
	• Spread replicas across nodes (avoid single-node failure)
	• Avoid running two database pods on same node
	• Ensure web pods do not run where cache pods run

⭐ Anti-Affinity Example – Different Pods cannot share same node
Scenario
You want no two Pods with label app=web to run on the same node.
Deployment example:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: web
            topologyKey: "kubernetes.io/hostname"
containers:
      - name: nginx
        image: nginx
Result:
✔ Each replica gets scheduled on a different node
✔ Ensures high availability
✔ If cluster has fewer nodes than replicas → extra pods stay Pending
describe shows:
0/3 nodes are available: pod anti-affinity rules not satisfied

🧠 How topologyKey works
topologyKey: kubernetes.io/hostname
→ Spread across nodes
Other options:
	• topology.kubernetes.io/zone
	• topology.kubernetes.io/region
Can force pods to schedule in different zones or regions.

⭐ Real-World Use Cases
✔ Node Affinity
	• Run GPU workloads on nodes with GPUs
	• Run high-memory workloads on special nodes
	• Run production pods only on prod-labeled nodes
✔ Pod Anti-Affinity
	• Spread replicas across nodes
	• Prevent two database pods from being co-located
	• Ensure Redis master and slave are on different nodes

🔥 Full Example Combining Both
apiVersion: v1
kind: Pod
metadata:
  name: complex-scheduler
  labels:
    app: payment
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node-type
            operator: In
            values:
            - high-memory
podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: payment
          topologyKey: kubernetes.io/hostname
containers:
  - name: payment-service
    image: nginx
Result:
	• Pod must run on nodes labeled node-type=high-memory
	• Pod prefers not to run on nodes already running other payment pods

🎉 Summary Table
Feature	Purpose	Example
Node Affinity	Pod MUST/PREFERS to run on specific nodes	env=prod
Pod Anti-Affinity	Pod MUST/PREFERS to avoid nodes with certain pods	Spread replicas
Taints/Tolerations	Nodes repel pods; tolerations allow exceptions	maintenance nodes

If you want, I can also provide:
✅ Real interview questions on Affinity & Taints
✅ A combined Affinity + Taint scheduling lab
✅ A cheat sheet diagram for Affinity rules
Just tell me!

7️⃣ SecurityContextConstraints / PSP Failure (OpenShift)
apiVersion: v1
kind: Pod
metadata:
  name: privileged-denied
spec:
  containers:
  - name: app
    image: nginx
    securityContext:
      privileged: true
Describe:
pod is not allowed to use SecurityContextConstraints "privileged"
Fix:
oc adm policy add-scc-to-user privileged -z default

🧠 Why would someone use privileged mode?
	• Running storage plugins
	• Running device drivers
	• Running monitoring tools that need host access
	• Managing the host network
	• Running low-level system daemons (e.g., kube-proxy)
For regular applications (Nginx, Java, Node.js apps),
privileged mode should NEVER be used.

🎯 Summary
This Pod:
	• Runs nginx
	• Asks for privileged mode
	• Will fail in OpenShift unless privileged SCC is granted
	• Would run on vanilla Kubernetes (if no PodSecurityPolicy prevents it)

If you want, I can also explain:
✅ What SCCs are and how they work
✅ The difference between PodSecurityPolicy vs SCC
✅ How to run containers as non-root


8️⃣ ResourceQuota Violations
Example: Memory request over quota.
apiVersion: v1
kind: Pod
metadata:
  name: quota-memory-fail
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "40Gi"
Describe:
exceeded quota: compute-deploy, requested: requests.memory=40Gi
Fix:
Reduce memory request.

9️⃣ GPU Requests Not Allowed
apiVersion: v1
kind: Pod
metadata:
  name: gpu-fail
spec:
  containers:
  - name: cuda
    image: nvidia/cuda
    resources:
      requests:
        nvidia.com/gpu: 1
Error:
exceeded quota: requests.nvidia.com/gpu=1, limited: 0
Fix:
Remove GPU request.

🔟 Ephemeral Storage Violations
apiVersion: v1
kind: Pod
metadata:
  name: ephemeral-fail
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        ephemeral-storage: "20Gi"   # Limit is usually smaller
Describe:
exceeded quota: requests.ephemeral-storage

1️⃣1️⃣ Common Troubleshooting Commands
Check events:
kubectl describe pod <pod>
Check node conditions:
kubectl describe node <node>
Check taints:
kubectl describe node <node> | grep -i taint
Check resource usage:
kubectl top pods
kubectl top nodes
Check PVC:
kubectl get pvc
kubectl describe pvc <pvc>

🎓 Conclusion
This lab provides hands-on examples and troubleshooting practice for all major causes of Pods stuck in Pending:
	• Resource shortages
	• Taints
	• PVC issues
	• Node selector mismatch
	• Affinity constraints
	• Security restrictions
	• ResourceQuota enforcement
	• GPU & ephemeral storage limits
<img width="852" height="20279" alt="image" src="https://github.com/user-attachments/assets/02752e84-5de4-4551-8822-70231e61521a" />

