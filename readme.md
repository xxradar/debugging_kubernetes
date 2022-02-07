### Essential POD behaviour

Pods are the fundamental building block of Kubernetes applications. 
Pods are disposable and  replaceable. You cannot add a container to a running Pod for example to troubleshoot a problem. 
When you try to update or patch a running pod, the pod is terminated and a new one is deployed.

This behavior can be illustrated.

Let's deploy a few `nginx` pods ...
```
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
EOF
```
Note the pod name as well as the READY 1/1 state. 
This 1/1 actually indicates the number of running containers in the pod.

```
kubectl get po --selector app=nginx
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-7848d4b86f-sp6q5   1/1     Running   0          19s
nginx-deployment-7848d4b86f-672tn   1/1     Running   0          19s
nginx-deployment-7848d4b86f-w22ft   1/1     Running   0          19s
```
Let's try to add a container to the running pods ...
```
cat <<EOF >patch.yaml
spec:
  template:
    spec:
      containers:
      - name: busybox
        image: busybox
        command:
        - sleep
        - "3600"
EOF
```
```
kubectl patch deploy nginx-deployment --patch "$(cat patch.yaml)"
```
```
 kubectl get po --selector app=nginx

NAME                                READY   STATUS        RESTARTS   AGE
nginx-deployment-89cfdb59c-x7z94    2/2     Running       0          20s
nginx-deployment-89cfdb59c-4zzjl    2/2     Running       0          16s
nginx-deployment-7848d4b86f-672tn   0/1     Terminating   0          64s
nginx-deployment-89cfdb59c-mdr89    2/2     Running       0          12s
```
As you can see the running pods are terminated and replaced by new pods. Also note the 2/2 indicating we have two running containers in the pod.
We can verify this
```
% kubectl describe po nginx-deployment-89cfdb59c-x7z94
Name:         nginx-deployment-89cfdb59c-x7z94
Namespace:    default
Priority:     0
Node:         lima-rancher-desktop/192.168.5.15
Start Time:   Mon, 07 Feb 2022 19:44:38 +0100
Labels:       app=nginx
              pod-template-hash=89cfdb59c
Annotations:  <none>
Status:       Running
IP:           10.42.0.234
IPs:
  IP:           10.42.0.234
Controlled By:  ReplicaSet/nginx-deployment-89cfdb59c
Containers:
  busybox:
    Container ID:  containerd://6ec4c86dc6176b9b51b0cb6a2fbd4456cced3f2f39f2e167411a51010c159547
    Image:         busybox
    Image ID:      docker.io/library/busybox@sha256:afcc7f1ac1b49db317a7196c902e61c6c3c4607d63599ee1a82d702d249a0ccb
    Port:          <none>
    Host Port:     <none>
    Command:
      sleep
      3600
    State:          Running
      Started:      Mon, 07 Feb 2022 19:44:40 +0100
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-fkwnb (ro)
  nginx:
    Container ID:   containerd://d807f08e432896c265f14fa53bec26ad3036ece3b81106672de6d9309c3cd0db
    Image:          nginx
    Image ID:       docker.io/library/nginx@sha256:2834dc507516af02784808c5f48b7cbe38b8ed5d0f4837f16e78d00deb7e7767
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Mon, 07 Feb 2022 19:44:42 +0100
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-fkwnb (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  kube-api-access-fkwnb:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  7m41s  default-scheduler  Successfully assigned default/nginx-deployment-89cfdb59c-x7z94 to lima-rancher-desktop
  Normal  Pulling    7m40s  kubelet            Pulling image "busybox"
  Normal  Pulled     7m39s  kubelet            Successfully pulled image "busybox" in 1.742582062s
  Normal  Created    7m39s  kubelet            Created container busybox
  Normal  Started    7m39s  kubelet            Started container busybox
  Normal  Pulling    7m39s  kubelet            Pulling image "nginx"
  Normal  Pulled     7m37s  kubelet            Successfully pulled image "nginx" in 1.547346047s
  Normal  Created    7m37s  kubelet            Created container nginx
  Normal  Started    7m37s  kubelet            Started container nginx
 ```

Sometimes it's necessary to inspect the state of an existing Pod.
In order to examine a running pod, you can use the `kubectl exec` mechanism. This works fine to troublehshoot issues, except that secure container images do not contain debugging tools (distroless or hardened images) amd runtime security solutions might block installing them.

The example of patching containers might be a solution but as already stated, the pods are replaced and the second container only shares the pods networking namespace by default.

To troubleshoot a hard-to-reproduce bug, this might be challenging.

### `kubectl debug` to the rescue
`kubectl debug` might help us in a few different ways.
  * Create a copy of an existing pod (with certain attributes changed)
  * Add an ephemeral container to an already running pod, for example to add debugging utilities without restarting the pod.
  * Create a new pod that runs in the node's host namespaces and can access the node's filesystem.

