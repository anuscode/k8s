1. Pod 생성하기

- kubectl run 명령으로 생성
```
$kubectl run webserver --image==nginx:1.14
```

- pod yaml 을 이용해 생성

```
root@k8s-master:~# kubectl run web1 --image=nginx:1.14 --port=80 -o yaml --dry-run=client

apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: web1
  name: web1
spec:
  containers:
  - image: nginx:1.14
    name: web1
    ports:
    - containerPort: 80
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}

```
```
yaml 파일을 이용하여 생성 시 create 사용.
$ kubectl create -f pod-nginx.yaml
```

- 자세 하게 확인하기 Pods
```
root@k8s-master:~# kubectl get pods -o wide
NAME                     READY   STATUS    RESTARTS   AGE     IP          NODE        NOMINATED NODE   READINESS GATES
mainui-d77bf4d8f-nvsbn   1/1     Running   0          7d17h   10.44.0.2   k8s-node1   <none>           <none>
mainui-d77bf4d8f-vdx4k   1/1     Running   0          7d17h   10.36.0.2   k8s-node2   <none>           <none>
mainui-d77bf4d8f-xq7jb   1/1     Running   0          6d20h   10.44.0.3   k8s-node1   <none>           <none>
nginx-pod                1/1     Running   0          6d18h   10.36.0.3   k8s-node2   <none>           <none>
```

- watch kubectl get nodes -o wide
```
Every 2.0s: kubectl get nodes -o wide                                                            k8s-master: Sun Oct  3 14:13:51 2021

NAME         STATUS   ROLES                  AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION    CON
TAINER-RUNTIME
k8s-master   Ready    control-plane,master   7d23h   v1.22.2   10.128.0.3    <none>        Ubuntu 20.04.3 LTS   5.11.0-1018-gcp   doc
ker://20.10.8
k8s-node1    Ready    <none>                 7d22h   v1.22.2   10.128.0.4    <none>        Ubuntu 20.04.3 LTS   5.11.0-1018-gcp   doc
ker://20.10.8
k8s-node2    Ready    <none>                 7d22h   v1.22.2   10.128.0.5    <none>        Ubuntu 20.04.3 LTS   5.11.0-1018-gcp   doc
ker://20.10.8
```


-- Pod 안에 여러개 인스턴스 띄우기

```
apiVersion: v1
kind: Pod
metadata:
  name: multipod
spec:
  containers:
  - image: nginx:1.14
    name: nginx-container
    ports:
    - containerPort: 80
  - name: centos-container
    image: centos:7
    command:
    - sleep
    - "10000"
```

-- Pod 안에 인스턴스들 확인하기
```
root@k8s-master:~# kubectl describe pod multipod

Name:         multipod
Namespace:    default
Priority:     0
Node:         k8s-node2/10.128.0.5
Start Time:   Sun, 03 Oct 2021 14:21:18 +0000
Labels:       <none>
Annotations:  <none>
Status:       Running
IP:           10.36.0.1
IPs:
  IP:  10.36.0.1
Containers:
  web1:
    Container ID:   docker://da92ac82c5f08a3109280d0523251a3000ba2c0e6f66ae1b2ff727db96338c44
    Image:          nginx:1.14
    Image ID:       docker-pullable://nginx@sha256:f7988fb6c02e0ce69257d9bd9cf37ae20a60f1df7563c3a2a6abe24160306b8d
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running

 생략..
```

- Pod 안에 nginx Container로 진입하기.
```
kubectl exec multipod -c nginx-container -it -- /bin/bash
```

- Log 확인하기
```
kubectl log multipod -c nginx-container
(멀티인 경우에만 컨테이너 지정.)
```