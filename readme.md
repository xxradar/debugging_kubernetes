### Essential POD behaviour

Pods are the fundamental building block of Kubernetes applications. 
Pods are disposable and replaceabl. You cannot add a container to a running Pod for examplel 
When you try to update or patch running pod, the pod is terminated and a new one is deployed.

This behavior can be illustrated.
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

```
```
% kubectl get po --selector app=nginx
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-7848d4b86f-sp6q5   1/1     Running   0          19s
nginx-deployment-7848d4b86f-672tn   1/1     Running   0          19s
nginx-deployment-7848d4b86f-w22ft   1/1     Running   0          19s
```
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
% kubectl patch deploy nginx-deployment --patch "$(cat patch.yaml)"
```
```
 kubectl get po --selector app=nginx

NAME                                READY   STATUS        RESTARTS   AGE
nginx-deployment-89cfdb59c-x7z94    2/2     Running       0          20s
nginx-deployment-89cfdb59c-4zzjl    2/2     Running       0          16s
nginx-deployment-7848d4b86f-672tn   0/1     Terminating   0          64s
nginx-deployment-89cfdb59c-mdr89    2/2     Running       0          12s

```

Sometimes it's necessary to inspect the state of an existing Pod, however, for example to troubleshoot a hard-to-reproduce bug. In these cases you can run an ephemeral container in an existing Pod to inspect its state and run arbitrary commands.
What is an ephemeral con
