# [Coursera] Architecting with Google Kubernetes Engine: Workloads

## The kubectl command
> kubectl 커맨드가 처리되는 과정
- `kubectl`은 command를 API call로 변경 
- `master`의 `kube-apiserver`가 API call을 받고 `etcd`를 질의하여 요청을 처리
- kube-apiserver가 결과를 https로 전달
![./images/kubectl.png](./images/kubectl.png)
- config
~~~bash
$ $HOME/.kube/config # target cluster name, credentials
$ kubectl config view
$ gcloud container clusters \
  get-credentials [CLUSTER_NAME] \
  --zone [ZONE_NAME]
~~~
> .kube/config가 설정된 이후에는 kubectl 커맨드가 credential을 묻지 않고 default cluster에 연결가능
- kubectl syntax
~~~bash
# kubectl [command] [type] [name] [flags]
$ kubectl get pods my-test-app -o=yaml
$ kubectl describe pod [POD_NAME]
$ kubectl exec [POD_NAME] ps aux
$ kubectl exec [POD_NAME] env
$ kubectl exec -it [POD_NAME] -- [command]
$ kubectl exec -it demo -- /bin/bash
~~~

command: get, describe, logs, exec ...
type: pod, deployents, nodes ...
name: object name
flags: -o=yaml, -o=wide ..
> TIP! -o=yaml 옵션은 kubernetes object를 다시 만들 때 유용

## Introspection
- The act of gathering information about containers, pods, services, and other engines running within the cluster
- Pod phases: pending, running, succeeded, failed, unknown
> What is most common reason for a Pod to report `CrashLoopBackOff` as its state? The Pod's configuration is not correct.

## Pod Networking
- VPC cloud range 내 1 Pod 1 IP addr (alias IP)
> Pod가 독립된 IP 주로를 가지기 때문에 Pod끼리 통신이 가능
`VPC`s are locally isolated networks that provides connectivity for resource you deploy with GCP, such as kubernetes cluster, compute engine instance, and App Engine Flex instances
> IP가 부족하지 않도록  클러스터 내 통용되는 IP로 4000개를 확보하고 Pod를 위한 IP range를 별개로 부여함. /14 로 부여하면 25만개 IP address인데 실제로는 하나의 클러스터에서 25만개 IP를 쓸 일이 거의 없다고 가정하여 한 노드에는 250개 정도의 IP만 사용하도록 제한하여 1000 노드를 쓸 수 있도록 함. 1000노드 100 Pod가 기본값
![./images/pod_alias_ip.png](./images/pod_alias_ip.png)

## Service
- In ever-changing container amendments, services give pods a `stable IP address` and name that remains the same through updates, upgrades, scalability changes, and even pod failures
> Service가 Pod 앞에 있으면서 Pod에게 IP를 부여하고 트래픽을 전달하는데 Pod는 깨진 다음에 다시 생성될 때 새로운 IP를 가지기 때문에 Service쪽에 static IP를 부여. 
>Pod를 Service와 엮는 방법으로 `label selector` 사용
![./images/service_vip.png](./images/service_vip.png)
- Find services: Environment Variables, Kubernetes DNS add on for service discovery, Istio

## Service Types and Load Balencers
- `cluster IP`: static IP address for `internal` communications within a cluster
- `node port`: Enable external communication
- `load balencer`: Exposing services outside a cluster. Inbound access from outside the cluster into the service. To eliminate `double hop`, set `externalTrafficPolicy: Local`
~~~yaml
apiVersion: v1
kind: Service
metadata:
    name: my-service
spec:
    type: ClusterIP
    selector:
        app: Backend
    ports:
        - protocol: TCP
          port: 80
          targetPort: 9376
~~~
![./images/service_type.png](./images/service_type.png)
> 더블 홉 딜레마: HTTP 로드 밸런서가 노드를 랜덤으로 선택하여 트래픽을 분산하고 노드가 kube proxy를 사용해서 Pod를 (공평하게) 선택하는 과정에서 선택한 Pod가 다른 노드에 있는 경우 트래픽을 보내고 응답을 받는 과정에서 트래픽이 낭비되는 현상. 이 현상을 방지하기 위한 옵션을 사용하거나 Container-native load balancer를 사용한다.

## Ingress resource
- To provide external access to one or more Services
- Collection of rules that direct external inbound connections to a set of services within the cluster (kind of a service for services)
- The traffic doesn't match any of these host-based or path-based rules, it simply send to the default backend.
> spec에 특정 host와 path가 어떤 service:port로 가야 하는지에 대한 rule을 기록
~~~bash
$ kubectl edit ingress [NAME]
$ kubectl replace -f [FILE]
~~~

## Lab - Creating a GKE Cluster via Cloud Shell
~~~bash
$ export my_zone=us-central1-a
$ export my_cluster=standard-cluster-1
$ gcloud container clusters create $my_cluster --num-nodes 3 --zone $my_zone --enable-ip-alias
$ kubectl cluster-info
Kubernetes master is running at https://35.224.225.105
GLBCDefaultBackend is running at https://35.224.225.105/api/v1/namespaces/kube-system/services/default-http-backend:http/proxy
Heapster is running at https://35.224.225.105/api/v1/namespaces/kube-system/services/heapster/proxy
KubeDNS is running at https://35.224.225.105/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://35.224.225.105/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
$ kubectl config get-contexts
                   NAMESPACE
*         gke_qwiklabs-gcp-fab7f79387782b8f_us-central1-a_standard-cluster-1   gke_qwiklabs-gcp-fab
7f79387782b8f_us-centr
$ kubectl top nodes
NAME                                                CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
gke-standard-cluster-1-default-pool-60988e96-5lbm   51m          5%     534Mi           20%
gke-standard-cluster-1-default-pool-60988e96-634p   44m          4%     570Mi           21%
gke-standard-cluster-1-default-pool-60988e96-qjkf   56m          5%     536Mi           20%
~~~

## Lab - Upgrading Kubernetes Engine Clusters
> 쿠버네티스 버전이 낮은 경우 Clusters > Details 메뉴의 master version 옆에 `upgrade avarilable` 메뉴가 떠서 클릭하면 업그레이드됨

## Lab - Creating Kubernetes Engine Deployments (*)
~~~
git clone https://github.com/GoogleCloudPlatformTraining/training-data-analyst
~~~
- nginx-deployment.yaml
~~~bash
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
        image: nginx:1.7.9
        ports:
        - containerPort: 80
~~~
~~~bash
$ kubectl apply -f ./nginx-deployment.yaml
~~~
- Scale: replica 늘리기
~~~bash
# Workloads > Deployment details > Actions > Scale
kubectl scale --replicas=3 deployment nginx-deployment
~~~
> 실습 화면을 좁게 사용하는 경우는 메뉴가 숨겨져있어서 바로 찾기 쉽지 않음
- deployment rollout and rollback
~~~
$ kubectl rollout status deployment.v1.apps/nginx-deployment
$ kubectl rollout undo deployments nginx-deployment
$ kubectl rollout history deployment nginx-deployment
~~~
- service : `ClusterIP`, `NodePort`, `LoadBalencer`
- service-nginx.yaml 
~~~yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 60000
    targetPort: 80
~~~
~~~
$ kubectl apply -f ./service-nginx.yaml
$ kubectl get service nginx
~~~
- canary deployment
~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-canary
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
        track: canary
        Version: 1.9.1
    spec:
      containers:
      - name: nginx
        image: nginx:1.9.1
        ports:
        - containerPort: 80
~~~
~~~
$ kubectl apply -f nginx-canary.yaml
$ kubectl get deployments
$ kubectl scale --replicas=0 deployment nginx-deployment
$ kubectl get deployments
~~~

- sessionAffinity
> same request same pod
> 카나리 배포에서 다른 버전으로 교체될 때의 문제를 방지하기 위해서 `sessionAffinity: ClientIP` 옵션을 사용
~~~yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  type: LoadBalancer
  sessionAffinity: ClientIP
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 60000
    targetPort: 80
~~~

## Lab - Deploying Jobs on GKE
> 첫번째 예제는 pi의 소수점 이하 2,000까지 출력하는 예제
> 두번째 예제는 crontab을 거는 예제
- `example-cronjob.yaml` 매 분마다 hello world를 출력하는 crone job
> `schedule` 부분을 보면 1분마다
~~~yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo "Hello, World!"
          restartPolicy: OnFailure
~~~
- 배포 및 확인하는 법
> kubectl get jobs는 잘못된 명령어인듯 보임
~~~bash
$ kubectl get pods
$ kubectl logs [POD-NAME]
Thu Dec 20 15:31:16 WET 2018
Hello,World!
~~~

## Lab - Configuring Pod autoscaling
~~~
$ kubectl autoscale deployment web --max 4 --min 1 --cpu-percent 1
~~~
> 실습이 다소 길지만 핵심은 `kubectl autoscale deployment` deployment는 미리 만들어진 hello-app을 `gcr.io/google-samples/hello-app:1.0`을 사용

## Lab - Deploying Kubernetes Engine via Helm Charts
- Helm Chart를 다운로드 받아서 설치 
~~~bash
$ wget https://storage.googleapis.com/kubernetes-helm/helm-v2.6.2-linux-amd64.tar.gz
$ tar zxfv helm-v2.6.2-linux-amd64.tar.gz -C ~/
# Kubernetes service account: server side of Helm
$ kubectl create serviceaccount tiller --namespace kube-system
$ kubectl create clusterrolebinding tiller-admin-binding \
   --clusterrole=cluster-admin \
   --serviceaccount=kube-system:tiller
$ ~/helm init --service-account=tiller
$ ~/helm repo update
~/helm install stable/redis
~~~
> Error: no available release name found 오류로 중단

## Lab - Deploying to Kubernetes Engine

## Lab - Configuring Google Kubernetes Engine
- Click the Availability, networking, security, and additional features => Enable VPC-native (using alias IP)
- Master IP Range, enter 172.16.0.0/28
> This step assumes that you don't have this range assigned to a subnet. Check VPC Networks in the GCP Console, and select a different range if necessary. Behind the scenes, GCP uses VPC peering to connect the VPC of the cluster master with your default VPC network.
- Inspect the cluster
~~~bash
$ gcloud container clusters describe private-cluster --region us-central1-a
~~~
- Enable master authorized networks (생략)
- Set egress policy (생략)
> 클러스터 이름과 pod수 외에도 상세 설정하는 부분에 대한 Lab

## Lab - Create Services and Ingreess
- dns demo: one service two pods
~~~bash
git clone https://github.com/GoogleCloudPlatformTraining/training-data-analyst
cd ~/training-data-analyst/courses/ak8s/10_Services/
kubectl apply -f dns-demo.yaml
kubectl get pods
kubectl describe pods dns-demo-2
echo $(kubectl get pod dns-demo-2 --template={{.status.podIP}})
ping dns-demo-2.dns-demo.default.svc.cluster.local # unknown
~~~
- ping test
~~~
apt-get update
apt-get install -y iputils-ping
kubectl exec -it dns-demo-1 /bin/bash
~~~
> pod가 ip를 가지지만 외부에서 ping이 되지 않고 1번 pod로 들어간 다음 ping을 설치하고 2번 pod로 ping 이 가는지 확인
그런데 ip를 직접 사용하지 않고 FQDNs를 사용하여 접근. 

- service
~~~yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-lb-svc
spec:
  type: LoadBalancer
  selector:
    name: hello-v2
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
~~~
- ingress
~~~yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: hello-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      Paths:
     - path: /v1
        backend:
          serviceName: hello-svc
          servicePort: 80
      - path: /v2
        backend:
          serviceName: hello-lb-svc
          servicePort: 80
~~~
~~~bash
$ kubectl apply -f hello-ingress.yaml
~~~

## Lab - Configuring Pod Autoscaling
> 로드를 부여하는 스크립트 load gen을 써서 오토 스케일링을 테스트
- HorizontalPodAutoscaler
~~~
$ kubectl get hpa
$ kubectl describe horizontalpodautoscaler web
~~~
~~~
apiVersion: apps/v1
kind: Deployment
metadata:
  name: loadgen
spec:
  replicas: 4
  selector:
    matchLabels:
      app: loadgen
  template:
    metadata:
      labels:
        app: loadgen
    spec:
      containers:
      - name: loadgen
        image: k8s.gcr.io/busybox
        args:
        - /bin/sh
        - -c
        - while true; do wget -q -O- http://web:8080; done
~~~

## Lab - Configuring Persistent Storage

## Lab - Working with GKE Secrets 


## Notes
kubectl get nodes -o=yaml
kubectl get pods
kubectl describe pod
> get pods하고 describe pod의 차이
kubectl get nodes
kubectl get logs
kubectl get pods -o=wird
kubectl exec -it -- /bin/bash
backofflimit
