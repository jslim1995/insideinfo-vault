## 1. Helm Chart Pull : VSO

**(1) Helm3 설치**

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

**(2) VSO Helm chart 파일 다운로드**

```bash
# 1.hashicorp helm repo 추가
helm repo add hashicorp https://helm.releases.hashicorp.com
"hashicorp" has been added to your repositories
# 1-1. helm repo 확인
helm repo list
NAME            URL
hashicorp       https://helm.releases.hashicorp.com

# 2. vault-secrets-operator Chart 확인
helm search repo hashicorp/vault
NAME                                    CHART VERSION   APP VERSION     DESCRIPTION
hashicorp/vault                         0.27.0          1.15.2          Official HashiCorp Vault Chart
hashicorp/vault-secrets-operator        0.4.3           0.4.3           Official Vault Secrets Operator Chart

# 3. vault-secrets-operator Chart 압축파일 생성
helm pull hashicorp/vault-secrets-operator

# 3-1. 파일 확인 : vault-secrets-operator-0.4.3.tgz
ll
total 20
-rw-r--r-- 1 ec2-user ec2-user 18816 Jan 19 01:09 vault-secrets-operator-0.4.3.tgz

# 3-2. (Option) 파일명 변경
mv  vault-secrets-operator-0.4.3.tgz vault-secrets-operator-0.4.3-chart.tgz
ll
total 55988
-rw-r--r-- 1 ec2-user ec2-user    18816 Jan 19 01:09 vault-secrets-operator-0.4.3-chart.tgz

# 4. hashicorp helm repo 제거
helm repo rm hashicorp
"hashicorp" has been removed from your repositories
# 4-1. 확인
helm repo list
Error: no repositories to show
```



## 2. Docker Image save

**VSO Image**

- [gcr.io/kubebuilder/kube-rbac-proxy:v0.15.0](https://console.cloud.google.com/gcr/images/kubebuilder/global/kube-rbac-proxy@sha256:d8cc6ffb98190e8dd403bfe67ddcb454e6127d32b87acc237b3e5240f70a20fb/details?tab=vulnz)
- hashicorp/vault-secrets-operator:0.4.3

**(1) Docker 설치**

```bash
# Amazon Linux 2023
sudo yum install -y docker
sudo systemctl start docker
```

**(2) Docker Image tar 생성**

```bash
# 1. VSO 사용에 필요한 image
# 1-1. Image : kube-rbac-proxy
sudo docker pull gcr.io/kubebuilder/kube-rbac-proxy:v0.15.0
v0.15.0: Pulling from kubebuilder/kube-rbac-proxy
07a64a71e011: Pull complete
fe5ca62666f0: Pull complete
b02a7525f878: Pull complete
fcb6f6d2c998: Pull complete
e8c73c638ae9: Pull complete
1e3d9b7d1452: Pull complete
4aa0ea1413d3: Pull complete
7c881f9ab25e: Pull complete
5627a970d25e: Pull complete
c9c9ec7a3926: Pull complete
Digest: sha256:d8cc6ffb98190e8dd403bfe67ddcb454e6127d32b87acc237b3e5240f70a20fb
Status: Downloaded newer image for gcr.io/kubebuilder/kube-rbac-proxy:v0.15.0
gcr.io/kubebuilder/kube-rbac-proxy:v0.15.0

# 1-2. Image : vault-secrets-operator
sudo docker pull hashicorp/vault-secrets-operator:0.4.3
0.4.3: Pulling from hashicorp/vault-secrets-operator
63b450eae87c: Already exists
960043b8858c: Already exists
fe09e3ee53d2: Pull complete
eebb06941f3e: Pull complete
02cd68c0cbf6: Pull complete
d3c894b5b2b0: Pull complete
b40161cd83fc: Pull complete
46ba3f23f1d3: Pull complete
4fa131a1b726: Pull complete
6f4ee5eb9f26: Pull complete
77a14740c84f: Pull complete
Digest: sha256:167897f4fd6e7d1d74c0b1e01e24f2e5a210ee84a97910540dfc9ab9e2d96c44
Status: Downloaded newer image for hashicorp/vault-secrets-operator:0.4.3
docker.io/hashicorp/vault-secrets-operator:0.4.3

# 2. local docker image 확인
sudo docker image list
\REPOSITORY                           TAG       IMAGE ID       CREATED        SIZE
hashicorp/vault-secrets-operator     0.4.3     c03ae4ee4193   8 days ago     72.6MB
gcr.io/kubebuilder/kube-rbac-proxy   v0.15.0   7ebda747308b   3 months ago   55.9MB

# 3.Image 압축 파일 생성
## Image : kube-rbac-proxy
sudo docker save gcr.io/kubebuilder/kube-rbac-proxy:v0.15.0 -o gcr.io_kubebuilder_kube-rbac-proxy_v0.15.0.tar
## Image : vault-secrets-operator
sudo docker save hashicorp/vault-secrets-operator:0.4.3 -o vault-secrets-operator-0.4.3.tar

# 3-1. Image 압축 파일 생성 확인
## gcr.io_kubebuilder_kube-rbac-proxy_v0.15.0.tar
## vault-secrets-operator-0.4.3.tar
ll
total 128288
-rw------- 1 root     root     57309696 Jan 19 01:14 gcr.io_kubebuilder_kube-rbac-proxy_v0.15.0.tar
-rw-r--r-- 1 ec2-user ec2-user    18816 Jan 19 01:09 vault-secrets-operator-0.4.3-chart.tgz
-rw------- 1 root     root     74033152 Jan 19 01:17 vault-secrets-operator-0.4.3.tar

# 4. Local Docker Images 제거
## Image : kube-rbac-proxy
sudo docker image rm gcr.io/kubebuilder/kube-rbac-proxy:v0.15.0
Untagged: gcr.io/kubebuilder/kube-rbac-proxy:v0.15.0
Untagged: gcr.io/kubebuilder/kube-rbac-proxy@sha256:d8cc6ffb98190e8dd403bfe67ddcb454e6127d32b87acc237b3e5240f70a20fb
Deleted: sha256:7ebda747308b686ef6471bd8377085bce9f210baef15ce2a9bb6c5d8ccd773dc
Deleted: sha256:09f5f7e06380e33f3208ae8a22777d6c4515b2670add5008193b6591e445aa65
Deleted: sha256:7e357c3e09ddfedb3a9109ddaecf7259c8b883d880420425b4eb8fb5d28cfeb9
Deleted: sha256:91465507bb422ae745f918b6e3145996aecf14be199e77587f8d050cfa77d100
Deleted: sha256:4f66bab1707c814dd9d6144e768fc1338e81eed5535291292b7775aabd29e3cb
Deleted: sha256:a2cf28222bc9ca66aa6576ab3060bb780b2247cff92cbbf7307fe6936dde6571
Deleted: sha256:c35abe48110d199036fea39e924fcd643390e5e90ff86f43058bd6fba1f95f24
Deleted: sha256:94c56a889dbce95654613ca8a431ebc71eb1d992ce6b66c2cfc809ffae77cdaf
Deleted: sha256:0c10e549f5634640bb2d35e0c739682204309c2db97a08e9d6c1e79c869d183a
Deleted: sha256:78bb85c0bb2f77cdeb29e520fa8a264e1899784b9271e3771ef957e43793a4db
Deleted: sha256:54ad2ec71039b74f7e82f020a92a8c2ca45f16a51930d539b56973a18b8ffe8d
## Image : vault-secrets-operator
sudo docker image rm hashicorp/vault-secrets-operator:0.4.3
Untagged: hashicorp/vault-secrets-operator:0.4.3
Untagged: hashicorp/vault-secrets-operator@sha256:167897f4fd6e7d1d74c0b1e01e24f2e5a210ee84a97910540dfc9ab9e2d96c44
Deleted: sha256:c03ae4ee419347786a62d48dee58d98da9656c095a7bb58219a804f2a9814f02
Deleted: sha256:81bbb3a1ceb8170b2c9966b544ea842fc7712b8604c120e193602e1c366288b4
Deleted: sha256:b6bb536cbbe3c019f7d4a8ea296e4c61b3f15eb6a95a3c164f007e1f811675e4
Deleted: sha256:d55f1df280573f697e644268ff40bf04ce9619b73e20baec8b29c9ee75949a2b
Deleted: sha256:667c2585269487527d59a778d94b7b2bd0afd4ce19b58e176596650bd8f50cfb
Deleted: sha256:af0ee46bc53fefb5ea57225661a0a9105c912ece121a505f364f02e210e9b390
Deleted: sha256:4cf9819ec20d07aef170d67f2bee5d3cc40d562595e60bc5316ed21c7549cfa2
Deleted: sha256:e94affbf149a32bb39ac597a98f70b1de9294b17fe1023211af272be30777974
Deleted: sha256:fa003b7a950e78a04d8679fec9ecb17e907068f87a41dc2bd668e438784ed34f
Deleted: sha256:cc9aa4a54b9d1c4db6df5579be431c581f51a3554f556e0953ec0fb9a1e23fba
# 4-2. 제거 확인
sudo docker images
REPOSITORY   TAG       IMAGE ID   CREATED   SIZE
```



## 3. Harbor Install

**(1) Ingress-Nginx Controller : Harbor에서 Ingress 타입 사용 시 구성**

- 각 리소스 확인 필요

- TLS 구성 확인 필요

- Security Group : eks-cluster-sg에 nlb 관련 inbound 규칙 추가됨

  ```bash
  # 1. Ingress-Nginx Controller 배포
  kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/aws/deploy.yaml

  namespace/ingress-nginx created
  serviceaccount/ingress-nginx created
  serviceaccount/ingress-nginx-admission created
  role.rbac.authorization.k8s.io/ingress-nginx created
  role.rbac.authorization.k8s.io/ingress-nginx-admission created
  clusterrole.rbac.authorization.k8s.io/ingress-nginx created
  clusterrole.rbac.authorization.k8s.io/ingress-nginx-admission created
  rolebinding.rbac.authorization.k8s.io/ingress-nginx created
  rolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
  clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx created
  clusterrolebinding.rbac.authorization.k8s.io/ingress-ngi0nx-admission created
  configmap/ingress-nginx-controller created
  service/ingress-nginx-controller created
  service/ingress-nginx-controller-admission created
  deployment.apps/ingress-nginx-controller created
  job.batch/ingress-nginx-admission-create created
  job.batch/ingress-nginx-admission-patch created
  ingressclass.networking.k8s.io/nginx created
  validatingwebhookconfiguration.admissionregistration.k8s.io/ingress-nginx-admission created

  # 1-1. 배포된 리소스 확인
  # service/ingress-nginx-controller: AWS NLB 생성.
  kubectl get all -n ingress-nginx
  NAME                                            READY   STATUS      RESTARTS   AGE
  pod/ingress-nginx-admission-create-lnjlp        0/1     Completed   0          104s
  pod/ingress-nginx-admission-patch-k99kl         0/1     Completed   1          104s
  pod/ingress-nginx-controller-8558859656-4p7x5   1/1     Running     0          104s

  NAME                                         TYPE           CLUSTER-IP      EXTERNAL-IP                                                                          PORT(S)                      AGE
  service/ingress-nginx-controller             LoadBalancer   10.100.33.179   a68a213ba97c94bb3b8c2af073f31d56-15f5b3121f8adcbb.elb.ap-southeast-1.amazonaws.com   80:30096/TCP,443:32659/TCP   104s
  service/ingress-nginx-controller-admission   ClusterIP      10.100.5.87     <none>                                                                               443/TCP                      104s

  NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
  deployment.apps/ingress-nginx-controller   1/1     1            1           104s

  NAME                                                  DESIRED   CURRENT   READY   AGE
  replicaset.apps/ingress-nginx-controller-8558859656   1         1         1       104s

  NAME                                       COMPLETIONS   DURATION   AGE
  job.batch/ingress-nginx-admission-create   1/1           5s         104s
  job.batch/ingress-nginx-admission-patch    1/1           7s         104s

  # 1-2. ingress-nginx-controller 서비스 확인
  kubectl get service -n ingress-nginx ingress-nginx-controller
  NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP                                                                          PORT(S)                      AGE
  ingress-nginx-controller   LoadBalancer   10.100.33.179   a68a213ba97c94bb3b8c2af073f31d56-15f5b3121f8adcbb.elb.ap-southeast-1.amazonaws.com   80:30096/TCP,443:32659/TCP   119s

  # 1-3. ingress-nginx-controller 서비스 값 확인
  kubectl get service -n ingress-nginx ingress-nginx-controller
  NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP                                                                          PORT(S)                      AGE
  ingress-nginx-controller   LoadBalancer   10.100.33.179   a68a213ba97c94bb3b8c2af073f31d56-15f5b3121f8adcbb.elb.ap-southeast-1.amazonaws.com   80:30096/TCP,443:32659/TCP   119s
  [ec2-user@ip-172-31-25-84 shinhan-harbor]$ kubectl describe service -n ingress-nginx ingress-nginx-controller
  Name:                     ingress-nginx-controller
  Namespace:                ingress-nginx
  Labels:                   app.kubernetes.io/component=controller
                            app.kubernetes.io/instance=ingress-nginx
                            app.kubernetes.io/name=ingress-nginx
                            app.kubernetes.io/part-of=ingress-nginx
                            app.kubernetes.io/version=1.8.2
  Annotations:              service.beta.kubernetes.io/aws-load-balancer-backend-protocol: tcp
                            service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: true
                            service.beta.kubernetes.io/aws-load-balancer-type: nlb
  Selector:                 app.kubernetes.io/component=controller,app.kubernetes.io/instance=ingress-nginx,app.kubernetes.io/name=ingress-nginx
  Type:                     LoadBalancer
  IP Family Policy:         SingleStack
  IP Families:              IPv4
  IP:                       10.100.33.179
  IPs:                      10.100.33.179
  LoadBalancer Ingress:     a68a213ba97c94bb3b8c2af073f31d56-15f5b3121f8adcbb.elb.ap-southeast-1.amazonaws.com
  Port:                     http  80/TCP
  TargetPort:               http/TCP
  NodePort:                 http  30096/TCP
  Endpoints:                172.31.38.75:80
  Port:                     https  443/TCP
  TargetPort:               https/TCP
  NodePort:                 https  32659/TCP
  Endpoints:                172.31.38.75:443
  Session Affinity:         None
  External Traffic Policy:  Local
  HealthCheck NodePort:     32546
  Events:
    Type    Reason                Age   From                Message
    ----    ------                ----  ----                -------
    Normal  EnsuringLoadBalancer  2m9s  service-controller  Ensuring load balancer
    Normal  EnsuredLoadBalancer   2m6s  service-controller  Ensured load balancer
  ```

**(2) Route53 - domain 등록  : Harbor에서 Ingress 타입 사용 시 구성**

- 트래픽 라우팅 대상 : Network Load Balancer (*Ingress-Nginx Controller)

  ​

**(3) Harbor Helm Chart : values.yaml 편집**

- **Helm ‘harbor/harbor’ chart 가져오기**

  ```bash
  # 1. helm repo 추가
  helm repo add harbor https://helm.goharbor.io
  "harbor" has been added to your repositories
  # 1-1. 확인
  helm repo list
  NAME    URL
  harbor  https://helm.goharbor.io

  # 2. harbor helm chart 가져오기
  helm pull harbor/harbor --untar 
  # 2-1. directory 확인
  ll
  total 0
  drwxr-xr-x 3 ec2-user ec2-user      111 Jan 19 01:49 harbor
  # 2-2. file 확인
  cd harbor/
  ll
  total 236
  -rw-r--r--  1 ec2-user ec2-user    568 Jan 19 01:49 Chart.yaml
  -rw-r--r--  1 ec2-user ec2-user  11357 Jan 19 01:49 LICENSE
  -rw-r--r--  1 ec2-user ec2-user 185720 Jan 19 01:49 README.md
  drwxr-xr-x 14 ec2-user ec2-user    220 Jan 19 01:49 templates
  -rw-r--r--  1 ec2-user ec2-user  36540 Jan 19 01:49 values.yaml
  ```


- **Harbor Helm 차트 템플릿 변수 파일 (values.yaml) 을 수정**

  ```bash
  vi values.yaml
  ```

  - **Service**

    ```yaml
    expose:
      type: ingress # 1.ingress type 사용
      tls:
        enabled: true
        certSource: auto
        auto:
          commonName: ""
        secret:
          secretName: ""
      ingress:
        hosts:
          core: harbor.inside-vault.com	 # 2.domain 지정
        controller: default
        kubeVersionOverride: ""
        className: "nginx"  # 3.ingress-nginx controller 배포 시, className 지정
        annotations:
          ingress.kubernetes.io/ssl-redirect: "true"
          ingress.kubernetes.io/proxy-body-size: "0"
          nginx.ingress.kubernetes.io/ssl-redirect: "true"
          nginx.ingress.kubernetes.io/proxy-body-size: "0"
        harbor:
          annotations: {}
          labels: {}
    ... 
    externalURL: https://harbor.inside-vault.com  # 4."expose.ingress.hosts.core"의 값
    ...
    ```

  - **Storage**

    ```yaml
    # The persistence is enabled by default and a default StorageClass
    # is needed in the k8s cluster to provision volumes dynamically.
    # Specify another StorageClass in the "storageClass" or set "existingClaim"
    # if you already have existing persistent volumes to use
    #
    # For storing images and charts, you can also use "azure", "gcs", "s3",
    # "swift" or "oss". Set it in the "imageChartStorage" section
    persistence:
      enabled: false  # 5.PVC disabled (*s3 사용 시) : true → false 변경
    ...
    imageChartStorage:
        # Specify whether to disable `redirect` for images and chart storage, for
        # backends which not supported it (such as using minio for `s3` storage type), please disable
        # it. To disable redirects, simply set `disableredirect` to `true` instead.
        # Refer to
        # https://github.com/distribution/distribution/blob/main/docs/configuration.md#redirect
        # for the detail.
        disableredirect: true # 6.false → true로 변경 (*s3 사용 시)
        type: s3  			# 7.s3 type 선택
        filesystem:
        ...
        s3:
          region: ap-southeast-1  # 8.s3 region 정보 입력
          bucket: lmy-tf-bucket   # 8.s3 bucket 이름 입력
          accesskey: awsaccesskey # 8.변경 필요
          secretkey: awssecretkey # 8.변경 필요
    ```

  - **harborAdminPassword**

    ```yaml
    # 9.password 확인
    existingSecretAdminPasswordKey: HARBOR_ADMIN_PASSWORD
    harborAdminPassword: "Harbor12345"
    ```

**(4) Harbor Helm 설치**

```bash
# 1. Kubernetes namespace 생성
kubectl create ns harbor
namespace/harbor created

# 2. Harbor 설치
helm install -f values.yaml harbor . -n harbor
NAME: harbor
LAST DEPLOYED: Fri Jan 19 02:05:19 2024
NAMESPACE: harbor
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Please wait for several minutes for Harbor deployment to complete.
Then you should be able to visit the Harbor portal at https://harbor.inside-vault.com
For more details, please visit https://github.com/goharbor/harbor

# 2-1. Harbor 설치 확인
kubectl get all -n harbor
NAME                                     READY   STATUS    RESTARTS      AGE
pod/harbor-core-7b8658cc65-vpcwx         1/1     Running   0             83s
pod/harbor-database-0                    1/1     Running   0             83s
pod/harbor-jobservice-8674896fcd-k949h   1/1     Running   2 (67s ago)   83s
pod/harbor-portal-6b8c795cf-qbp8t        1/1     Running   0             83s
pod/harbor-redis-0                       1/1     Running   0             83s
pod/harbor-registry-79bcdfdc55-bflfg     2/2     Running   0             83s
pod/harbor-trivy-0                       1/1     Running   0             83s

NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
service/harbor-core         ClusterIP   10.100.12.124    <none>        80/TCP              83s
service/harbor-database     ClusterIP   10.100.54.177    <none>        5432/TCP            83s
service/harbor-jobservice   ClusterIP   10.100.2.249     <none>        80/TCP              83s
service/harbor-portal       ClusterIP   10.100.124.21    <none>        80/TCP              83s
service/harbor-redis        ClusterIP   10.100.246.187   <none>        6379/TCP            83s
service/harbor-registry     ClusterIP   10.100.141.224   <none>        5000/TCP,8080/TCP   83s
service/harbor-trivy        ClusterIP   10.100.227.166   <none>        8080/TCP            83s

NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/harbor-core         1/1     1            1           83s
deployment.apps/harbor-jobservice   1/1     1            1           83s
deployment.apps/harbor-portal       1/1     1            1           83s
deployment.apps/harbor-registry     1/1     1            1           83s

NAME                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/harbor-core-7b8658cc65         1         1         1       83s
replicaset.apps/harbor-jobservice-8674896fcd   1         1         1       83s
replicaset.apps/harbor-portal-6b8c795cf        1         1         1       83s
replicaset.apps/harbor-registry-79bcdfdc55     1         1         1       83s

NAME                               READY   AGE
statefulset.apps/harbor-database   1/1     83s
statefulset.apps/harbor-redis      1/1     83s
statefulset.apps/harbor-trivy      1/1     83s

# 2-2. Harbor 확인
kubectl get deployments -n harbor -owide
NAME                READY   UP-TO-DATE   AVAILABLE   AGE    CONTAINERS             IMAGES                                                                 SELECTOR
harbor-core         1/1     1            1           101s   core                   goharbor/harbor-core:v2.10.0                                           app=harbor,component=core,release=harbor
harbor-jobservice   1/1     1            1           101s   jobservice             goharbor/harbor-jobservice:v2.10.0                                     app=harbor,component=jobservice,release=harbor
harbor-portal       1/1     1            1           101s   portal                 goharbor/harbor-portal:v2.10.0                                         app=harbor,component=portal,release=harbor
harbor-registry     1/1     1            1           101s   registry,registryctl   goharbor/registry-photon:v2.10.0,goharbor/harbor-registryctl:v2.10.0   app=harbor,component=registry,release=harbor

# 3. ingress 확인
kubectl get ingress -n harbor
NAME             CLASS   HOSTS                ADDRESS                                                                              PORTS     AGE
harbor-ingress   nginx   core.harbor.domain   a68a213ba97c94bb3b8c2af073f31d56-15f5b3121f8adcbb.elb.ap-southeast-1.amazonaws.com   80, 443   113s

# 3-1. ingress 정보 확인
kubectl describe ingress -n harbor harbor-ingress
Name:             harbor-ingress
Labels:           app=harbor
                  app.kubernetes.io/managed-by=Helm
                  chart=harbor
                  heritage=Helm
                  release=harbor
Namespace:        harbor
Address:          a68a213ba97c94bb3b8c2af073f31d56-15f5b3121f8adcbb.elb.ap-southeast-1.amazonaws.com
Ingress Class:    nginx
Default backend:  <default>
TLS:
  harbor-ingress terminates core.harbor.domain
Rules:
  Host                Path  Backends
  ----                ----  --------
  core.harbor.domain
                      /api/         harbor-core:80 (172.31.32.158:8080)
                      /service/     harbor-core:80 (172.31.32.158:8080)
                      /v2/          harbor-core:80 (172.31.32.158:8080)
                      /chartrepo/   harbor-core:80 (172.31.32.158:8080)
                      /c/           harbor-core:80 (172.31.32.158:8080)
                      /             harbor-portal:80 (172.31.23.179:8080)
Annotations:          ingress.kubernetes.io/proxy-body-size: 0
                      ingress.kubernetes.io/ssl-redirect: true
                      meta.helm.sh/release-name: harbor
                      meta.helm.sh/release-namespace: harbor
                      nginx.ingress.kubernetes.io/proxy-body-size: 0
                      nginx.ingress.kubernetes.io/ssl-redirect: true
Events:
  Type    Reason  Age                   From                      Message
  ----    ------  ----                  ----                      -------
  Normal  Sync    117s (x2 over 2m28s)  nginx-ingress-controller  Scheduled for sync

```

**(5) Harbor 접속**

```bash
# values.yaml 'externalURL' 사용
https://harbor.inside-vault.com

# 기본 계정
# values.yaml 'harborAdminPassword' 확인
admin
Harbor12345
```

![image](https://github.com/jslim1995/insideinfo-vault/assets/124943887/d80ae04e-27ab-4d9a-aa10-6b86aef6257d)




## 4. Harbor : Image Push

**(1) Harbor Projects 확인**

![image](https://github.com/jslim1995/insideinfo-vault/assets/124943887/1c5a9291-2ef6-4a59-a953-5bd630c2e867)


![image](https://github.com/jslim1995/insideinfo-vault/assets/124943887/c9396a03-a56e-47f0-b48e-a52d4d56e8e6)



**(2) Harbor Registry Certificate 저장**

```bash
# 1. Harbor UI : Projects > [Projects name]:library > Registry Certificate 다운 받아 저장

## 자체 서명 인증서 사용하는 Harbor의 경우
# 2-1. 'ca.crt' 확인
ll
-rw-r--r-- 1 ec2-user ec2-user     1127 Jan 19 02:23 ca.crt

# 2-2. directory 생성
sudo mkdir -p /etc/docker/certs.d/harbor.inside-vault.com

# 2-3. 'ca.crt' 이동
# sudo cp /path/to/harbor.inside-vault.com.crt /etc/docker/certs.d/harbor.inside-vault.com/ca.crt
sudo cp ./ca.crt /etc/docker/certs.d/harbor.inside-vault.com/ca.crt
```

**(3) Harbor에 Docker image Push**

```bash
# 1. Docker Image 압축 파일 목록
# Image: gcr.io_kubebuilder_kube-rbac-proxy_v0.15.0.tar, vault-secrets-operator-0.4.3.tar
ll
-rw-r--r-- 1 ec2-user ec2-user     1127 Jan 19 02:23 ca.crt
-rw------- 1 root     root     57309696 Jan 19 01:14 gcr.io_kubebuilder_kube-rbac-proxy_v0.15.0.tar
drwxr-xr-x 3 ec2-user ec2-user      111 Jan 19 02:16 harbor
-rw-r--r-- 1 ec2-user ec2-user    18816 Jan 19 01:09 vault-secrets-operator-0.4.3-chart.tgz
-rw------- 1 root     root     74033152 Jan 19 01:17 vault-secrets-operator-0.4.3.tar

# 2. Harbor login
sudo docker login harbor.inside-vault.com -u admin -p Harbor12345
WARNING! Using --password via the CLI is insecure. Use --password-stdin.
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
<https://docs.docker.com/engine/reference/commandline/login/#credentials-store>

Login Succeeded

##Image : gcr.io/kubebuilder/kube-rbac-proxy:v0.15.0 & hashicorp/vault-secrets-operator:0.4.3
# 3-1. local Images 확인
sudo docker image ls
REPOSITORY   TAG       IMAGE ID   CREATED   SIZE

# 3-2.Image Load
##Image : gcr.io/kubebuilder/kube-rbac-proxy:v0.15.0 
sudo docker load -i gcr.io_kubebuilder_kube-rbac-proxy_v0.15.0.tar
54ad2ec71039: Loading layer [==================================================>]  327.7kB/327.7kB
6fbdf253bbc2: Loading layer [==================================================>]   51.2kB/51.2kB
7bea6b893187: Loading layer [==================================================>]  3.205MB/3.205MB
ff5700ec5418: Loading layer [==================================================>]  10.24kB/10.24kB
d52f02c6501c: Loading layer [==================================================>]  10.24kB/10.24kB
e624a5370eca: Loading layer [==================================================>]  10.24kB/10.24kB
1a73b54f556b: Loading layer [==================================================>]  10.24kB/10.24kB
d2d7ec0f6756: Loading layer [==================================================>]  10.24kB/10.24kB
4cb10dd2545b: Loading layer [==================================================>]  225.3kB/225.3kB
35cbfef5b026: Loading layer [==================================================>]  53.41MB/53.41MB
Loaded image: gcr.io/kubebuilder/kube-rbac-proxy:v0.15.0
##Image : hashicorp/vault-secrets-operator:0.4.3
sudo docker load -i vault-secrets-operator-0.4.3.tar
54ad2ec71039: Loading layer [==================================================>]  327.7kB/327.7kB
6fbdf253bbc2: Loading layer [==================================================>]   51.2kB/51.2kB
accc3e6808c0: Loading layer [==================================================>]  3.205MB/3.205MB
ff5700ec5418: Loading layer [==================================================>]  10.24kB/10.24kB
d52f02c6501c: Loading layer [==================================================>]  10.24kB/10.24kB
e624a5370eca: Loading layer [==================================================>]  10.24kB/10.24kB
1a73b54f556b: Loading layer [==================================================>]  10.24kB/10.24kB
d2d7ec0f6756: Loading layer [==================================================>]  10.24kB/10.24kB
4cb10dd2545b: Loading layer [==================================================>]  225.3kB/225.3kB
6f85ee1a57c8: Loading layer [==================================================>]  70.12MB/70.12MB
8ed83070a24e: Loading layer [==================================================>]  7.168kB/7.168kB
Loaded image: hashicorp/vault-secrets-operator:0.4.3

# 3-3. Local Docker Images 확인
# Image: hashicorp/vault-secrets-operator:0.4.3, gcr.io/kubebuilder/kube-rbac-proxy:v0.15.0
sudo docker images
REPOSITORY                           TAG       IMAGE ID       CREATED        SIZE
hashicorp/vault-secrets-operator     0.4.3     c03ae4ee4193   8 days ago     72.6MB
gcr.io/kubebuilder/kube-rbac-proxy   v0.15.0   7ebda747308b   3 months ago   55.9MB

# 3-4.Image에 태그 추가
##Image : gcr.io/kubebuilder/kube-rbac-proxy:v0.15.0 
sudo docker tag hashicorp/vault-secrets-operator:0.4.3 harbor.inside-vault.com/library/vault-secrets-operator:0.4.3
##Image : hashicorp/vault-secrets-operator:0.4.3
sudo docker tag gcr.io/kubebuilder/kube-rbac-proxy:v0.15.0 harbor.inside-vault.com/library/gcr.io/kubebuilder/kube-rbac-proxy:v0.15.0

# 3-5. local Images 확인
# harbor.inside-vault.com/library/vault-secrets-operator:0.4.3
# harbor.inside-vault.com/library/gcr.io/kubebuilder/kube-rbac-proxy:v0.15.0 
sudo docker images
REPOSITORY                                                           TAG       IMAGE ID       CREATED        SIZE
hashicorp/vault-secrets-operator                                     0.4.3     c03ae4ee4193   8 days ago     72.6MB
harbor.inside-vault.com/library/vault-secrets-operator               0.4.3     c03ae4ee4193   8 days ago     72.6MB
gcr.io/kubebuilder/kube-rbac-proxy                                   v0.15.0   7ebda747308b   3 months ago   55.9MB
harbor.inside-vault.com/library/gcr.io/kubebuilder/kube-rbac-proxy   v0.15.0   7ebda747308b   3 months ago   55.9MB

# 3-5.Docker Push
##Image : gcr.io/kubebuilder/kube-rbac-proxy:v0.15.0 
sudo docker push harbor.inside-vault.com/library/gcr.io/kubebuilder/kube-rbac-proxy:v0.15.0
The push refers to repository [harbor.inside-vault.com/library/gcr.io/kubebuilder/kube-rbac-proxy]
35cbfef5b026: Pushed
4cb10dd2545b: Mounted from library/vault-secrets-operator
d2d7ec0f6756: Mounted from library/vault-secrets-operator
1a73b54f556b: Mounted from library/vault-secrets-operator
e624a5370eca: Mounted from library/vault-secrets-operator
d52f02c6501c: Mounted from library/vault-secrets-operator
ff5700ec5418: Mounted from library/vault-secrets-operator
7bea6b893187: Pushed
6fbdf253bbc2: Mounted from library/vault-secrets-operator
54ad2ec71039: Mounted from library/vault-secrets-operator
v0.15.0: digest: sha256:a02db6144ae2a082e799f0c3a4ecd82f28a91dbd9885040221ff904e957ab95a size: 2402
##Image : hashicorp/vault-secrets-operator:0.4.3
sudo docker push harbor.inside-vault.com/library/vault-secrets-operator:0.4.3
The push refers to repository [harbor.inside-vault.com/library/vault-secrets-operator]
8ed83070a24e: Pushed
6f85ee1a57c8: Pushed
4cb10dd2545b: Pushed
d2d7ec0f6756: Pushed
1a73b54f556b: Pushed
e624a5370eca: Pushed
d52f02c6501c: Pushed
ff5700ec5418: Pushed
accc3e6808c0: Pushed
6fbdf253bbc2: Pushed
54ad2ec71039: Pushed
0.4.3: digest: sha256:66c1e1d5c02d36a1adcb06f37707469cd89ae9f1ae3a377eea3e95845eca50f5 size: 2610
```

**(4) Harbor 확인 : Projects > library**

![image](https://github.com/jslim1995/insideinfo-vault/assets/124943887/98537e42-ae0d-476b-97a3-f703330952e1)

![image](https://github.com/jslim1995/insideinfo-vault/assets/124943887/00b117b6-c360-46d5-b62f-6bfd90878432)

![image](https://github.com/jslim1995/insideinfo-vault/assets/124943887/779b758e-72a7-47b5-9753-dbdbbec52607)




**(5) AWS : S3 Bucket 객체 ‘Bucket/docker/registry/…’ 생성 확인**

![image](https://github.com/jslim1995/insideinfo-vault/assets/124943887/ba968219-bd43-41b4-a82f-bdf424d79609)





## 5. VSO : Helm Chart Package 생성

: Image 수정 후, 패키지 생성

```bash
# 1.파일 확인 : vault-secrets-operator-0.4.3-chart.tgz
$ ll
total 121356
-rw-r--r-- 1 ec2-user ec2-user 57309696 Jan 18 05:24 gcr.io_kubebuilder_kube-rbac-proxy_v0.15.0.tar
-rw-r--r-- 1 ec2-user ec2-user    18816 Jan 18 01:35 vault-secrets-operator-0.4.3-chart.tgz
-rw-r--r-- 1 ec2-user ec2-user 74033152 Jan 18 01:35 vault-secrets-operator-0.4.3.tar

# 2.압축 해제
$ tar -xzvf vault-secrets-operator-0.4.3-chart.tgz
vault-secrets-operator/Chart.yaml
vault-secrets-operator/values.yaml
vault-secrets-operator/templates/_helpers.tpl
vault-secrets-operator/templates/default-transit-auth-method.yaml
vault-secrets-operator/templates/default-vault-auth-method.yaml
vault-secrets-operator/templates/default-vault-connection.yaml
vault-secrets-operator/templates/deployment.yaml
vault-secrets-operator/templates/hcpauth_editor_role.yaml
vault-secrets-operator/templates/hcpauth_viewer_role.yaml
vault-secrets-operator/templates/hcpvaultsecretsapp_editor_role.yaml
vault-secrets-operator/templates/hcpvaultsecretsapp_viewer_role.yaml
vault-secrets-operator/templates/leader-election-rbac.yaml
vault-secrets-operator/templates/manager-config.yaml
vault-secrets-operator/templates/manager-rbac.yaml
vault-secrets-operator/templates/metrics-reader-rbac.yaml
vault-secrets-operator/templates/metrics-service.yaml
vault-secrets-operator/templates/prometheus-servicemonitor.yaml
vault-secrets-operator/templates/proxy-rbac.yaml
vault-secrets-operator/templates/tests/test-runner.yaml
vault-secrets-operator/templates/vaultauth_editor_role.yaml
vault-secrets-operator/templates/vaultauth_viewer_role.yaml
vault-secrets-operator/templates/vaultconnection_editor_role.yaml
vault-secrets-operator/templates/vaultconnection_viewer_role.yaml
vault-secrets-operator/templates/vaultdynamicsecret_editor_role.yaml
vault-secrets-operator/templates/vaultdynamicsecret_viewer_role.yaml
vault-secrets-operator/templates/vaultpkisecret_editor_role.yaml
vault-secrets-operator/templates/vaultpkisecret_viewer_role.yaml
vault-secrets-operator/templates/vaultstaticsecret_editor_role.yaml
vault-secrets-operator/templates/vaultstaticsecret_viewer_role.yaml
vault-secrets-operator/.helmignore
vault-secrets-operator/crds/secrets.hashicorp.com_hcpauths.yaml
vault-secrets-operator/crds/secrets.hashicorp.com_hcpvaultsecretsapps.yaml
vault-secrets-operator/crds/secrets.hashicorp.com_vaultauths.yaml
vault-secrets-operator/crds/secrets.hashicorp.com_vaultconnections.yaml
vault-secrets-operator/crds/secrets.hashicorp.com_vaultdynamicsecrets.yaml
vault-secrets-operator/crds/secrets.hashicorp.com_vaultpkisecrets.yaml
vault-secrets-operator/crds/secrets.hashicorp.com_vaultstaticsecrets.yaml

# 2-1.확인
ll
...
drwxrwxr-x 4 ec2-user ec2-user      131 Jan 18 06:06 vault-secrets-operator
# 2-2.이동
cd vault-secrets-operator/
# 2-3.Chart 파일 확인
ll
total 56
-rw-r--r-- 1 ec2-user ec2-user   237 Jan 18 05:00 Chart.yaml
drwxrwxr-x 2 ec2-user ec2-user  4096 Jan 18 05:00 crds
drwxrwxr-x 3 ec2-user ec2-user  4096 Jan 18 05:00 templates
-rw-r--r-- 1 ec2-user ec2-user 23598 Jan 18 05:00 values.yaml

# 3.values.yaml 수정
# 3-1.image repository 변경
vi values.yaml
# (1)kubeRbacProxy
  kubeRbacProxy:
    image:
      repository: harbor.inside-vault.com/library/gcr.io/kubebuilder/kube-rbac-proxy # Harbor path로 수정
      tag: v0.15.0
# (2)vault-secrets-operator
 manager:
    image:
      repository: harbor.inside-vault.com/library/vault-secrets-operator  # Harbor path로 수정
      tag: 0.4.3

# 4.Chart.yaml : 주석 처리
$ vi Chart.yaml
#sources:
#- https://github.com/hashicorp/vault-secrets-operator
 
# 4.Package 새로 생성
$ helm package .
Successfully packaged chart and saved it to: /home/ec2-user/docker_images/vso/vault-secrets-operator/vault-secrets-operator-0.4.3.tgz

# 4-1.Package 파일 생성 확인
ll
total 56
-rw-r--r-- 1 ec2-user ec2-user   235 Jan 11 02:04 Chart.yaml
drwxrwxr-x 2 ec2-user ec2-user  4096 Jan 19 03:38 crds
drwxrwxr-x 3 ec2-user ec2-user  4096 Jan 19 03:38 templates
-rw-r--r-- 1 ec2-user ec2-user 23598 Jan 19 03:52 values.yaml
-rw-rw-r-- 1 ec2-user ec2-user 18851 Jan 19 03:52 vault-secrets-operator-0.4.3.tgz
```



## 6. VSO 설치

**(1) EKS 각 Node에 Harbor Registry Certificate 저장**

```bash
# EKS : containerd 기반 kubernetes
# /etc/containerd/config.toml 확인
# config_path: '/etc/docker/certs.d' 지정되어 있어 사용 가능
[plugins."io.containerd.grpc.v1.cri".registry]
config_path = "/etc/containerd/certs.d:/etc/docker/certs.d"

# 1.'ca.crt' 확인
ll
-rw-r--r-- 1 ec2-user ec2-user 1127 Jan 19 04:11 ca.crt

# 2.directory 생성
sudo mkdir -p /etc/docker/certs.d/harbor.inside-vault.com

# 3.'ca.crt' 이동
sudo cp ./ca.crt /etc/docker/certs.d/harbor.inside-vault.com/ca.crt
```



## 6-1. Chart TAR 파일 사용

- Helm install 수행

```bash
# 1.Package 파일 확인
ll
total 56
-rw-r--r-- 1 ec2-user ec2-user   235 Jan 11 02:04 Chart.yaml
drwxrwxr-x 2 ec2-user ec2-user  4096 Jan 19 03:38 crds
drwxrwxr-x 3 ec2-user ec2-user  4096 Jan 19 03:38 templates
-rw-r--r-- 1 ec2-user ec2-user 23598 Jan 19 03:52 values.yaml
-rw-rw-r-- 1 ec2-user ec2-user 18851 Jan 19 03:52 vault-secrets-operator-0.4.3.tgz

# 2.Helm install 수행
helm install --create-namespace --namespace vsoperator vault-secrets-operator vault-secrets-operator-0.4.3.tgz
NAME: vault-secrets-operator
LAST DEPLOYED: Fri Jan 19 03:54:00 2024
NAMESPACE: vsoperator
STATUS: deployed
REVISION: 1

# 3-1. 배포 확인
kubectl get all -n vsoperator
NAME                                                             READY   STATUS    RESTARTS   AGE
pod/vault-secrets-operator-controller-manager-77dc869559-9dhgw   2/2     Running   0          14s

NAME                                             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/vault-secrets-operator-metrics-service   ClusterIP   10.100.26.122   <none>        8443/TCP   14s

NAME                                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/vault-secrets-operator-controller-manager   1/1     1            1           14s

NAME                                                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/vault-secrets-operator-controller-manager-77dc869559   1         1         1       14s


# 4. vault-secrets-operator-controller-manager pod : Events-pulled image 확인
# Successfully pulled image "harbor.inside-vault.com/library/gcr.io/kubebuilder/kube-rbac-proxy:v0.15.0"
# Successfully pulled image "harbor.inside-vault.com/library/vault-secrets-operator:0.4.3"
kubectl describe -n vsoperator pod vault-secrets-operator-controller-manager-77dc869559-9dhgw
```



## 6-2. Harbor - OCI 표준 규격 Helm Chart 저장소 사용

**[참고] Harbor App Version : 2.10.0 설치 **

[Docs v 2.10.0 : Working with OCI Helm Charts](https://goharbor.io/docs/2.10.0/working-with-projects/working-with-oci/working-with-helm-oci-charts/)

Harbor App Version : v2.10.0

Helm Version : v3.11.1

**(1) Harbor의 OCI-Compatible Registry에 로그인**

```bash
# If the project that the helm chart belongs to is private, you must sign in first
# helm registry login xx.xx.xx.xx
helm registry login harbor.inside-vault.com
Username: admin
Password:
Login Succeeded
```

**(2) Pushing OCI Helm charts**

- Harbor : Projects 확인 (vso_chart 생성)

![image](https://github.com/jslim1995/insideinfo-vault/assets/124943887/eade9d40-fd4d-4b2a-a79c-c4f5305b26df)
![image](https://github.com/jslim1995/insideinfo-vault/assets/124943887/1396b48f-aaea-4eee-89e7-72876c6645a1)



- **인증서 등록**

```bash
# Harbor UI : Projects > [Projects name] > Registry Certificate 다운 받아 저장
# 1. ca.crt 파일 확인
ll
total 60
-rw-r--r-- 1 ec2-user ec2-user  1127 Jan 19 02:23 ca.crt
-rw-r--r-- 1 ec2-user ec2-user   235 Jan 11 02:04 Chart.yaml
drwxrwxr-x 2 ec2-user ec2-user  4096 Jan 19 03:38 crds
drwxrwxr-x 3 ec2-user ec2-user  4096 Jan 19 03:38 templates
-rw-r--r-- 1 ec2-user ec2-user 23598 Jan 19 03:52 values.yaml
-rw-rw-r-- 1 ec2-user ec2-user 18851 Jan 19 03:52 vault-secrets-operator-0.4.3.tgz
# 2. 인증서 이동
sudo cp ca.crt /etc/ssl/certs/
# 2-1. 확인: ca.crt 파일
ll /etc/ssl/certs/
total 16
lrwxrwxrwx 1 root root   49 Feb 23  2023 ca-bundle.crt -> /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem
lrwxrwxrwx 1 root root   55 Feb 23  2023 ca-bundle.trust.crt -> /etc/pki/ca-trust/extracted/openssl/ca-bundle.trust.crt
-rw-r--r-- 1 root root 1127 Jan 19 03:12 ca.crt
-rwxr-xr-x 1 root root  610 Feb  3  2023 make-dummy-cert
-rw-r--r-- 1 root root 2516 Feb  3  2023 Makefile
-rwxr-xr-x 1 root root  829 Feb  3  2023 renew-dummy-cert

```

- **Harbor에 Helm charts Push 수행**

```bash
# 파일 확인 : vault-secrets-operator-0.4.3.tgz
ll
total 60
-rw-r--r-- 1 ec2-user ec2-user  1127 Jan 19 02:23 ca.crt
-rw-r--r-- 1 ec2-user ec2-user   235 Jan 11 02:04 Chart.yaml
drwxrwxr-x 2 ec2-user ec2-user  4096 Jan 19 03:38 crds
drwxrwxr-x 3 ec2-user ec2-user  4096 Jan 19 03:38 templates
-rw-r--r-- 1 ec2-user ec2-user 23598 Jan 19 03:52 values.yaml
-rw-rw-r-- 1 ec2-user ec2-user 18851 Jan 19 03:52 vault-secrets-operator-0.4.3.tgz

# 2. Harbor에 Helm charts push 수행
# helm push <chart_name_and_version>.tgz oci://<harbor_address>/<project>
helm push vault-secrets-operator-0.4.3.tgz oci://harbor.inside-vault.com/vso_chart
Pushed: harbor.inside-vault.com/vso_chart/vault-secrets-operator:0.4.3
Digest: sha256:2b4e89478f314e9aa440533c6d84bf3955e3e372d1b36b38d9ac69c761fd265b

# 3. tar 파일 제거
rm vault-secrets-operator-0.4.3.tgz
# 3-1. tar 파일 제거 확인
ll
total 40
-rw-r--r-- 1 ec2-user ec2-user  1127 Jan 19 02:23 ca.crt
-rw-r--r-- 1 ec2-user ec2-user   235 Jan 11 02:04 Chart.yaml
drwxrwxr-x 2 ec2-user ec2-user  4096 Jan 19 03:38 crds
drwxrwxr-x 3 ec2-user ec2-user  4096 Jan 19 03:38 templates
-rw-r--r-- 1 ec2-user ec2-user 23598 Jan 19 03:52 values.yaml

```

- **Harbor 확인**
![image](https://github.com/jslim1995/insideinfo-vault/assets/124943887/ef162232-238d-429b-9128-35ca554c3fa6)

![image](https://github.com/jslim1995/insideinfo-vault/assets/124943887/860dee4d-bed4-4b7b-911c-3fe7ffd68da0)

![image](https://github.com/jslim1995/insideinfo-vault/assets/124943887/7a1a1684-61a4-4d90-a181-bc0b2ae012fb)

- **AWS S3 확인**
![image](https://github.com/jslim1995/insideinfo-vault/assets/124943887/8ced7438-ed60-4e25-a261-1cb40045ea55)


**(3-1) Pulling Helm Charts : harbor에 저장된 Chart 다운로드 후 설치 수행**

```bash
# Helm Pull Chart 수행 : 압축 파일 생성
# 1. Harbor에서 Helm chart Pull
# helm pull oci://<harbor_address>/<project>/<chart_name> --version <version>
helm pull oci://harbor.inside-vault.com/vso_chart/vault-secrets-operator --version 0.4.3
Pulled: harbor.inside-vault.com/vso_chart/vault-secrets-operator:0.4.3
Digest: sha256:0e9c0ea4878645ed526700834d6c29056cf6eeba7b73bfae4c7244271f3b552a
harbor.inside-vault.com/vso_chart/vault-secrets-operator:0.4.3 contains an underscore.

OCI artifact references (e.g. tags) do not support the plus sign (+). To support
storing semantic versions, Helm adopts the convention of changing plus (+) to
an underscore (_) in chart version tags when pushing to a registry and back to
a plus (+) when pulling from a registry.

# 2. 파일 확인
ll
total 60
-rw-r--r-- 1 ec2-user ec2-user  1127 Jan 19 02:23 ca.crt
-rw-r--r-- 1 ec2-user ec2-user   235 Jan 11 02:04 Chart.yaml
drwxrwxr-x 2 ec2-user ec2-user  4096 Jan 19 03:38 crds
drwxrwxr-x 3 ec2-user ec2-user  4096 Jan 19 03:38 templates
-rw-r--r-- 1 ec2-user ec2-user 23598 Jan 19 03:52 values.yaml
-rw-r--r-- 1 ec2-user ec2-user 18851 Jan 19 04:07 vault-secrets-operator-0.4.3.tgz

# 3. install 수행
helm install --create-namespace -n vsoperator vault-secrets-operator vault-secrets-operator-0.4.3.tgz
# 3-1. 배포 확인
kubectl get all -n vsoperator
NAME                                                             READY   STATUS    RESTARTS   AGE
pod/vault-secrets-operator-controller-manager-77dc869559-9dhgw   2/2     Running   0          14s

NAME                                             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/vault-secrets-operator-metrics-service   ClusterIP   10.100.26.122   <none>        8443/TCP   14s

NAME                                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/vault-secrets-operator-controller-manager   1/1     1            1           14s

NAME                                                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/vault-secrets-operator-controller-manager-77dc869559   1         1         1       14s


# 4. vault-secrets-operator-controller-manager pod : Events-pulled image 확인
# Successfully pulled image "harbor.inside-vault.com/library/gcr.io/kubebuilder/kube-rbac-proxy:v0.15.0"
# Successfully pulled image "harbor.inside-vault.com/library/vault-secrets-operator:0.4.3"
kubectl describe -n vsoperator pod vault-secrets-operator-controller-manager-77dc869559-9dhgw
...
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  95s   default-scheduler  Successfully assigned vsoperator/vault-secrets-operator-controller-manager-77dc869559-9dhgw to ip-172-31-21-112.ap-southeast-1.compute.internal
  Normal  Pulling    94s   kubelet            Pulling image "harbor.inside-vault.com/library/gcr.io/kubebuilder/kube-rbac-proxy:v0.15.0"
  Normal  Pulled     94s   kubelet            Successfully pulled image "harbor.inside-vault.com/library/gcr.io/kubebuilder/kube-rbac-proxy:v0.15.0" in 421ms (421ms including waiting)
  Normal  Created    94s   kubelet            Created container kube-rbac-proxy
  Normal  Started    94s   kubelet            Started container kube-rbac-proxy
  Normal  Pulling    94s   kubelet            Pulling image "harbor.inside-vault.com/library/vault-secrets-operator:0.4.3"
  Normal  Pulled     94s   kubelet            Successfully pulled image "harbor.inside-vault.com/library/vault-secrets-operator:0.4.3" in 226ms (226ms including waiting)
  Normal  Created    94s   kubelet            Created container manager
  Normal  Started    94s   kubelet            Started container manager
```

**(3-2) Installing OCI Helm Chart : harbor에 저장된 Chart 바로 사용하여 설치**

```bash
# 1. kubernetes namespace 생성
kubectl create namespace vsoperator
namespace/vsoperator created

# 1-1. kubernetes namespace 생성 확인
kubectl get namespace
NAME              STATUS   AGE
default           Active   29h
harbor            Active   23h
ingress-nginx     Active   24h
kube-node-lease   Active   29h
kube-public       Active   29h
kube-system       Active   29h
vsoperator        Active   3s

# 2. Helm install : harbor에서 Chart 가져와 사용
# helm install <release_name> oci://<harbor_address>/<project>/<chart_name> --version <version>
helm install -n vsoperator vault-secrets-operator oci://harbor.inside-vault.com/vso_chart/vault-secrets-operator --version 0.4.3
Pulled: harbor.inside-vault.com/vso_chart/vault-secrets-operator:0.4.3
Digest: sha256:2b4e89478f314e9aa440533c6d84bf3955e3e372d1b36b38d9ac69c761fd265b
harbor.inside-vault.com/vso_chart/vault-secrets-operator:0.4.3 contains an underscore.

OCI artifact references (e.g. tags) do not support the plus sign (+). To support
storing semantic versions, Helm adopts the convention of changing plus (+) to
an underscore (_) in chart version tags when pushing to a registry and back to
a plus (+) when pulling from a registry.
NAME: vault-secrets-operator
LAST DEPLOYED: Fri Jan 19 04:27:40 2024
NAMESPACE: vsoperator
STATUS: deployed
REVISION: 1

# 2-1. 설치 확인
kubectl get all -n vsoperator
NAME                                                             READY   STATUS    RESTARTS   AGE
pod/vault-secrets-operator-controller-manager-77dc869559-ngcq8   2/2     Running   0          32s

NAME                                             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/vault-secrets-operator-metrics-service   ClusterIP   10.100.254.21   <none>        8443/TCP   32s

NAME                                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/vault-secrets-operator-controller-manager   1/1     1            1           32s

NAME                                                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/vault-secrets-operator-controller-manager-77dc869559   1         1         1       32s


# 4. vault-secrets-operator-controller-manager pod : Events-pulled image 확인
# Successfully pulled image "harbor.inside-vault.com/library/gcr.io/kubebuilder/kube-rbac-proxy:v0.15.0"
# Successfully pulled image "harbor.inside-vault.com/library/vault-secrets-operator:0.4.3"
kubectl describe -n vsoperator pod vault-secrets-operator-controller-manager-77dc869559-ngcq8
```

