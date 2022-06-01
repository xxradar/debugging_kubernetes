## Advanced debugging techniques in Kubernetes
## Introduction
Pods are the fundamental building block of Kubernetes applications. They are the smallest, most basic deployable resource and can represent as little as a single instance of a running process. Pods are made of one or more containers sharing some specific Linux namespaces (`netns`, `utsns` and `ipcns`). That is why containers in a pod can share the network interface, IP address, network ports and hostname and communicate over `localhost` or the `127.0.0.1` IP address. On the other hand, containers inside the pod do not share the filesystem (`mntns`), nor can they see each other processes (`pidns`) by default.  You can visualize this by:
```
# Quickly launch a pod
kubectl run --image nginx demowww

# identify the node where the pod is running
kubect get po -o wide 

# On the scheduled node, let's find the process id of an nginx process
$ ps aux | grep -i nginx
root      6229  0.0  0.0   9092  6548 ?        Ss   16:07   0:00 nginx: master process nginx -g daemon off;

# Based on the process id, find all assigned namespaces
$ sudo ps -ax -n -o pid,netns,utsns,ipcns,mntns,pidns,cmd | grep 6229
 6229 4026532927 4026532396 4026532397 4026532456 4026532457 nginx: master process nginx -g daemon off;
 ...

# Based on the ex. netns, we can find all processes of the pod and find the (non)-shared namespaces.
$ sudo ps -ax -n -o pid,netns,utsns,ipcns,mntns,pidns,cmd | grep 4026532927
 4759 4026532927 4026532396 4026532397 4026532395 4026532398 /pause
 6229 4026532927 4026532396 4026532397 4026532456 4026532457 nginx: master process nginx -g daemon off;
 6778 4026532927 4026532396 4026532397 4026532456 4026532457 nginx: worker process
 7016 4026532927 4026532396 4026532397 4026532456 4026532457 nginx: worker process
 7017 4026532927 4026532396 4026532397 4026532456 4026532457 nginx: worker process
 7018 4026532927 4026532396 4026532397 4026532456 4026532457 nginx: worker process
 ...
```
Let's cleanup
```
kubectl delete po demowww
```
## Patching a pod deployment for debugging
Sometimes, you might want to add a container to a running pod for debugging puposes, but this is not as simple as it sounds. When you try to update or patch a running pod to include an additional debugging container, the pod is terminated and a new one is deployed.

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
This `1/1` indicates the number of running containers in the pod.
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
kubectl describe deploy  nginx-deployment
Name:                   nginx-deployment
Namespace:              default
CreationTimestamp:      Wed, 01 Jun 2022 19:28:48 +0000
Labels:                 app=nginx
Annotations:            deployment.kubernetes.io/revision: 2
Selector:               app=nginx
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx
  Containers:
   busybox:
    Image:      busybox
    Port:       <none>
    Host Port:  <none>
    Command:
      sleep
      3600
    Environment:  <none>
    Mounts:       <none>
   nginx:
    Image:        nginx
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   nginx-deployment-7b8b8fdb57 (3/3 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  90s   deployment-controller  Scaled up replica set nginx-deployment-6c8b449b8f to 3
  Normal  ScalingReplicaSet  63s   deployment-controller  Scaled up replica set nginx-deployment-7b8b8fdb57 to 1
  Normal  ScalingReplicaSet  59s   deployment-controller  Scaled down replica set nginx-deployment-6c8b449b8f to 2
  Normal  ScalingReplicaSet  59s   deployment-controller  Scaled up replica set nginx-deployment-7b8b8fdb57 to 2
  Normal  ScalingReplicaSet  54s   deployment-controller  Scaled down replica set nginx-deployment-6c8b449b8f to 1
  Normal  ScalingReplicaSet  54s   deployment-controller  Scaled up replica set nginx-deployment-7b8b8fdb57 to 3
  Normal  ScalingReplicaSet  51s   deployment-controller  Scaled down replica set nginx-deployment-6c8b449b8f to 0
 ```
Sometimes it's necessary to inspect the state of an existing pod. To examine a running pod, you can use the `kubectl exec` mechanism. <br>
This works fine to troubleshoot issues, except that secure container images do not always (and should not) contain debugging tools (distroless or hardened images) and runtime security solutions might block installing them.

The example of patching containers might be a solution but as already stated, the pods are replaced, and the new debug container only shares the pods networking namespace by default and not really practical and error prone <br>

To troubleshoot a hard-to-reproduce bug, this might be challenging.

## `kubectl debug` to the rescue
`kubectl debug` might help us in a few different ways.
  * Create a copy of an existing pod (with certain attributes changed)
  * Add an ephemeral container to an already running pod, for example to add debugging utilities without restarting the pod.
  * Create a new pod that runs in the node's host namespaces and can access the node's filesystem.

### Usecase 1: Create a copy of an existing pod
To demonstrate the behavior, let's reset our deployment.
```
kubectl delete deploy nginx-deployment
kubectl apply -f nginx-deployment.yaml
```
Verify if everything is up and running again.
```
kubectl get po --selector app=nginx

NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-7848d4b86f-6qzm9   1/1     Running   0          2m11s
nginx-deployment-7848d4b86f-xztfq   1/1     Running   0          2m11s
nginx-deployment-7848d4b86f-g6md9   1/1     Running   0          2m11s
```
For simplicity, let's put the pod name in an environment varaible.
```
export PODNAME=nginx-deployment-7848d4b86f-6qzm9
```
```
kubectl debug -it $PODNAME  --image=xxradar/hackon --copy-to=my-debugger
Defaulting debug container name to debugger-h2pdm.
If you don't see a command prompt, try pressing enter.
root@my-debugger:/#
```
At this stage the pod is copied. A new pod `my-debugger` is started and a container with the specified `image` is attached. 
You can verify this in a different terminal.
```
kubectl get po
NAME                                READY   STATUS             RESTARTS   AGE
nginx-deployment-7848d4b86f-6qzm9   1/1     Running            0          5m23s
nginx-deployment-7848d4b86f-xztfq   1/1     Running            0          5m23s
nginx-deployment-7848d4b86f-g6md9   1/1     Running            0          5m23s
my-debugger                         2/2     Running            0          2m9s
```
We only share the essential namespaces, but the most important in this example is the `netns`.<br>
```
root@my-debugger:/# curl 127.0.0.1
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
root@my-debugger:/#
```
This is great to test the application over the shared networking stack, but does not grant us access to the processes and filesystem. <br>
Sharing the process namespace can be obtained via the `--share-processes=true` flag.
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
### Usecase 2: Adding an ephemeral container to an existing pod
Ephemeral containers were introduced in Kubernetes v1.23 as beta and as such available by default. Earlier versions of K8S will require you to enable feature gates [Feature Gates for Ephemeral Containers](https://xxradar.medium.com/how-to-tcpdump-using-ephemeral-containers-in-kubernetes-d066e6855785). More about this kind of containers can be found [here](https://kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/#understanding-ephemeral-containers), but in short, ephemeral containers differ from other containers in that they lack guarantees for resources or execution, and they will never be automatically restarted, so they are not really appropriate for building applications. Also, ephemeral containers may not have ports, so fields such as ports, livenessProbe, readinessProbe are disallowed. The good news is that ephemeral containers are useful for interactive troubleshooting when `kubectl exec` is insufficient because a container has crashed or a container image doesn't include debugging utilities. <b>*In contrast with patching or creating a copy of a pod, ephemeral containers can be added to a running pod without any restart!*</b>

*Note: Ephemeral containers cannot be removed from an existing pod.*<br><br>
So let's give it a try.<br>
First, reset the previous deployment
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
The following line will add an ephemeral container to an existing pod WITHOUT re-deploying the pods
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
In the following example, we'll create an ephemeral container and share <b>the process namespace of an existing container</b> inside the pod.
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
So, we added a debug container and made it have practically full access to `--target` container!
We can push this ever further. <br>
A trick you can apply to accessing the files in the `nginx` container is via the following directory:
```
cd /proc/1/root/etc/nginx
```
```
cat nginx.conf 
...
```
When you're debug container image has for example `tcpdump` installed, you can even sniff traffic inside a running pod!
```
root@nginx-deployment-74d589986c-5bmgh:~# tcpdump -n port 80
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
15:28:29.970033 IP 192.168.228.0.42704 > 192.168.172.172.80: Flags [S], seq 3769776304, win 62587, options [mss 8941,sackOK,TS val 3815764348 ecr 0,nop,wscale 7], length 0
15:28:29.970072 IP 192.168.172.172.80 > 192.168.228.0.42704: Flags [S.], seq 3784113359, ack 3769776305, win 62503, options [mss 8941,sackOK,TS val 2005209487 ecr 3815764348,nop,wscale 7], length 0
15:28:29.970405 IP 192.168.228.0.42704 > 192.168.172.172.80: Flags [.], ack 1, win 489, options [nop,nop,TS val 3815764349 ecr 2005209487], length 0
15:28:29.970525 IP 192.168.228.0.42704 > 192.168.172.172.80: Flags [P.], seq 1:80, ack 1, win 489, options [nop,nop,TS val 3815764349 ecr 2005209487], length 79: HTTP: GET / HTTP/1.1
```

### Usecase 3: Accessing a node
The first two usecases discussed focus on debugging a troublesome pod. In some cases we might need to debug and troubleshoot the kubernetes node.
Inspecting a node typically requires SSH access and requires probably private/public keys to access the node. The node OS should also have all the debugging tools installed. But maybe, there is a simpler solution!<br><br>
First pick a node
```
kubectl get no
NAME            STATUS   ROLES                  AGE   VERSION
ip-10-1-2-12    Ready    control-plane,master   13d   v1.23.3
ip-10-1-2-180   Ready    <none>                 13d   v1.23.3
ip-10-1-2-26    Ready    <none>                 13d   v1.23.3```
```
`Kubectl debug` allows to easily spin up a pod including troubleshooting tools and gain access to the node itself.
```
kubectl debug node/ip-10-1-2-180  -it  --image xxradar/hackon
Creating debugging pod node-debugger-ip-10-1-2-180-6htp2 with container debugger on node ip-10-1-2-180.
If you don't see a command prompt, try pressing enter.
root@ip-10-1-2-180:/# ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0 225384  9048 ?        Ss   Feb14   1:02 /sbin/init
root         2  0.0  0.0      0     0 ?        S    Feb14   0:00 [kthreadd]
root         3  0.0  0.0      0     0 ?        I<   Feb14   0:00 [rcu_gp]
root         4  0.0  0.0      0     0 ?        I<   Feb14   0:00 [rcu_par_gp]
...
```
You can also access the host filesystem. The host file system is mounted in the pod under the `/host` directory.
```
root@ip-10-1-2-180:/host/etc# cd /host/etc/
root@ip-10-1-2-180:/host/etc# ls -la
total 824
drwxr-xr-x 92 root root    4096 Feb  3 06:39 .
drwxr-xr-x 23 root root    4096 Feb 14 14:51 ..
-rw-------  1 root root       0 Jan 12  2020 .pwd.lock
drwxr-xr-x  3 root root    4096 Jan 12  2020 NetworkManager
drwxr-xr-x  4 root root    4096 Jan 12  2020 X11
drwxr-xr-x  4 root root    4096 Jan 12  2020 acpi
-rw-r--r--  1 root root    3028 Jan 12  2020 adduser.conf
...
```

## Introducing EBPF and @Ciliumproject Tetragon
### What is EBPF ?
eBPF is a revolutionary technology with origins in the Linux kernel that can run sandboxed programs in an operating system kernel. It is used to safely and efficiently extend the capabilities of the kernel without requiring to change kernel source code or load kernel modules. Please checkout https://ebpf.io/ for more detailed information. By allowing to run sandboxed programs within the operating system, application developers can run eBPF programs to add additional capabilities to the operating system at runtime. A lot of new intersting projects already adopted ebpf, avoiding coding skills to get most out of it. A lot of them are situated 'security observabilty' space. A project that is super interesting by @ciliumproject is https://github.com/cilium/tetragon. As stated on the github repo: "Cilium’s new Tetragon component enables powerful realtime, eBPF-based Security Observability and Runtime Enforcement. Tetragon detects and is able to react to security-significant events, such as process execution events, system call activity, I/O activity including network & file access." This project enhances security a lot, but the project is also extremely helpfull in anaylysing, understanding and debuging pod behavior in a kubernetes environment.

### Installing tetragon
To get started, checkout the github repo. 
```
git clone https://github.com/cilium/tetragon.git
```
Installing tetragon via helm is straightforward
```
helm repo add cilium https://helm.cilium.io
helm repo update
helm install tetragon cilium/tetragon -n kube-system
kubectl rollout status -n kube-system ds/tetragon -w
```
To verify if everything is working, we can see the first results via raw output of the tetragon logs (more specifcically the stdout container log)
```
kubectl logs -n kube-system -l app.kubernetes.io/name=tetragon -c export-stdout -f
```
```
{"process_exec":{"process":{"exec_id":"aXAtMTAtMS0yLTY4OjE0NDcwODAwMDAwMDA6MTcxNzA=","pid":17170,"uid":0,"cwd":"/","binary":"/usr/sbin/nginx","flags":"procFS auid rootcwd","start_time":"2022-06-01T19:35:53.684Z","auid":0,"pod":{"namespace":"default","name":"nginx-deployment-6c8b449b8f-tpz4h","container":{"id":"containerd://a486a3173f3ef8b300e7cc970010e86f13f097c90c159362483e2a3b324f35ce","name":"nginx","image":{"id":"docker.io/library/nginx@sha256:2bcabc23b45489fb0885d69a06ba1d648aeda973fae7bb981bafbb884165e514","name":"docker.io/library/nginx:latest"},"start_time":"2022-06-01T19:35:53Z","pid":31}},"docker":"a486a3173f3ef8b300e7cc970010e86","parent_exec_id":"aXAtMTAtMS0yLTY4OjE0NDcwNDAwMDAwMDA6MTcxMjI=","refcnt":1},"parent":{"exec_id":"aXAtMTAtMS0yLTY4OjE0NDcwNDAwMDAwMDA6MTcxMjI=","pid":17122,"uid":0,"cwd":"/","binary":"/usr/sbin/nginx","flags":"procFS auid rootcwd","start_time":"2022-06-01T19:35:53.644Z","auid":0,"pod":{"namespace":"default","name":"nginx-deployment-6c8b449b8f-tpz4h","container":{"id":"containerd://a486a3173f3ef8b300e7cc970010e86f13f097c90c159362483e2a3b324f35ce","name":"nginx","image":{"id":"docker.io/library/nginx@sha256:2bcabc23b45489fb0885d69a06ba1d648aeda973fae7bb981bafbb884165e514","name":"docker.io/library/nginx:latest"},"start_time":"2022-06-01T19:35:53Z","pid":1}},"docker":"a486a3173f3ef8b300e7cc970010e86","parent_exec_id
```
### Installing tetragon-cli
```
curl -L https://github.com/cilium/tetragon/releases/download/tetragon-cli/tetragon-linux-amd64.tar.gz -o tetragon-linux-amd64.tar.gz
sudo tar -C /usr/local/bin -xzvf tetragon-linux-amd64.tar.gz
```
Let's give it a try:
```
kubectl logs -n kube-system ds/tetragon -c export-stdout -f | tetragon observe
```
### Tracingpolicies
By applying tracingpolicies, when can apply ebpf kernel probes easily. A series of examples are availble in the github repo.
For example, if want to trace which process executes which metwork connections, we can simply apply:
```
kubectl apply -f ./crds/examples/tcp-connect.yaml
``
```
kubectl logs -n kube-system ds/tetragon -c export-stdout -f | tetragon observe --namespace default 
```
```
....

```

## Recap

Let's do a recap on what you've learned in this article:

 * `kubectl debug` allows for easy debugging of pods by copying or adding an ephemeral container
 * `kubectl debug` also allows for easy debugging of the K8S nodes without the need to install additional tools.
