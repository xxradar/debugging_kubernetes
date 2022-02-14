## Kubectl debug and ephemeral containers
### Essential POD behaviour

Pods are the fundamental building block of Kubernetes applications. 
Pods are disposable and  replaceable. You cannot add a container to a running Pod for example to troubleshoot a problem. 
When you try to update or patch a running pod, the pod is terminated and a new one is deployed.

This behavior can be illustrated.

Let's deploy a few `nginx` pods ...
```
cat <<EOF >nginx-deployment.yaml
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
```
kubectl apply -f nginx-deployment.yaml
```
Note the pod name as well as the `READY 1/1` state. 
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
As you can see the running pods are terminated and replaced by new pods. Also note the `2/2` indicating we have two running containers in the pod.
We can list the container images like this
```
kubectl get deploy/nginx-deployment -o jsonpath='{.spec.template.spec.containers[:2].image}'
```
or 
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

### Create a copy of an existing pod
In order to demonstrate the behavior, let's reset out deployment.
```
kubectl delete deploy nginx-deployment
kubectl apply -f nginx-deployment.yaml
```
```
kubectl get po --selector app=nginx

NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-7848d4b86f-6qzm9   1/1     Running   0          2m11s
nginx-deployment-7848d4b86f-xztfq   1/1     Running   0          2m11s
nginx-deployment-7848d4b86f-g6md9   1/1     Running   0          2m11s
```
```
export PODNAME=nginx-deployment-7848d4b86f-6qzm9
```

Ephemeral containers ... prereq + small describtion ....

```
kubectl debug -it $PODNAME  --image=xxradar/hackon --copy-to=my-debugger
Defaulting debug container name to debugger-h2pdm.
If you don't see a command prompt, try pressing enter.
root@my-debugger:/#
```
At this stage the pod is copied. A new pod `my-debugger` is started and a container with the specified `image` is attached . 
You can verify this in a different terminal.
```
kubectl get po
NAME                                READY   STATUS             RESTARTS   AGE
nginx-deployment-7848d4b86f-6qzm9   1/1     Running            0          5m23s
nginx-deployment-7848d4b86f-xztfq   1/1     Running            0          5m23s
nginx-deployment-7848d4b86f-g6md9   1/1     Running            0          5m23s
my-debugger                         2/2     Running            0          2m9s
```
We only shared the network namespace in this example. This is great to test the application over the shared networking stack, but does not grant us access to the process and filesystem. Sharing the process namespace can be obtained via the `--share-processes=true` flag.
```
kubectl debug  -it $PODNAME --image=xxradar/hackon  --copy-to=my-debugger2 --share-processes=true -- bash
Defaulting debug container name to debugger-w9s48.
If you don't see a command prompt, try pressing enter.
root@my-debugger2:/# ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.2  0.0   1028     4 ?        Ss   19:32   0:00 /pause
root         7  0.0  0.0   8856  5328 ?        Ss   19:32   0:00 nginx: master process nginx -g daemon off;
tcpdump     39  0.0  0.0   9244  2456 ?        S    19:32   0:00 nginx: worker process
tcpdump     40  0.0  0.0   9244  2456 ?        S    19:32   0:00 nginx: worker process
root        41  0.2  0.0   4116  3408 pts/0    Ss   19:32   0:00 bash
root        51  0.0  0.0   5888  2844 pts/0    R+   19:32   0:00 ps aux
root@my-debugger2:/#
```
### Adding an ephemeral container to an existing pod
Reset the previous deployment
```
kubectl rollout undo deploy/nginx-deployment
```
```
kubectl get po --selector app=nginx
NAME                                READY   STATUS              RESTARTS   AGE
nginx-deployment-74d589986c-m5k2d   1/1     Running             0          8s
nginx-deployment-74d589986c-pdcnl   1/1     Running             0          8s
nginx-deployment-74d589986c-whhcq   1/1     Running             0          8s  
```
The folowing line will add an ephemeral container to an existing pod WITHOUT re-deploying the pods
```
kubectl debug -it nginx-deployment-74d589986c-m5k2d  --image=xxradar/hackon  -c debug -- bash
```
As we look closer, a container called `debug` is added to an existing pod and shares the network namespace.
```
root@nginx-deployment-74d589986c-m5k2d:/# ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0   4236  3476 pts/0    Ss   14:53   0:00 bash
root        15  0.0  0.0   5880  2944 pts/0    R+   14:59   0:00 ps aux

root@nginx-deployment-74d589986c-m5k2d:/# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 8981
        inet 192.168.172.162  netmask 255.255.255.255  broadcast 0.0.0.0
        inet6 fe80::c466:8aff:feca:e002  prefixlen 64  scopeid 0x20<link>
        ether c6:66:8a:ca:e0:02  txqueuelen 0  (Ethernet)
        RX packets 5  bytes 446 (446.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 12  bytes 936 (936.0 B)
        TX errors 0  dropped 1 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

root@nginx-deployment-74d589986c-m5k2d:/#
```
You can verify in a second terminal and notice that the pod is NOT restarted, neither did the READY state change.<br>
The deployment declaration is not changed neither.
```
kubectl get po --selector app=nginx
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-74d589986c-m5k2d   1/1     Running   0          9m19s
nginx-deployment-74d589986c-pdcnl   1/1     Running   0          9m19s
nginx-deployment-74d589986c-whhcq   1/1     Running   0          9m19s
```
To find more detail, we need to drill to the pod level and find the `Ephemeral Containers` section.
```
kubectl describe po nginx-deployment-74d589986c-m5k2d
Name:         nginx-deployment-74d589986c-m5k2d
Namespace:    default
Priority:     0
Node:         ip-10-1-2-180/10.1.2.180
Start Time:   Mon, 14 Feb 2022 14:52:26 +0000
Labels:       app=nginx
              pod-template-hash=74d589986c
Annotations:  cni.projectcalico.org/containerID: 9d6cb4f2966e0b25090e10f5b6af6433b67fd8f2b21a94a8d09129645e6d9cbb
              cni.projectcalico.org/podIP: 192.168.172.162/32
              cni.projectcalico.org/podIPs: 192.168.172.162/32
Status:       Running
IP:           192.168.172.162
IPs:
  IP:           192.168.172.162
Controlled By:  ReplicaSet/nginx-deployment-74d589986c
Containers:
  nginx:
    Container ID:   containerd://792271189b20e28b291bd19261249296a1dd2a5be7573966ba9d268e4b6030d2
    Image:          nginx
    Image ID:       docker.io/library/nginx@sha256:2834dc507516af02784808c5f48b7cbe38b8ed5d0f4837f16e78d00deb7e7767
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Mon, 14 Feb 2022 14:52:28 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-8jb29 (ro)
Ephemeral Containers:
  debug:
    Container ID:  containerd://c1a73864b5fd1ec2de53c1e6f3e9dc3a75e315508b4be53f72f1bfbfec1b321c
    Image:         xxradar/hackon
    Image ID:      docker.io/xxradar/hackon@sha256:0e90f1b9adf1a1608390c22f81531802257a16bb30fb4bd2b58523c80072c7f6
    Port:          <none>
    Host Port:     <none>
    Command:
      bash
    State:          Running
      Started:      Mon, 14 Feb 2022 14:53:13 +0000
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:         <none>
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  kube-api-access-8jb29:
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
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  11m   default-scheduler  Successfully assigned default/nginx-deployment-74d589986c-m5k2d to ip-10-1-2-180
  Normal  Pulling    11m   kubelet            Pulling image "nginx"
  Normal  Pulled     11m   kubelet            Successfully pulled image "nginx" in 893.906403ms
  Normal  Created    11m   kubelet            Created container nginx
  Normal  Started    11m   kubelet            Started container nginx
  Normal  Pulling    10m   kubelet            Pulling image "xxradar/hackon"
  Normal  Pulled     10m   kubelet            Successfully pulled image "xxradar/hackon" in 7.636289587s
  Normal  Created    10m   kubelet            Created container debug
  Normal  Started    10m   kubelet            Started container debug
ubuntu@ip-10-1-2-12:~$
```
In the follwing example, we'll create an ephemeral container and share the process namespace of an existing container inside the pod.
```
$ kubectl debug -it nginx-deployment-74d589986c-96x5s --image=xxradar/hackon --target nginx -c debug -- bash
Targeting container "nginx". If you don't see processes from this container it may be because the container runtime doesn't support this feature.
If you don't see a command prompt, try pressing enter.
root@nginx-deployment-74d589986c-96x5s:/# ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0   8848  5400 ?        Ss   15:09   0:00 nginx: master process nginx -g daemon off;
tcpdump     31  0.0  0.0   9236  2476 ?        S    15:09   0:00 nginx: worker process
tcpdump     32  0.0  0.0   9236  2476 ?        S    15:09   0:00 nginx: worker process
tcpdump     33  0.0  0.0   9236  2476 ?        S    15:09   0:00 nginx: worker process
tcpdump     34  0.0  0.0   9236  2476 ?        S    15:09   0:00 nginx: worker process
root        35  0.0  0.0   4108  3448 pts/0    Ss   15:10   0:00 bash
root        43  0.0  0.0   5880  2848 pts/0    R+   15:10   0:00 ps aux
```
A trick you can apply to accessing the files in the `nginx` container is via 
```
cd /proc/1/root/etc/nginx
```
```
cat nginx.conf 
...
```
When you're debug container image has for example `tcpdump` installed, you can sniff traffic in a running pod !
```
root@nginx-deployment-74d589986c-5bmgh:~# tcpdump -n port 80
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
15:28:29.970033 IP 192.168.228.0.42704 > 192.168.172.172.80: Flags [S], seq 3769776304, win 62587, options [mss 8941,sackOK,TS val 3815764348 ecr 0,nop,wscale 7], length 0
15:28:29.970072 IP 192.168.172.172.80 > 192.168.228.0.42704: Flags [S.], seq 3784113359, ack 3769776305, win 62503, options [mss 8941,sackOK,TS val 2005209487 ecr 3815764348,nop,wscale 7], length 0
15:28:29.970405 IP 192.168.228.0.42704 > 192.168.172.172.80: Flags [.], ack 1, win 489, options [nop,nop,TS val 3815764349 ecr 2005209487], length 0
15:28:29.970525 IP 192.168.228.0.42704 > 192.168.172.172.80: Flags [P.], seq 1:80, ack 1, win 489, options [nop,nop,TS val 3815764349 ecr 2005209487], length 79: HTTP: GET / HTTP/1.1
```

### Accessing a node
You can access a specific node (w/o the need for SSH) ... 
kubectl debug node/ip-10-1-2-55 -it  --image xxradar/hackon
...
ps aux
...
Accessing host filesystem
cd /host/home/ubuntu/
...
