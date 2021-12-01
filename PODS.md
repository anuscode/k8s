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

- Rolling update 과정 이미지
- 아래 과정을 오른쪽 한개 생성 왼쪽 한개 삭제 식으로 진행 후 ReplicaSet xxx 를 삭제함.
![alt text](https://github.com/anuscode/k8s/blob/master/images/deployment_rolling_update.png?raw=true)

- Rolling update
  - kubectl set image deployment <deploy_name> <container_name> = <new_version_image>

- Roll back
  - kubectl rollout history deployment <deploy_name>
  - kubectl rollout undo deploy <deploy_name>

- Deployment Rolling update & Rolling Back (1)
```
$ kubectl create -f deployment-exam1.yaml --record
$ kubectl get deployment,rs,pod,svc

application rollingUpdate
$ kubectl set image deploy app-deploy web=nginx:1.15 --record
$ kubectl set image deploy app-deploy web=nginx:1.16 --record
$ kubectl set image deploy app-deploy web=nginx:1.17 --record

controlling rolling update
$ kubectl rollout pause deploy app-deploy
$ kubectl rollout resume deploy app-deploy
$ kubectl rollout status deploy app-deploy

application rollback
$ kubectl rollout history deployment app-deploy
$ kubectl rollout undo deployment app-deploy --to-revision=3
$ kubectl rollout undo deployment app-deploy

```

- Deployment Rolling update & Rolling Back (2)
```
apiVersion: apps/v1                
kind: Deployment                                       $ kubectl apply -f deployment-exam2.yaml
metadata:                                              $ kubectl get deployment,rs,pod,svc
  name: deploy-nginx                                   이미지 업데이트
  `annotations:`                                       $ vi deployment-exam2.yaml
    `kubernetes.io/change-cause: version 1.14`         annotations:     
spec:                                                    kubernetes.io/change-cause: version 1.15
  `progressDeadlineSeconds: 600`                       ...
  `revisionHistoryLimit: 10`                           spec:
  `stratege:`                                            containers:
    `rollingUpdate:`                                     - name: web
      `maxSurge: 25%`                                      image: nginx:1.15
      `maxUnavailable: 25%`
    `type: RollingUpdate`                              $ kubectl apply -f deployment-exam2.yaml
  replicas: 5                      
  selector:                                            이미지 롤백
    matchLables:                                       $ kubectl rollout history deployment app/v1 
      app: webui                                       $ kubectl rollout undo deployment app/v1
  template:                        
    metadata:                      
      name: nginx-pod              
      lables:                      
        app: webui                 
    spec:                          
      containers:                  
      - name: nginx-container      
        image: nginx:1.14          
```

# DaemonSet
- 전체 노드에서 노드당 Pod 가 한 개씩 실행되도록 보장
- 로그 수집기, 모니터링 에이전트와 같은 프로그램 실행 시 적용
![alt text](./images/daemon_set.png)

- ReplicaSet vs DaemonSet definition
```
apiVersion: apps/v1              |       apiVersion: apps/v1           
`kind: ReplicaSet`               |       `kind: DaemonSet`              
metadata:                        |       metadata:                     
  name: rs-nignx                 |         name: daemonset-nginx           
spec:                            |       spec:                         
  `replicas: 5`                  |                                      
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

- daemonSet 을 업데이트 하는 법.
```
kubectl edit daemonsets daemonset-nginx
```
```
apiVersion: apps/v1           
kind: DaemonSet          
metadata:                     
  name: daemonset-nginx         
spec:                         
                              
  selector:                   
    matchLables:              
      app: webui              
  template:                   
    metadata:                 
      name: nginx-pod         
      lables:                 
        app: webui            
    spec:                     
      containers:             
      - name: nginx-container 
        image: nginx: 1.15 # 1.14 -> 1.15 로 변경     
```

- daemonSet 을 rollback 하는 법.
```
kubectl rollout undo daemonset daemonset-nginx
```

# StatefulSet
- Pod의 상태를 유지해주는 컨트롤러
  - Pod의 이름
  - Pod의 볼륨 (스토리지)

- ReplicaSet vs StatefulSet yaml 비교
```                                                                     
apiVersion: apps/v1              |       apiVersion: apps/v1            
`kind: ReplicaSet`               |       `kind: StatefulSet`              
metadata:                        |       metadata:                      
  name: rs-nignx                 |         name: sf-nginx        
spec:                            |       spec:                          
  replicas: 5                    |         replicas: 5
                                 |         `serviceName: sf-nginx-service`
                                 |         podManagementPolicy: Parallel (OrderedReady 가 default 값임)          
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
- StatefulSet 을 업데이트 하는 법.
```
kubectl edit statefulset sf-nginx
```
```
apiVersion: apps/v1           
kind: DaemonSet          
metadata:                     
  name: daemonset-nginx         
spec:                         
                              
  selector:                   
    matchLables:              
      app: webui              
  template:                   
    metadata:                 
      name: nginx-pod         
      lables:                 
        app: webui            
    spec:                     
      containers:             
      - name: nginx-container 
        image: nginx: 1.15 # 1.14 -> 1.15 로 변경     
```

- StatefulSet 을 rollback 하는 법.
```
kubectl rollout undo statefulset sf-nginx
```

# Job Controller
- Kubernetes는 Pod를 running 중인 상태로 유지
- Batch 처리하는 Pod는 작업이 완료되면 종료 됨
- Batch 처리에 적합한 컨트롤러로 Pod의 성공적인 완료를 보장
  - 비정상 종료 시 다시 실행
  - 정상 종료 시 완료
```
apiVersion: batch/v1           
kind: Job          
metadata:                     
  name: centos-job         
spec:             
#  completions: 5
#  parallelism: 2
#  activeDeadlineSeconds: 5
  template:                   
    spec:
      containers:
      - name: centos-container
        image: centos:7
        command: ["bash"]                
        args:
        - "-c"
        - "echo 'Hello World'; sleep 50; echo 'Bye'"
        # restartPolicy: Never
        restartPolicy: OnFailure
  backoffLimit: 3    
```

# CronJob
- job 컨트롤러로 실행할 어플리케이션 Pod를 주기적으로 반복해서 실행
- Linux의 크론잡의 스케줄링 기능을 잡 컨트롤러에 추가한 api
- 다음과 같은 반복해서 실행하는 잡을 운영해야 할 때 사용
  - 데이터 백업
  - 이메일 발송
  - Cleaning tasks

- Cronjob Schedule: "0 3 1 * *"
  - Minutes (from 0 to 59)
  - Hours (from 0 to 23)
  - Day of the month (from 1 to 31)
  - Month (from 1 to 12)
  - Day of the week (from 0 to 6, 1-5 는 주중)

- Job vs Cronjob
```
apiVersion: batch/v1                                  |  apiVersion: batch/v1                                
kind: Job                                             |  kind: CronJob                                           
metadata:                                             |  metadata:                                           
  name: centos-job                                    |    name: cronjob-definition
                                                      |  spec:
                                                      |    schedule: "0 3 1 * *"  
                                                      |    jobTemplate:
spec:                                                 |      spec:                                               
  template:                                           |        template:                                         
    spec:                                             |          spec:                                           
      containers:                                     |            containers:                                   
      - name: centos-container                        |            - name: centos-container                      
        image: centos:7                               |              image: centos:7                             
        command: ["bash"]                             |              command: ["bash"]                           
        args:                                         |              args:                                       
        - "-c"                                        |              - "-c"                                      
        - "echo 'Hello World'; sleep 50; echo 'Bye'"  |              - "echo 'Hello World'; sleep 50; echo 'Bye'"
```


# Service
```
apiVersion: v1
kind: Service
metadata:
  name: webui-svc
spec:
  clusterIP: 10.96.100.100
  selector:
    `app: webui`
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```