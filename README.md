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
