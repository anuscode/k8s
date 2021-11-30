# Pod 생성하기

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

# 자세 하게 확인하기 Pods
```
root@k8s-master:~# kubectl get pods -o wide
NAME                     READY   STATUS    RESTARTS   AGE     IP          NODE        NOMINATED NODE   READINESS GATES
mainui-d77bf4d8f-nvsbn   1/1     Running   0          7d17h   10.44.0.2   k8s-node1   <none>           <none>
mainui-d77bf4d8f-vdx4k   1/1     Running   0          7d17h   10.36.0.2   k8s-node2   <none>           <none>
mainui-d77bf4d8f-xq7jb   1/1     Running   0          6d20h   10.44.0.3   k8s-node1   <none>           <none>
nginx-pod                1/1     Running   0          6d18h   10.36.0.3   k8s-node2   <none>           <none>
```

# watch kubectl get nodes -o wide
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


# Pod 안에 여러개 Containers 띄우기

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

# Pod 안에 인스턴스들 확인하기
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

# Pod 안에 nginx Container로 진입하기.
```
kubectl exec multipod -c nginx-container -it -- /bin/bash
```

# Log 확인하기
```
kubectl log multipod -c nginx-container
(멀티인 경우에만 컨테이너 지정.)
```

# Pod lifecycle 
![alt text](https://github.com/anuscode/k8s/blob/master/images/k8s-pod-lifecycle.png?raw=true)

# look up pods
```
kubectl get pods (현재 네임스페이스 기준)
kubectl get pods -o wide (현재 네임스페이스 기준)
kubectl describe pod webserver (현재 네임스페이스 기준)
kubectl get pods --all-namespaces (모든 네임스페이스 기준)
```

# edit pods
```
kubectl edit pod webserver
```

# delete pods
```
kubectl delete pod webserver
kubectl delete pod --all
```

# Pod Question1
![alt text](https://github.com/anuscode/k8s/blob/master/images/pod-question1.png?raw=true)

# 레디스 yaml file 만들기
```
kubectl run redis --image=redis123 --dry-run -o yaml > redis.yaml
```

# Redis 생성 실패 디버깅 하기
```
root@k8s-master:~# kubectl create -f redis.yaml
root@k8s-master:~# kubectl describe pod redis

아래에서 디버깅 확인.

Events:
  Type     Reason     Age   From               Message
  ----     ------     ----  ----               -------
  Normal   Scheduled  13s   default-scheduler  Successfully assigned default/redis to k8s-node2
  Normal   Pulling    13s   kubelet            Pulling image "redis123"
  Warning  Failed     12s   kubelet            Failed to pull image "redis123": rpc error: code = Unknown desc = Error response from daemon: pull access denied for redis123, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
  Warning  Failed     12s   kubelet            Error: ErrImagePull
  Normal   BackOff    11s   kubelet            Back-off pulling image "redis123"
  Warning  Failed     11s   kubelet            Error: ImagePullBackOff
```


# Liveness Probe(1)

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
    livenessProbe:  # 여기가 살아있는지 측정하는 부분.
      httpGet:
        path: /
        port: 80
    initialDelaySeconds: 15
    periodSeconds: 20
    timeoutSeconds: 1
    successThreshold: 1
    failureThreshold: 3
```

# Livebess Probe(2)
- httpGet probe
지정한 IP port에 HTTP GET 요청을 보내 해당 컨테이너가 응답 하는지 확인.
200이 아닌 경우 컨테이너를 다시 시작한다.

```
livenessProbe:
  httpGet:
    path: /
    port: 80
```

- tcpSocket probe
지정된 포트에 TCP연결을 시도. 연결되지 않으면 컨테이너를 다시 시작한다.

```
livenessProbe:
  tcpSocket:
    port: 22
```

- exec probe: exec
명령을 전달하고 명령의 종료 코드가 0이 아니면 컨테이너를 다시 시작한다.

```
livenessProbe:
  exec:
    command:
    - ls
    - /data/file

```

# Init Container
- 앱 컨테이너 실행 전에 미리 동작시킬 컨테이너
- 본 Container가 실행되기 전에 사전 작업이 필요 할 경우 사용
- '초기화 컨테이너가 모두 실행 된 후에 앱 컨테이너를 실행'
- https://kubernetes.io/ko/docs/concepts/workloads/pods/init-containers/

# Init Container 실행 해보기

```
$ kubectl create -f init-container.yaml

apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup myservice.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for myservice; sleep 2; done"]
  - name: init-mydb
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup mydb.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for mydb; sleep 2; done"]
```

```
NAME                     READY   STATUS     RESTARTS   AGE   IP          NODE        NOMINATED NODE   READINESS GATES
mainui-d77bf4d8f-6wnrm   1/1     Running    0          89m   10.44.0.3   k8s-node1   <none>           <none>
mainui-d77bf4d8f-7xt22   1/1     Running    0          89m   10.44.0.1   k8s-node1   <none>           <none>
mainui-d77bf4d8f-vncc5   1/1     Running    0          89m   10.44.0.2   k8s-node1   <none>           <none>
multipod                 1/1     Running    0          34m   10.36.0.1   k8s-node2   <none>           <none>
myapp-pod                0/1     Init:0/2   0          7s    10.36.0.2   k8s-node2   <none>           <none>
```

# Infra container(pause) 이해하기
- Pod 마다 Pause container (hidden) 이 있음.
- 여러개의 컨테이너를 가진 Pod 도 한개를 가지고 있음
- 아이피나 호스트네임 같은걸 관리 및 생성 하는 컨테이너

# Static pod 만들기
- 일반적으로는 k8s API 에 요청
- 하지만 스태틱 pod의 경우 노드에 있는 kubelet 이 관리하는 staticPodPath 에
  yaml 파일을 직접 넣으면 생성 해줌.
- kublet daemon에 의해 실행 되는 pod를 static pod 라고 부름.
- /var/lib/kubelet/config.yaml 조회 시 staticPodPath 에 기재 되어 있음.

```
... 생략

runtimeRequestTimeout: 0s
shutdownGracePeriod: 0s
shutdownGracePeriodCriticalPods: 0s
staticPodPath: /etc/kubernetes/manifests
streamingConnectionIdleTimeout: 0s
syncFrequency: 0s
volumeStatsAggPeriod: 0s
```


# pod 에 Resource 할당

```

apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod-resource
spec:
  containers:
  - name: myapp-container
    image: nginx:1.14
    ports:
    - containerPort: 80
          protocol: TCP
    
    # 여기서 리소스 할당.
    resources:
      requests:
        cpu: 200m  # 1000m == 1core 
        memory: 250Mi
      limits:
        cpu: 1
        memory: 500Mi
     
```


# 환경변수 주입하기
- 도커에서 미리 지정한 버전 같은 것들을 대체 하고 싶을 때.
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod-env
spec:
  containers:
  - name: myapp-container
    image: nginx:1.14
    ports:
    - containerPort: 80
      protocol: TCP
    
    # 여기서 리소스 할당.
    env:
      - name: MYVAR
        value: "testvalue"
```
```
kubectl exec nginx-pod-env -it -- /bin/bash
env
```

# ReplicationController
- 요구하는 Pod의 개수를 보장하며 파드 집합의 실행을 항상 안정적으로 유지 하는 것을 목표.
- 요구하는 pod의 개수보다 부족하면 템플릿을 이용해 팟을 추가
- 요구하는 팟 수보다 많으면 최근에 생성 된 팟을 삭제.
- 기본구성
  - selector
  - replicas
  - template

```
apiVersion: v1
kind: ReplicationController
metadata:
  name: <RC name>
spec:
  replicas: 5
  selector:
    app: web-ui
  template:
    metadata:
      name: nginx-pod
      lables:
        app: webui  # selector 에 있는 label과 무조건 일치 해야하는게 있어야 함.
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.14 
```
- 쿠버야 nginx 웹 서버 3개 실행해줘!
- `kubectl create rc-exam --image=nginx --replicas=3 --selector=app=webui`
  - app=webui 라벨이 달린 애들에 대해서 관리
  - 개수 보장은 Controller 가 담당함.

- scale up/down
  - `kubectl edit rc rc-nginx` 로 edit 가능
  - `kubectl scale rc rc-nginx --relicas=3`

# ReplicaSet
- ReplicationController 와 같은 역할을 하는 컨트롤러
- ReplicationController 보다 풍부한 selector
```
selector:
  matchLables::
    component: redis
  matchExpressions:
    - {key: tier, operator: In, Values:[cache]}
    - {key: environment, operator: NotIn, values: [dev]}
```
- matchExpressions 연산자
  - In: key 와 values 를 지정하여 key, value 가 일치하는 Pod만 연결
  - NotIn: key는 일치하고 value는 일치하지 않는 Pod에 연결
  - Exists: key에 맞는 label의 pod를 연결
  - DoesNotExists: key와 다른 label의 pod를 연결

- ReplicationController
```
spec:
  replicas: 5
  selector:
    app: web-ui
    version: "2.1"
```
- ReplicaSet
```
spec:
  replicas: 3
  selector:
    matchLables:
      app: webui
    matchExpressions:
    - {key: version, operator: In, value: ["2.1"]}  # matchExpressions 참조.
```
- replicaset 상태 확인
```
kubectl get replicaset
NAME         DESIRED     CURRENT    READY    AGE
rs-nginx     3           3          3        46s
```
- replicaset 삭제 Pod 도 같이 삭제
```
kubectl delete replicaset rs-nignx
```

- replicaset 만 삭제 Pod 는 No 삭제
```
kubectl delete replicaset rs-nignx --cascade=False
```

# Deployment

- ReplicaSet을 컨트롤해서 Pod 수를 조절.
- Rolling Update & Rolling Back
- ReplicaSet vs Deployment definition
```
apiVersion: apps/v1              |       apiVersion: apps/v1           
kind: ReplicaSet                 |       kind: Deployment              
metadata:                        |       metadata:                     
  name: rs-nignx                 |       name: deploy-nginx           
spec:                            |       spec:                         
  replicas: 5                    |         replicas: 5                 
  selector:                      |         selector:                   
    app: web-ui                  |           app: web-ui               
  selector:                      |         selector:                   
    matchLables:                 |           matchLables:              
      app: webui                 |             app: webui              
  template:                      |         template:                   
    metadata:                    |           metadata:                 
      name: nginx-pod            |             name: nginx-pod         
      lables:                    |             lables:                 
        app: webui               |               app: webui            
    spec:                        |           spec:                     
      containers:                |             containers:             
      - name: nginx-container    |             - name: nginx-container 
        image: nginx:1.14        |               image: nginx:1.14     
```
- Rolling update 모드만 빼면 ReplicaSet 과 exactly 똑같다.

- Rolling update
  - kubectl set image deployment <deploy_name> <container_name> = <new_version_image>
- 
- Roll back
  - kubectl rollout history deployment <deploy_name>
  - kubectl rollout undo deploy <deploy_name>
- 