# kubernetes-docker-infra
infra with kubernetes, docker


## /Vagrantfile_1: 단일 CentOs 구성 Vagrantfile

## /Vagrantfile_2: 다중(3걔) CentOs 구성 Vagrantfile

## /Vagrantfile_3: Kubernetes Master Node 1개, Worker Node 3개

#### kubelet : 쿠버네티스에서 파드의 생성과 상태 관리 및 복구 등을 담당하는 구성 요소
kubelet 서비스를 종료 시키는 경우 pod 생성 관리등이 정상적으로 작동하지 않음을 마스터노드에서 볼 수 있음

#### kube-proxy: kube-proxy는 파트의 통신을 담당함. 

#### kubernetes

##### 파드를 실행하는 방법
```shell
# kubectl run을 통해서 파드를 실행
kubectl run nginx-pod --image=nginx
```

```shell
kubectl create nginx --image=nginx

Error: unknown flag: --image
See 'kubectl create --help' for usage. 
# create에는 이미지 옵션이 없음.

# create에 이미지를 지정하기 위해서는 deployment로 생성해야 함
kubectl create deployment <deployment name> --image=nginx
```

```shell
# deployment의 pod를 3개로 증가 시킴
kubectl scale deployment <deployment name> --replicas=3
```
3개의 nginx pod를 디플로이먼트 오브젝트로 만드는 방법
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-hname
  labels:
    app: nginx
spec:
  replicas: 6
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: echo-hname
        image: sysnet4admin/echo-hname
```
```shell
kubectl create -f xxxx.yml
```

yaml 파일을 통해서 파드를 늘리는 방법
```shell
kubectl create -f xxxx.yml

Error from server (AlreadyExists): error when creating "/root/_Book_k8sInfra/ch3/3.2.4/echo-hname.yaml": deployments.apps "echo-hname" already 
# 이미 있으므로 만들 수 없다는 경고가 뜸
```

apply 통해서 한다면
```shell
kubectl apply -f xxxx.yml

Warning: kubectl apply should be used on resource created by either kubectl create --save-config or kubectl apply
deployment.apps/echo-hname configured

#처음부터 apply로 생성한 것이 아니어서 경고가 뜸. 작동에는 이상없지만 일관성에 문제가 생김
```

따라서 변경 가능성이 있는 복잡한 오브젝트는 파일로 작성한 후 apply로 적용하는 것이 좋고, 애드훅으로 오브젝트를 생성할 떄는 create로 사용하는 것이 좋음

| 구분     | RUN  | CREATE | APPLY    |
|--------|------|--------|----------|
| 명령 실행  | 제한적  | 가능함    | 안 됨      |
| 파일 실행  | 안 됨  | 가능함    | 가능함      |
| 변경 가능  | 안 됨  | 안 됨    | 가능함      |
| 실행 편의성 | 매우 좋음 | 매우 좋음  | 좋음       |
| 기능 유지  | 제한적  | 지원함    | 다양하게 지원함 |


##### 파드의 컨테이너 자동 복구

```shell
kubectl delete pods nginx-pods
# 디플로이먼트에 속한 파드도 아니고 어떤 컨트롤러도 이 파드를 관리 하지 않음 => 바로 삭제되고 바로 생성됨

kubectl delete pods echo-hname-7894b67f-24r57
# 디플로이먼트에 의해 관리 되고 있으므로 다른 이름으로 재 생성됨(deployment에서 replicas =6으로 선언했기 때문에 6개로 맞추기 위해 서)

kubectl delete deployment echo-hname
# 디플로이먼트에 속한 파드는 상위 디플로이먼트를 삭제해야 파드가 삭제됨.
``` 

##### 노드 자원 보호하기

```shell
# get pods 커스터마이징을 통해서 가져오기
kubectl get pods -o=custom-columns=NAME:.metadata.name,IP:.status.podIP,STATUS:.status.phase,NODE:.spec.nodeName
NAME


NAME                        IP               STATUS    NODE
echo-hname-7894b67f-65jfn   172.16.103.133   Running   w2-k8s
echo-hname-7894b67f-ckpfk   172.16.103.135   Running   w2-k8s
echo-hname-7894b67f-gjcsb   172.16.221.136   Running   w1-k8s
echo-hname-7894b67f-gpdmp   172.16.132.9     Running   w3-k8s
echo-hname-7894b67f-h5qfs   172.16.221.135   Running   w1-k8s
echo-hname-7894b67f-jsb7r   172.16.132.8     Running   w3-k8s
echo-hname-7894b67f-pn8gs   172.16.103.134   Running   w2-k8s
echo-hname-7894b67f-rks7w   172.16.132.7     Running   w3-k8s
echo-hname-7894b67f-vtlzb   172.16.221.137   Running   w1-k8s
```

```shell
# 노드에 문제가 자주 발생하는 경우 현재 상태를 보존해야할때 cordon 명령어를 실행함
kubectl cordon w3-k8s

node/w3-k8s cordoned

# w3-k8s에는 더이상 파드가 할당되지 않는 상태로 변경됨. 
kubectl get nodes

NAME     STATUS                     ROLES    AGE   VERSION
m-k8s    Ready                      master   45h   v1.18.4
w1-k8s   Ready                      <none>   44h   v1.18.4
w2-k8s   Ready                      <none>   44h   v1.18.4
w3-k8s   Ready,SchedulingDisabled   <none>   44h   v1.18.4

# pod 수를 증량시키는 경우 w3-k8s에서는 제외됨

NAME                        IP               STATUS    NODE
echo-hname-7894b67f-65jfn   172.16.103.133   Running   w2-k8s
echo-hname-7894b67f-744ng   172.16.221.138   Running   w1-k8s
echo-hname-7894b67f-cv4jn   172.16.103.137   Running   w2-k8s
echo-hname-7894b67f-dw64h   172.16.221.140   Running   w1-k8s
echo-hname-7894b67f-dxrq2   172.16.221.139   Running   w1-k8s
echo-hname-7894b67f-h5qfs   172.16.221.135   Running   w1-k8s
echo-hname-7894b67f-nthtd   172.16.103.136   Running   w2-k8s
echo-hname-7894b67f-rks7w   172.16.132.7     Running   w3-k8s
echo-hname-7894b67f-xgx47   172.16.103.138   Running   w2-k8s

# uncordon을 통해서 할당이 다시 되도록 할 수 있음
kubectl uncordon w3-k8s

node/w3-k8s uncordoned
```

##### 노드 유지보수
유지보수를 위해 노드를 꺼야하는 상황이 발생함. drain 기능을 통해 지정된 노드의 파드를 전부 다른 곳으로 이동시킬 수 있음

pod는 언제든 삭제되고 다시 생성할 수 있기 떄문에 실제로는 옮기지는 않고 새로 생성함

하지만 DaemontSet은 노드에 하나만 존재하는 파드라서 drain으로는 삭제할 수 없음

```shell
# drain 실행
kubectl drain w3-k8s --ignore-daemonsets

WARNING: ignoring DaemonSet-managed Pods: kube-system/calico-node-4tclx, kube-system/kube-proxy-z7pnv
evicting pod default/echo-hname-7894b67f-rks7w
pod/echo-hname-7894b67f-rks7w evicted
node/w3-k8s evicted
```

```shell
# drain을 사용하면 cordon명령어를 실행된 것 처럼 바뀜
NAME     STATUS                     ROLES    AGE   VERSION
m-k8s    Ready                      master   45h   v1.18.4
w1-k8s   Ready                      <none>   45h   v1.18.4
w2-k8s   Ready                      <none>   45h   v1.18.4
w3-k8s   Ready,SchedulingDisabled   <none>   45h   v1.18.4
```

##### pod 업데이트하고 복구
```shell
# 파드 업데이트
kubectl apply -f ~/_Book_k8sInfra/ch3/3.2.10/rollout-nginx.yaml  --record
# record 옵션을 사용하면 배포한 정보의 히스토리를 기록함.

kubectl rollout history deployment rollout-nginx
# rollout history 명령어를 통해서 히스토리를 확인할 수 있음
deployment.apps/rollout-nginx
REVISION  CHANGE-CAUSE
1         kubectl apply --filename=/root/_Book_k8sInfra/ch3/3.2.10/rollout-nginx.yaml --record=true


# 파드 업데이트
 kubectl set image deployment rollout-nginx nginx=nginx:1.16.0 --record
# 파드는 하나하나 재생성함. 따라서 이름과 IP가 변경됨. 업데이트는 전체의 25%씩 임. 최소 값은 1개

# deployment 상태 확인
kubectl rollout status deployment rollout-nginx

deployment.apps/rollout-nginx
REVISION  CHANGE-CAUSE
1         kubectl apply --filename=/root/_Book_k8sInfra/ch3/3.2.10/rollout-nginx.yaml --record=true
2         kubectl set image deployment rollout-nginx nginx=nginx:1.16.0 --record=true

# 실패 상황 만들기 nginx 컨테이너를 버젼 1.17.23으로 변경
kubectl set image deployment rollout-nginx nginx=nginx:1.17.23 --record
deployment.apps/rollout-nginx image updated

# Pending 상태에서 정지 되어 있음
kubectl get pods -o=custom-columns=NAME:.metadata.name,IP:.status.podIP,STATUS:.status.phase,NODE:.spec.nodeName

NAME                             IP               STATUS    NODE
rollout-nginx-8566d57f75-4gtzs   172.16.221.143   Running   w1-k8s
rollout-nginx-8566d57f75-z9lkg   172.16.103.141   Running   w2-k8s
rollout-nginx-8566d57f75-zv2tj   172.16.103.142   Running   w2-k8s
rollout-nginx-856f4c79c9-wbx8j   172.16.132.10    Pendin

# rollout status로 확인한 상태
kubectl rollout status deployment rollout-nginx
Waiting for deployment "rollout-nginx" rollout to finish: 1 out of 3 new replicas have been updated...

# 조금더 기다리면 시도후 생성 실패했다고 뜬다
kubectl rollout status deployment rollout-nginx
error: deployment "rollout-nginx" exceeded its progress deadline

# describe 명령어로 확인
kubectl describe deployment rollout-nginx

... 
OldReplicaSets:  rollout-nginx-8566d57f75 (3/3 replicas created)
NewReplicaSet:   rollout-nginx-856f4c79c9 (1/1 replicas created)
...
# 새로 생성 되는 과정에서 멈춰 있는 것을 확인할 수 있음.

# 업데이트 할 때 사용했던 명령어들을 rollout history로 확인
kubectl rollout history deployment rollout-nginx
deployment.apps/rollout-nginx
REVISION  CHANGE-CAUSE
1         kubectl apply --filename=/root/_Book_k8sInfra/ch3/3.2.10/rollout-nginx.yaml --record=true
2         kubectl set image deployment rollout-nginx nginx=nginx:1.16.0 --record=true
3         kubectl set image deployment rollout-nginx nginx=nginx:1.17.23 --record=true

# 마지막 단계(rev 3)에서 전 단계(rev 2)로 되돌리면 파드가 정상으로 돌고 있음을 확인할 수 있음
kubectl get pods -o=custom-columns=NAME:.metadata.name,IP:.status.podIP,STATUS:.status.phase,NODE:.spec.nodeName

NAME                             IP               STATUS    NODE
rollout-nginx-8566d57f75-4gtzs   172.16.221.143   Running   w1-k8s
rollout-nginx-8566d57f75-z9lkg   172.16.103.141   Running   w2-k8s
rollout-nginx-8566d57f75-zv2tj   172.16.103.142   Running   w2-k8s

# rollout history로 확인하면 다음과 같으며
kubectl rollout history deployment rollout-nginx

deployment.apps/rollout-nginx
REVISION  CHANGE-CAUSE
1         kubectl apply --filename=/root/_Book_k8sInfra/ch3/3.2.10/rollout-nginx.yaml --record=true
3         kubectl set image deployment rollout-nginx nginx=nginx:1.17.23 --record=true
4         kubectl set image deployment rollout-nginx nginx=nginx:1.16.0 --record=true

# rev2로 되돌렸기 떄문에 rev2는 삭제되고 rev4가 추가되있음. 그리고 가장 최근 상태는 rev4임

#특정 rev로 돌아가는 방법
kubectl rollout undo deployment rollout-nginx --to-revision=1
```

