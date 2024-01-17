## 1. Kubernetes 구성 : **Argocd 설치**

```bash
# 1. K8s Namespace
kubectl create namespace argocd
namespace/argocd created

# 2. Argocd 설치 (Stable)
kubectl apply -n argocd -f <https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml>

# 2-1. Argocd 설치 확인
kubectl get all -n argocd

NAME                                                   READY   STATUS    RESTARTS   AGE
pod/argocd-application-controller-0                    1/1     Running   0          33s
pod/argocd-applicationset-controller-5f975ff5-7smm6    1/1     Running   0          34s
pod/argocd-dex-server-7bb445db59-fmbc7                 1/1     Running   0          34s
pod/argocd-notifications-controller-566465df76-2ctbr   1/1     Running   0          34s
pod/argocd-redis-6976fc7dfc-cxf44                      1/1     Running   0          34s
pod/argocd-repo-server-6d8d59bbc7-pkk64                1/1     Running   0          34s
pod/argocd-server-58f5668765-d27zk                     1/1     Running   0          34s

NAME                                              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/argocd-applicationset-controller          ClusterIP   10.100.68.88     <none>        7000/TCP,8080/TCP            34s
service/argocd-dex-server                         ClusterIP   10.100.202.230   <none>        5556/TCP,5557/TCP,5558/TCP   34s
service/argocd-metrics                            ClusterIP   10.100.48.78     <none>        8082/TCP                     34s
service/argocd-notifications-controller-metrics   ClusterIP   10.100.65.231    <none>        9001/TCP                     34s
service/argocd-redis                              ClusterIP   10.100.20.182    <none>        6379/TCP                     34s
service/argocd-repo-server                        ClusterIP   10.100.131.62    <none>        8081/TCP,8084/TCP            34s
service/argocd-server                             ClusterIP   10.100.193.81    <none>        80/TCP,443/TCP               34s
service/argocd-server-metrics                     ClusterIP   10.100.85.4      <none>        8083/TCP                     34s

NAME                                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/argocd-applicationset-controller   1/1     1            1           34s
deployment.apps/argocd-dex-server                  1/1     1            1           34s
deployment.apps/argocd-notifications-controller    1/1     1            1           34s
deployment.apps/argocd-redis                       1/1     1            1           34s
deployment.apps/argocd-repo-server                 1/1     1            1           34s
deployment.apps/argocd-server                      1/1     1            1           34s

NAME                                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/argocd-applicationset-controller-5f975ff5    1         1         1       34s
replicaset.apps/argocd-dex-server-7bb445db59                 1         1         1       34s
replicaset.apps/argocd-notifications-controller-566465df76   1         1         1       34s
replicaset.apps/argocd-redis-6976fc7dfc                      1         1         1       34s
replicaset.apps/argocd-repo-server-6d8d59bbc7                1         1         1       34s
replicaset.apps/argocd-server-58f5668765                     1         1         1       34s

NAME                                             READY   AGE
statefulset.apps/argocd-application-controller   1/1     34s

# 3. Argocd-Server : LB 접속 구성
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
service/argocd-server patched

# 3-1. LB 적용 확인
kubectl get svc argocd-server -n argocd
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP                                                                    PORT(S)                      AGE
argocd-server   LoadBalancer   10.100.193.81   a0d52df0e3e0344c3944789df28aeb7b-1139286844.ap-southeast-1.elb.amazonaws.com   80:31038/TCP,443:31146/TCP   6m13s

```



## 2. Argocd 구성 : Repository (Gitlab) 연결 생성

```bash
# 1. Argocd admin 계정 Password 조회
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
Y3Sol-1vkxyAFUem

# 2. Argocd  UI 접속 : argocd-server 서비스의 LoadBalancer EXTERNAL-IP 값 사용
a0d52df0e3e0344c3944789df28aeb7b-1139286844.ap-southeast-1.elb.amazonaws.com

# 3. Argocd Repository (GitLab) 연결 구성
# Argocd UI : Settings > Repositories > CONNECT REPO
```

- **ArgoCD - UI : Settings > Repositories > CONNECT REPO**

![image](https://github.com/jslim1995/insideinfo-vault/assets/124943887/dac30052-ded3-437b-915e-283bfef9b846)

![image](https://github.com/jslim1995/insideinfo-vault/assets/124943887/5e7cbf6c-6a41-457b-bcec-9ab0056c34f5)

![image](https://github.com/jslim1995/insideinfo-vault/assets/124943887/59103e9a-4337-4b3a-8dd0-22b9f5c8d3f1)


- (참고) ArgoCD - CLI

  ```bash
  # 1. admin Password 조회
  $ kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
  Y3Sol-1vkxyAFUem
  # 2. argocd-server LB 조회
  $ kubectl get svc argocd-server -n argocd
  NAME            TYPE           CLUSTER-IP      EXTERNAL-IP                                                                    PORT(S)                      AGE
  argocd-server   LoadBalancer   10.100.193.81   a0d52df0e3e0344c3944789df28aeb7b-1139286844.ap-southeast-1.elb.amazonaws.com   80:31038/TCP,443:31146/TCP   6m13s

  # 3. argocd-server 접속 및 로그인
  $ kubectl get pod -n argocd 
  argocd-server-58f5668765-d27zk                     1/1     Running   0          32m
  $ kubectl exec -it argocd-server-58f5668765-ztts6 -n argocd -- /bin/bash
  argocd@argocd-server-58f5668765-d27zk:~$ argocd login a0d52df0e3e0344c3944789df28aeb7b-1139286844.ap-southeast-1.elb.amazonaws.com
  WARNING: server certificate had error: tls: failed to verify certificate: x509: certificate is valid for localhost, argocd-server, argocd-server.argocd, argocd-server.argocd.svc, argocd-server.argocd.svc.cluster.local, not a0d52df0e3e0344c3944789df28aeb7b-1139286844.ap-southeast-1.elb.amazonaws.com. Proceed insecurely (y/n)? y
  Username: admin
  Password:
  'admin:login' logged in successfully
  Context 'a0d52df0e3e0344c3944789df28aeb7b-1139286844.ap-southeast-1.elb.amazonaws.com' updated

  # 4. Repo add
  # argocd repo add https://<REPOURL> --username <gitlab-username> --password <password>
  argocd@argocd-server-58f5668765-d27zk:~$ argocd repo add <http://3.99.145.122:8100/vault_team/vault_argocd.git> --username admin --password password

  # List
  $ argocd repo list
  TYPE  NAME  REPO                                                  INSECURE  OCI    LFS    CREDS  STATUS      MESSAGE  PROJECT
  git         <http://3.99.145.122:8100/vault_team/vault_argocd.git>  false     false  false  true   Successful           default

  ```



## 3. GitLab 구성 : arogcd-vault-plugin 파일 저장

- **arogcd-vault-plugin 파일 확인** 

  : {repository}/argocd-vault-plugin-1.14.0/arogcd-vault-plugin

![image](https://github.com/jslim1995/insideinfo-vault/assets/124943887/86c8686d-2b47-47cc-9a29-a34f3cdb0d24)

![image](https://github.com/jslim1995/insideinfo-vault/assets/124943887/7fb18c32-7aa8-4d46-bdae-37a52ba24a78)



## 4. Vault 작업 : KV 저장 및 Token 발급

**(1) KV 구성**

```bash
# 1. Vault Namespace 생성
vault namespace create shinhan
Key                Value
---                -----
custom_metadata    map[]
id                 TSEpa
path               shinhan/

# 2. KV secrets engine version 2 활성화
# Path : kvv2/
vault secrets enable -ns=shinhan  -path=kvv2 -version=2 kv
Success! Enabled the kv secrets engine at: kvv2/

# 2-1. KV secrets engine 생성 확인 (Path: kvv2/)
vault secrets list -ns=shinhan
Path          Type            Accessor                 Description
----          ----            --------                 -----------
cubbyhole/    ns_cubbyhole    ns_cubbyhole_c0e74533    per-token private secret storage
identity/     ns_identity     ns_identity_51ab6b33     identity store
kvv2/         kv              kv_eb56bef7              n/a
sys/          ns_system       ns_system_55d71bc4       system endpoints used for control, policy and debugging

# 3. KV secrets 저장 
# Path : kvv2/myapp/config
vault kv put -ns=shinhan  kvv2/myapp/config \
username="appuser" \
password="suP3rsec(et"
===== Secret Path =====
kvv2/data/myapp/config

======= Metadata =======
Key                Value
---                -----
created_time       2024-01-16T12:50:26.909552687Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1

# 3-1. kv-v2/myapp/config 값 확인
vault kv get -ns=shinhan kvv2/myapp/config
===== Secret Path =====
kvv2/data/myapp/config

======= Metadata =======
Key                Value
---                -----
created_time       2024-01-16T12:50:26.909552687Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1

====== Data ======
Key         Value
---         -----
password    suP3rsec(et
username    appuser

# 4. Policy 생성 : kv read  권한
vault policy write -ns=shinhan policy-kv2-myapp-config -<<EOF
path "kvv2/data/myapp/config" {
 capabilities=["read"]
}
EOF
Success! Uploaded policy: policy-kv2-myapp-config

# 4-1. Policy 생성 확인
vault policy read -ns=shinhan policy-kv2-myapp-config

path "kvv2/data/myapp/config" {
 capabilities=["read"]
}


```

**(2) Entity 구성**

```bash
# 1. Entity 생성
vault write -ns=shinhan -format=json identity/entity \
name=argocd-user \
policies=policy-kv2-myapp-config \
| jq -r ".data.id" > entity_id.txt
# 1-1. entity_id.txt 값 확인
cat entity_id.txt
74127201-7705-1329-6cbb-0d7d081760d1

# 1-2. Entity 생성 확인
vault read -ns=shinhan identity/entity/name/argocd-user
Key                    Value
---                    -----
aliases                []
creation_time          2024-01-16T12:50:54.891362959Z
direct_group_ids       []
disabled               false
group_ids              []
id                     74127201-7705-1329-6cbb-0d7d081760d1
inherited_group_ids    []
last_update_time       2024-01-16T12:50:54.891362959Z
merged_entity_ids      <nil>
metadata               <nil>
mfa_secrets            map[]
name                   argocd-user
namespace_id           TSEpa
policies               [policy-kv2-myapp-config]

# 2. Entity Alias 생성
# 2-1. Token Auth 조회
vault auth list -ns=shinhan -format=json | jq -r '.["token/"].accessor' > auth_token_accessor.txt

# 2-2. Token Auth mount_accessor 값 확인
cat auth_token_accessor.txt
auth_ns_token_7c73f994

# 2-3. Entity-Alias 생성
vault write -ns=shinhan identity/entity-alias \
name=argocd-user \
canonical_id=$(cat entity_id.txt) \
mount_accessor=$(cat auth_token_accessor.txt)
Key             Value
---             -----
canonical_id    74127201-7705-1329-6cbb-0d7d081760d1
id              7e6a8041-ad3f-1cc3-498f-701d84dbbd84

```

**(3) Token 발급**

```bash
# 1. Token Role 구성
# allowed_entity_aliases : 위에서 생성한 Entity Alias 이름
# token_bound_cidrs : Token을 사용할 수 있는 cidr 제한. K8s worker node 값으로 지정
vault write -ns=shinhan auth/token/roles/role-argocd-user \
allowed_entity_aliases=argocd-user \
orphan=true \
token_period=1d \
token_bound_cidrs="43.202.165.54/32,13.125.181.221/32" # 해당하는 값으로 변경
Success! Data written to: auth/token/roles/role-argocd-user

# 1-1. Token Role 구성 확인
vault read -ns=shinhan auth/token/roles/role-argocd-user
Key                         Value
---                         -----
allowed_entity_aliases      [argocd-user]
allowed_policies            []
allowed_policies_glob       []
disallowed_policies         []
disallowed_policies_glob    []
explicit_max_ttl            0s
name                        role-argocd-user
orphan                      true
path_suffix                 n/a
period                      0s
renewable                   true
token_bound_cidrs           [43.202.165.54 13.125.181.221]
token_explicit_max_ttl      0s
token_no_default_policy     false
token_period                24h
token_type                  default-service

# 2. Token 발급 : token role, entty-alias 사용
vault token create -ns=shinhan -role=role-argocd-user -entity-alias=argocd-user
Key                  Value
---                  -----
token                hvs.CAESICCqu2-tbSJWsm1G5T-KVk2haF2ESTXW0V6WX8-2pTqcGicKImh2cy5nN0trZjNBQTlock8xdllnMUZKeElqYWYuVFNFcGEQuAo
token_accessor       ptNa4poe9ARuqirHBR6TcJtB.TSEpa
token_duration       24h
token_renewable      true
token_policies       ["default"]
identity_policies    []
policies             ["default"]

```



## 5. GitLab 작업 : Vault Auth 정보 저장

- **Kubernetes Secret 정의**

  : Kubernetes Secret으로 Vault Auth 정보 저장

```yaml
# GitLab Directory: './vault-creds' 생성
# Kubernetes Secret YAML 작성 : Vault Auth 정보 저장
argocd-vault-plugin-credentials.yaml

apiVersion: v1
kind: Secret
metadata:
  name: argocd-vault-plugin-credentials
  namespace: argocd
stringData:
  VAULT_ADDR: "http://47.128.149.217:8200/"  # vault 접속 주소
  VAULT_NAMESPACE: "shinhan"  # vault auth가 구성된 vault namespace
  VAULT_TOKEN: "hvs.CAESIGjXWMdHZ1YAFUXMfV5TWvc6_hCSj44sfPrvbQb9YdK2GicKImh2cy5DOVJ5b3RBaGZSSGJhV2lKNXZhOXlaSHAueFJWS1MQ_go"  # 발급받은 vault token 입력
  AVP_TYPE: "vault"
  AVP_AUTH_TYPE: "token"
type: Opaque

```



## 6. Argocd 작업 : Vault Auth 정보 K8s Secret 배포 Application 생성

![image](https://github.com/jslim1995/insideinfo-vault/assets/124943887/fbfece4f-7084-46fc-92c9-2d1d885650d3)

![image](https://github.com/jslim1995/insideinfo-vault/assets/124943887/2035722f-76a6-4081-b231-7b68766220c2)



- **[Argocd] Application : vault-creds 생성 확인**

![image](https://github.com/jslim1995/insideinfo-vault/assets/124943887/143f8599-52d5-4c6a-8796-caa278f8fa18)

![image](https://github.com/jslim1995/insideinfo-vault/assets/124943887/a644a368-8c27-4c9e-a292-6628b13a8677)


- **[Kubernetes]에서 Applications 및 Kubernetes Secret (argocd-vault-plugin-credentials) 생성 확인**

```bash
# 1. argocd/Applications 생성 확인
# Argocd application : vault-creds 생성
kubectl get applications -n argocd
NAME          SYNC STATUS   HEALTH STATUS
vault-creds   Synced        Healthy

# 2. Kubernetes Secret : argocd-vault-plugin-credentials 생성 확인 
# 2-1. GitLab에 작성한 argocd-vault-plugin-credentials.yaml 파일 배포 확인
kubectl get secrets -n argocd
NAME                              TYPE     DATA   AGE
argocd-initial-admin-secret       Opaque   1      12m
argocd-notifications-secret       Opaque   0      12m
argocd-secret                     Opaque   5      12m
argocd-vault-plugin-credentials   Opaque   5      101s
repo-3095605567                   Opaque   5      9m34s

# 2-2. Kubernetes Secret: argocd-vault-plugin-credentials 정보 확인
kubectl describe -n argocd secret argocd-vault-plugin-credentials
Name:         argocd-vault-plugin-credentials
Namespace:    argocd
Labels:       app.kubernetes.io/instance=vault-creds
Annotations:  <none>
Type:  Opaque
Data
====
AVP_AUTH_TYPE:    5 bytes
AVP_TYPE:         5 bytes
VAULT_ADDR:       57 bytes
VAULT_NAMESPACE:  8 bytes
VAULT_TOKEN:      107 bytes

# 2-3. Kubernetes Secret: argocd-vault-plugin-credentials 값 확인
$ kubectl get secret argocd-vault-plugin-credentials -o jsonpath="{.data}" -n argocd | jq
{
  "AVP_AUTH_TYPE": "dG9rZW4=",
  "AVP_TYPE": "dmF1bHQ=",
  "VAULT_ADDR": "aHR0cDovLzQ3LjEyOC4xNDkuMjE3OjgyMDA=",
  "VAULT_NAMESPACE": "c2hpbmhhbg==",
  "VAULT_TOKEN": "aHZzLkNBRVNJS25PaHo3UTNZTzczN19mV0FDQkZXWWJBWktMdExESzAzTzRCSmVoRzJMdkdpY0tJbWgyY3k1cmVtcENNV2xRVjNrek1sUjVhRmhwZEV0Q2J6Uk1Zek11VjBaeU5UWVExd1k="
}
$ kubectl get secret argocd-vault-plugin-credentials -o jsonpath="{.data.VAULT_NAMESPACE}" -n argocd | base64 -d; echo
shinhan
$ kubectl get secret argocd-vault-plugin-credentials -o jsonpath="{.data.VAULT_TOKEN}" -n argocd | base64 -d; echo
hvs.CAESIO7juMmTJfSBOsERXu_CSXMK-7J5g6cNj13CI0mYjvjsGicKImh2cy5WRVlNeldwdk1xc2JSMGd5NkE0RjBRVTEuaGZjZWcQ1QU

```



## 7. Kubernetes 작업 : Rolebinding 생성

위에서 생성한 'Secret: argocd-vault-plugin-credentials'를 읽을 수 있는 Role 생성하여 'ServiceAccount:argocd-repo-server'에 적용

```bash
# 생성 전
kubectl get RoleBinding -n argocd
NAME                               ROLE                                    AGE
argocd-application-controller      Role/argocd-application-controller      37m
argocd-applicationset-controller   Role/argocd-applicationset-controller   37m
argocd-dex-server                  Role/argocd-dex-server                  37m
argocd-notifications-controller    Role/argocd-notifications-controller    37m
argocd-server                      Role/argocd-server                      37m

# 1. Role 및 RoleBinding Yaml 작성
# 위에서 생성한 'Secret: argocd-vault-plugin-credentials'를 읽을 수 있는 Role 생성하여 'ServiceAccount:argocd-repo-server'에 적용
tee argocd-repo-server-role.yaml <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: argocd-repo-server
  namespace: argocd
rules:
- apiGroups:
  - ""
  resources:
  - secrets
  verbs:
  - get
  - watch
  resourceNames:
  - argocd-vault-plugin-credentials
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: argocd-repo-server
  namespace: argocd
subjects:
- kind: ServiceAccount
  name: argocd-repo-server
  namespace: argocd
roleRef:
  kind: Role
  name: argocd-repo-server
  apiGroup: rbac.authorization.k8s.io
EOF

# 1-1. Role 및 RoleBinding 생성
kubectl apply -f argocd-repo-server-role.yaml
role.rbac.authorization.k8s.io/argocd-repo-server created
rolebinding.rbac.authorization.k8s.io/argocd-repo-server created

# 1-2. Role 생성 확인
# Resource Names : [argocd-vault-plugin-credentials]
kubectl describe role -n argocd argocd-repo-server

Name:         argocd-repo-server
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources  Non-Resource URLs  Resource Names                     Verbs
  ---------  -----------------  --------------                     -----
  secrets    []                 [argocd-vault-plugin-credentials]  [get watch]

# 1-3. Rolebinding 생성 확인
# Role : argocd-repo-server
# Subjects(대상) : ServiceAccount argocd-repo-server 확인
kubectl describe rolebinding -n argocd argocd-repo-server

Name:         argocd-repo-server
Labels:       <none>
Annotations:  <none>
Role:
  Kind:  Role
  Name:  argocd-repo-server
Subjects:
  Kind            Name                Namespace
  ----            ----                ---------
  ServiceAccount  argocd-repo-server  argocd

```



## 8. Kubernetes 작업 : Arogcd-Vault-Plugin 정의

```yaml
# 1. ConfigMap: cmp-plugin 작성
# : data로 plugin.yaml 정의
# : plugin은 Vault Auth 정보 저장한 K8s Secret 참조
vi cmp-plugin.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cmp-plugin
  namespace: argocd
data:
  argocd-vault-plugin.yaml: |
    apiVersion: argoproj.io/v1alpha1
    kind: ConfigManagementPlugin
    metadata:
      name: argocd-vault-plugin  # ArgoCD에서 출력되는 Plugin name
    spec:
      allowConcurrency: true
      discover:
        find:
          command:
            - sh
            - "-c"
            - "find . -name '*.yaml' | xargs -I {} grep \"<path\\|avp\\.kubernetes\\.io\" {} | grep ."
      generate:
        command:
          - argocd-vault-plugin
          - generate
          - "-s"
          - "argocd-vault-plugin-credentials"
          - "."
      lockRepo: false

# 1-1. ConfigMap: cmp-plugin 생성
kubectl apply -f cmp-plugin.yaml
configmap/cmp-plugin configured

# 1-2. ConfigMap: cmp-plugin 생성 확인
kubectl get configmap -n argocd
NAME                        DATA   AGE
argocd-cm                   0      122m
argocd-cmd-params-cm        0      122m
argocd-gpg-keys-cm          0      122m
argocd-notifications-cm     0      122m
argocd-rbac-cm              0      122m
argocd-ssh-known-hosts-cm   1      122m
argocd-tls-certs-cm         0      122m
cmp-plugin                  1      21m
kube-root-ca.crt            1      122m

# 1-3. ConfigMap: cmp-plugin 값(Data) 확인
kubectl describe configmap -n argocd cmp-plugin
Name:         cmp-plugin
Namespace:    argocd
Labels:       <none>
Annotations:  <none>

Data
====
argocd-vault-plugin.yaml:
----
apiVersion: argoproj.io/v1alpha1
kind: ConfigManagementPlugin
metadata:
  name: argocd-vault-plugin
spec:
  allowConcurrency: true
  discover:
    find:
      command:
        - sh
        - "-c"
        - "find . -name '*.yaml' | xargs -I {} grep \"<path\|avp\.kubernetes\.io                            \" {} | grep ."
  generate:
    command:
      - argocd-vault-plugin
      - generate
      - "-s"
      - "argocd-vault-plugin-credentials"
      - "."
  lockRepo: false


BinaryData
====

Events:  <none>
```



## 9. Kubernetes 작업 : Arogcd-Vault-Plugin 생성

- **[GitLab]** arogcd-vault-plugin 파일 확인

![image](https://github.com/jslim1995/insideinfo-vault/assets/124943887/86c8686d-2b47-47cc-9a29-a34f3cdb0d24)

![image](https://github.com/jslim1995/insideinfo-vault/assets/124943887/7fb18c32-7aa8-4d46-bdae-37a52ba24a78)

- **[Kubernetes] Deployment Argocd-Repo-Server 수정 : arogcd-vault-plugin 추가**

```yaml
# 1. Argocd repo server에 Plugin 적용
kubectl edit -n argocd deployment argocd-repo-server

apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-repo-server
spec:
  template:
    spec:
      # (1) automountServiceAccountToken : false → true 변경
      automountServiceAccountToken: true
      
      # (2) Volumes 추가 : cmp-plugins, custom-tools
      volumes:
      - configMap:
          name: cmp-plugin
        name: cmp-plugin
      - name: custom-tools
        emptyDir: {}
        
      # (3) initContainers 수정 
      # argocd-vault-plugin 설치 : GitLab에 올려둔 파일을 curl 통해 가져옴
      # volume : custom-tools에 저장
      initContainers:
      - name: download-tools
        image: registry.access.redhat.com/ubi8/ubi:8.9-1107
        command: [sh, -c]
        args:
          - >-
            curl -L "http://3.99.145.122:8100/api/v4/projects/2/repository/files/argocd-vault-plugin-1.14.0%2Fargocd-vault-plugin/raw?ref=main&private_token=glpat-gupzj4js8zx3fsYRxeTp" -o argocd-vault-plugin &&
            chmod +x argocd-vault-plugin &&
            mv argocd-vault-plugin /custom-tools/
        volumeMounts:
          - mountPath: /custom-tools
            name: custom-tools
 
      # (4) containers : argocd-vault-plugin 추가 
      # sidecar container = Plugin 
      containers:
      ...(생략)
      - name: argocd-vault-plugin
        command: [/var/run/argocd/argocd-cmp-server]
        image: alpine
        securityContext:
          runAsNonRoot: true
          runAsUser: 999
        volumeMounts:
        - mountPath: /var/run/argocd
          name: var-files
        - mountPath: /home/argocd/cmp-server/plugins
          name: plugins
        - mountPath: /tmp
          name: tmp
          # 위에서 생성한 plugin 정의해둔 cmp-plugin: cmp-plugin 사용
        - mountPath: /home/argocd/cmp-server/config/plugin.yaml
          name: cmp-plugin
          subPath: argocd-vault-plugin.yaml
          # initContainers에서 받은 argocd-vault-plugin 사용
        - mountPath: /usr/local/bin/argocd-vault-plugin
          name: custom-tools
          subPath: argocd-vault-plugin
          
# 2. Container 추가 확인 : argocd-vault-plugin
# 2-1. argocd-repo-server Pod 조회
kubectl get pod -n argocd
NAME                                               READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                    1/1     Running   0          20h
argocd-applicationset-controller-5f975ff5-l6rkb    1/1     Running   0          20h
argocd-dex-server-7bb445db59-cmphk                 1/1     Running   0          20h
argocd-notifications-controller-566465df76-qjbqb   1/1     Running   0          20h
argocd-redis-6976fc7dfc-9xqb4                      1/1     Running   0          20h
argocd-repo-server-5f4bc446-fjctt                  2/2     Running   0          3m55s
argocd-server-58f5668765-ztts6                     1/1     Running   0          20h

# 2-2. argocd-repo-server Pod에서 실행 중인 container 목록에 'argocd-vault-plugin' 추가 확인
# kubectl get pod <pod_name> -o jsonpath='{.spec.containers[*].name}'
# : pod_name 변경
kubectl get pod argocd-repo-server-5f4bc446-fjctt -n argocd -o jsonpath='{.spec.containers[*].name}'

argocd-vault-plugin argocd-repo-server
```



## 10. GitLab 작업 : Pod 생성 YAML 저장

: Pod의 env에 vault secret 연동

```yaml
# GitLab Directory: './MyApp' 생성
# Pod 생성 YAML 저장
# env에서 vault secret 연동
vault-kv2-pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: vault-kv2-pod
  namespace: default
spec:
  serviceAccountName: default
  containers:
    - name: vault-kv2
      image: alpine
      command: ["sleep","infinity"]
      env:
        - name: Username
          value: <path:kvv2/data/myapp/config#username>  # Vault KV path(kvv2/data/myapp/config) 및 Key(username) 입력
        - name: Password
          value: <path:kvv2/data/myapp/config#password>  # Vault KV path(kvv2/data/myapp/config) 및 Key(password) 입력

```



## 11. Argocd 작업 : Pod 배포 Application 생성

- **[Argocd] Application : vault-myapp 생성 시, Plugin 선택**

![image](https://github.com/jslim1995/insideinfo-vault/assets/124943887/9b3478c3-8046-466e-b531-44e293eb8a1d)

![image](https://github.com/jslim1995/insideinfo-vault/assets/124943887/af5399d1-9e48-4715-8c1c-07ab619d3705)

![image](https://github.com/jslim1995/insideinfo-vault/assets/124943887/11e89d95-73af-4e42-a149-a9eaeafda8b5)

![image](https://github.com/jslim1995/insideinfo-vault/assets/124943887/87193a66-fe07-4983-ba58-a901d0a86d1b)

- **[Argocd] Application : vault-myapp 생성 확인**

![image](https://github.com/jslim1995/insideinfo-vault/assets/124943887/8e2f6c16-1e64-40c2-85b9-b04f231a200a)

- **[Kubernetes] Secret : Argocd application 및 Pod (vault-kv2-pod) Environment 확인**

```bash
# 1. argocd/Applications 생성 확인
# Argocd application : myapp 생성
kubectl get applications -n argocd
NAME          SYNC STATUS   HEALTH STATUS
myapp         Synced        Healthy
vault-creds   Synced        Healthy

# 2. Pod 생성 확인
# Pod : vault-kv2-pod 생성
kubectl get pod
NAME            READY   STATUS    RESTARTS   AGE
vault-kv2-pod   1/1     Running   0          18m

# 3. Pod : Containers(vault-kv2) Environment 확인
kubectl describe pod vault-kv2-pod
...(생략)
Containers:
  vault-kv2:
    Container ID:  containerd://6fdc5ce21957fe93d85bd052ac0148ebcdc2942852fb5e477d1cca9c16a06dc8
    Image:         alpine
    Image ID:      docker.io/library/alpine@sha256:51b67269f354137895d43f3b3d810bfacd3945438e94dc5ac55fdac340352f48
    Port:          <none>
    Host Port:     <none>
    Command:
      sleep
      infinity
    State:          Running
      Started:      Tue, 16 Jan 2024 23:12:10 +0000
    Ready:          True
    Restart Count:  0
    Environment:
      Username:  appuser
      Password:  suP3rsec(et
...(생략)

# 3. Pod에서 env 출력 : Username, Password 값이 Vault KV 값과 일치 확인
kubectl exec -it vault-kv2-pod -n default -- env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=vault-kv2-pod
Username=appuser
Password=suP3rsec(et
...(생략)

```
