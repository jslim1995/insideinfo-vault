## 1. **Vault 작업 : KV 구성**

```bash
# 1. Vault Namespace 생성
vault namespace create shinhan

Key                Value
---                -----
custom_metadata    map[]
id                 ExaRF
path               shinhan/

# 2. KV secrets engine version 2 활성화
# Path : kvv2/
vault secrets enable -ns=shinhan  -path=kvv2 -version=2 kv
Success! Enabled the kv secrets engine at: kvv2/

# 2-1. KV secrets engine 생성 확인 (Path: kvv2/)
vault secrets list -ns=shinhan

Path          Type            Accessor                 Description
----          ----            --------                 -----------
cubbyhole/    ns_cubbyhole    ns_cubbyhole_16a518c1    per-token private secret storage
identity/     ns_identity     ns_identity_ca453a9e     identity store
kvv2/         kv              kv_39400b33              n/a
sys/          ns_system       ns_system_83ee2787       system endpoints used for control, policy and debugging

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
created_time       2024-01-16T07:42:48.4834714Z
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
created_time       2024-01-16T07:42:48.4834714Z
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
vault policy write -ns=shinhan policy-kv2-myapp-config <<EOF
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



## 2. **Kubernetes 작업 : ServiceAccount구성**

**(1) vault-account**

: vault의 kubernetes auth 구성 시 사용

- **K8s Namespace :** vsoperator
- **ServiceAccount :** vault-account
- **ClusterRoleBinding : ** 'system:auth-delegator' 연결
- **Token (secret) :** vault-account-token

```bash
# 1. namespace(vsoperator) 생성
kubectl create namespace vsoperator
namespace/vsoperator created

# 2. Service Account 1 : 'vsoperator/vault-account' YAML 작성
# vault의 kubernetes auth 구성 시 사용
tee sa-vault-account.yaml <<EOF
# ServiceAccount : vault-account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vault-account
  namespace: vsoperator
---
# ClusterRoleBinding 
# ServiceAccount 'vault-account'에 ClusterRole 'system:auth-delegator'을 부여하여 인증 작업 위임
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: role-tokenreview-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: vault-account
  namespace: vsoperator
---
# (JWT)service-account-token 생성
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: vault-account-token
  namespace: vsoperator
  annotations:
    kubernetes.io/service-account.name: "vault-account"
EOF

# 2-1. 'vsoperator/vault-account' 생성
kubectl apply -f sa-vault-account.yaml
serviceaccount/vault-account created
clusterrolebinding.rbac.authorization.k8s.io/role-tokenreview-binding created
secret/vault-account-token created

# 2-2. 'vsoperator/vault-account' 생성 확인
# serviceaccount 목록에 vault-account 추가 확인
kubectl get sa -n vsoperator
NAME                                        SECRETS   AGE
default                                     0         14m
vault-account                               0         83s

# 2-3. serviceaccount : Name, Namespace, Tokens 값 확인
kubectl describe sa vault-account -n vsoperator
Name:                vault-account
Namespace:           vsoperator
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   <none>
Tokens:              vault-account-token
Events:              <none>

# 2-4. clusterrolebinding: role-tokenreview-binding 생성 확인 
kubectl get clusterrolebinding | grep role-tokenreview-binding
role-tokenreview-binding        ClusterRole/system:auth-delegator        111s

# 2-5. 'vsoperator/vault-account'의 Token: vault-account-token 생성 확인 
kubectl get secret -n vsoperator | grep vault-account-token
vault-account-token   kubernetes.io/service-account-token   3      3m16s

```

**(2) vso-account**

: kubernetes에서 vso 통해 vault kv 접근 시, vault kubernetes auth를 위해 사용

- **K8s Namespace :** app
- **ServiceAccount :** vso-account

```bash
# 1. namespace: app 생성
kubectl create namespace app
namespace/app created

# 2. Service Account 2 : 'app/vso-account' YAML 작성
# kubernetes에서 vso 통해 vault kv 접근 시, vault kubernetes auth를 위해 사용
tee sa-vso-account.yaml 
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vso-account
  namespace: app

# 2-1. 'app/vso-account' 생성
kubectl apply -f sa-vso-account.yaml
serviceaccount/vso-account created

# 2-2. 'app/vso-account' 생성 확인
# serviceaccount 목록에 vso-account 추가 확인
kubectl get sa -n app
NAME          SECRETS   AGE
default       0         4m32s
vso-account   0         2m6s

# serviceaccount : Name, Namespace 값 확인
kubectl describe sa vso-account -n app
Name:                vso-account
Namespace:           app
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   <none>
Tokens:              <none>
Events:              <none>

```



## 3. **Kubernetes 작업 :** Vault Auth 구성을 위한 Kubernetes 정보 조회

- **Secret :** vault-account-token 값 (*ServiceAccount : vault-account의 Token 사용)
- **Host endpoint :** Kuebrnetes API server Endpoint
- **CA.cert :** Kubernetes Cluster의 CA(certificate-authority-data)

```bash
# 5. vault kubernetes auth 설정을 위한 값 조회
# 5-1. JWT : 'vsoperator/vault-account'의 Token
# ClusterRole 'system:auth-delegator'을 부여한 ServiceAccount의 Token
kubectl get secret vault-account-token -n vsoperator -ojsonpath='{.data.token}' | base64 -d
eyJhbGciOiJSUzI1NiIsImtpZCI6IlppTHgxdk5rVTBvZGw2U1V1YmRQTmJZNktWa0dwMkRFcjZENnJ6dDdQRDQifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJ2c29wZXJhdG9yIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6InZhdWx0LWFjY291bnQtdG9rZW4iLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoidmF1bHQtYWNjb3VudCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImYwODhmODkzLWU1MTAtNDkzMy04MTc5LTlhN2EwZGZjNTM0ZSIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDp2c29wZXJhdG9yOnZhdWx0LWFjY291bnQifQ.vitw3Us_lmiwxbdHU01xYqBF_j05PdEVsccIXhK8Mc8O9jkKWMeJ7rIf_6oGm8O4Brn-hjqqQHWVY2kVG0g4YibkBUkSP0hhd87Wkuw7Yep47xFvFia5zeadlxwr6T7hb45cd1yL7AGxZi2Va6fEG2VfmLNKGBUrr3sng5yChARVcQ28KtG8rav7CEA-7krFZsc7K7TJyaxgWgo6gJTGjfGm-rz14c-KcRUgTVh1e8GewMmyq6sNtFDMBNUhSyZaWIVg9HRtD57C6dIiNCbK60dfPW1h-8clJ3LZZPeTBT0Aq19uNGT1hNenVvhVkfIzg3gki1TyipG13uMDgXbg_w

# 5-2. Host endpoint
# Kuebrnetes API server Endpoint
kubectl config view --raw --minify --flatten --output='jsonpath={.clusters[].cluster.server}'
<https://D3943FF0D40F4E7351CC52E4A25B4E57.gr7.ap-southeast-1.eks.amazonaws.com>

# 5-3. CA.cert
# Kubernetes Cluster의 CA(certificate-authority-data)
kubectl config view --raw --minify --flatten --output='jsonpath={.clusters[].cluster.certificate-authority-data}' | base64 --decode
-----BEGIN CERTIFICATE-----
MIIDBTCCAe2gAwIBAgIIQ9/4pnaTWWswDQYJKoZIhvcNAQELBQAwFTETMBEGA1UE
AxMKa3ViZXJuZXRlczAeFw0yMzEyMjIwMDMzMjVaFw0zMzEyMTkwMDM4MjVaMBUx
EzARBgNVBAMTCmt1YmVybmV0ZXMwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEK
AoIBAQCfJIxMPr47N83aVX1RukV1zQigyIQAfZD3YA+6DTFMLGhHoGg8m8hB4L6U
uzYt26C5vluVSScN2NZAwW7wvaJu+Ewkihkhlha/Jxn2JFu4xUqCnQM1UGsHKiwE
jLoQpioee0x4ycqEOPhpgD9Rk3CDdVGI4G9vTBOBJ0nofLyZiDdQGlO26bIkEbcY
DuXRUNVCEZEWOp05r54Q89uY2ltaZpFzt2epCBfn0jNP0GiNJnr9a8vzGGltFPxb
ziwC+oB71DJ/OkMfb3O+7xGFWR21ayieVPGl+1SoGa967OVHeb6l9XU+zb7ryQQW
5p2lmFw6zqdxyM/Se5fPh1GLR3szAgMBAAGjWTBXMA4GA1UdDwEB/wQEAwICpDAP
BgNVHRMBAf8EBTADAQH/MB0GA1UdDgQWBBRGlCf5OMj7fXghjxFr17EfUhqmvDAV
BgNVHREEDjAMggprdWJlcm5ldGVzMA0GCSqGSIb3DQEBCwUAA4IBAQCeCILYreGj
+1JnNNCC/bMNaKL8Yp14A58LXGfvE5D/8Cm11+tkkHrUZlxjH1MHLMMrDQnnS6CT
cuV7DXB5KMKilvFKPFANF7t+vSU4xlLtGHfi7pR6YzZ+N1Jb4wyB7cWTVHeYKqfP
dKlQ/Y0c+ZP2UmrRS9bj9+bSt0wdSToix/2zIzdBNlNAoF1Box9OImZEhazlWWtz
U1itkDhEqnbohd4BLe/cWEGdbaFm97VCNuBedg2gmVPX0EVI2SqpA0ozCyYFimbQ
7cGU+r0vTLTYyPIbw7aK5I2C+/BZebETe63u9jGp+c+ZDy4sz1ON7G91JdLoFsAI
LQ+AKizmhsJE
-----END CERTIFICATE-----

```



## 4. **Vault 작업 : Kubernetes Auth 구성**

- **Kubernetes Auth 구성**

```bash
# 1. 위에서 조회한 k8s 사전 정보 저장
# 1-1. JWT : 'vsoperator/vault-account'의 Token
# ClusterRole 'system:auth-delegator'을 부여한 ServiceAccount의 Token
K8S_TOKEN=eyJhbGciOiJSUzI1NiIsImtpZCI6IlppTHgxdk5rVTBvZGw2U1V1YmRQTmJZNktWa0dwMkRFcjZENnJ6dDdQRDQifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJ2c29wZXJhdG9yIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6InZhdWx0LWFjY291bnQtdG9rZW4iLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoidmF1bHQtYWNjb3VudCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImYwODhmODkzLWU1MTAtNDkzMy04MTc5LTlhN2EwZGZjNTM0ZSIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDp2c29wZXJhdG9yOnZhdWx0LWFjY291bnQifQ.vitw3Us_lmiwxbdHU01xYqBF_j05PdEVsccIXhK8Mc8O9jkKWMeJ7rIf_6oGm8O4Brn-hjqqQHWVY2kVG0g4YibkBUkSP0hhd87Wkuw7Yep47xFvFia5zeadlxwr6T7hb45cd1yL7AGxZi2Va6fEG2VfmLNKGBUrr3sng5yChARVcQ28KtG8rav7CEA-7krFZsc7K7TJyaxgWgo6gJTGjfGm-rz14c-KcRUgTVh1e8GewMmyq6sNtFDMBNUhSyZaWIVg9HRtD57C6dIiNCbK60dfPW1h-8clJ3LZZPeTBT0Aq19uNGT1hNenVvhVkfIzg3gki1TyipG13uMDgXbg_w

# 1-2. Host endpoint
# Kuebrnetes API server Endpoint
K8S_HOST=https://D3943FF0D40F4E7351CC52E4A25B4E57.gr7.ap-southeast-1.eks.amazonaws.com

# 1-3. CA.cert
# Kubernetes Cluster의 CA(certificate-authority-data)
cat > K8S.crt -<<EOF
-----BEGIN CERTIFICATE-----
MIIDBTCCAe2gAwIBAgIIQ9/4pnaTWWswDQYJKoZIhvcNAQELBQAwFTETMBEGA1UE
AxMKa3ViZXJuZXRlczAeFw0yMzEyMjIwMDMzMjVaFw0zMzEyMTkwMDM4MjVaMBUx
EzARBgNVBAMTCmt1YmVybmV0ZXMwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEK
AoIBAQCfJIxMPr47N83aVX1RukV1zQigyIQAfZD3YA+6DTFMLGhHoGg8m8hB4L6U
uzYt26C5vluVSScN2NZAwW7wvaJu+Ewkihkhlha/Jxn2JFu4xUqCnQM1UGsHKiwE
jLoQpioee0x4ycqEOPhpgD9Rk3CDdVGI4G9vTBOBJ0nofLyZiDdQGlO26bIkEbcY
DuXRUNVCEZEWOp05r54Q89uY2ltaZpFzt2epCBfn0jNP0GiNJnr9a8vzGGltFPxb
ziwC+oB71DJ/OkMfb3O+7xGFWR21ayieVPGl+1SoGa967OVHeb6l9XU+zb7ryQQW
5p2lmFw6zqdxyM/Se5fPh1GLR3szAgMBAAGjWTBXMA4GA1UdDwEB/wQEAwICpDAP
BgNVHRMBAf8EBTADAQH/MB0GA1UdDgQWBBRGlCf5OMj7fXghjxFr17EfUhqmvDAV
BgNVHREEDjAMggprdWJlcm5ldGVzMA0GCSqGSIb3DQEBCwUAA4IBAQCeCILYreGj
+1JnNNCC/bMNaKL8Yp14A58LXGfvE5D/8Cm11+tkkHrUZlxjH1MHLMMrDQnnS6CT
cuV7DXB5KMKilvFKPFANF7t+vSU4xlLtGHfi7pR6YzZ+N1Jb4wyB7cWTVHeYKqfP
dKlQ/Y0c+ZP2UmrRS9bj9+bSt0wdSToix/2zIzdBNlNAoF1Box9OImZEhazlWWtz
U1itkDhEqnbohd4BLe/cWEGdbaFm97VCNuBedg2gmVPX0EVI2SqpA0ozCyYFimbQ
7cGU+r0vTLTYyPIbw7aK5I2C+/BZebETe63u9jGp+c+ZDy4sz1ON7G91JdLoFsAI
LQ+AKizmhsJE
-----END CERTIFICATE-----
EOF

# 2. Vault Kubernetes Auth 활성화
vault auth enable -ns=shinhan kubernetes
Success! Enabled kubernetes auth method at: kubernetes/

# 3. Vault Kubernetes Auth : Config 설정
# K8S_TOKEN, K8S_TOKEN, K8S.crt 값 사용
vault write -ns=shinhan auth/kubernetes/config \
token_reviewer_jwt=${K8S_TOKEN} \
kubernetes_host=${K8S_HOST} \
kubernetes_ca_cert=@K8S.crt
Success! Data written to: auth/kubernetes/config
```

- **Entity 구성**

```bash
# 1. Entity 생성
# name='app/vso-account'의 형식 : '{k8s_namespace}/{k8s_serviceaccount}' 
vault write -ns=shinhan -format=json identity/entity \
name="app/vso-account" \
policies=policy-kv2-myapp-config \
| jq -r ".data.id" > entity_id.txt
cat entity_id.txt
bcd27694-3ca9-518b-801c-45e16c25516b

# 2. Entity Alias 구성: Vault Kubernetes Auth와 연결
# 2-1. Kubernetes Auth의 Mount Accessor 조회
vault auth list -ns=shinhan -format=json | jq -r '.["kubernetes/"].accessor' > auth_token_accessor.txt
cat auth_token_accessor.txt
auth_kubernetes_b1acc008
# 2-2. Entity Alias 생성
vault write -ns=shinhan identity/entity-alias \
name="app/vso-account" \
canonical_id=$(cat entity_id.txt) \
mount_accessor=$(cat auth_token_accessor.txt)
Key             Value
---             -----
canonical_id    bcd27694-3ca9-518b-801c-45e16c25516b
id              02767c2e-6285-19c6-febb-f54466070dbf

# 3. Kubernetes Auth의 Role 생성: Kubernetes ServiceAccount와 연결
# bound_service_account_names : 연결할 kubernetes ServiceAccount 이름
# bound_service_account_namespaces : 연결할 kubernetes ServiceAccount이 있는 namespace
# alias_name_source : '{k8s_namespace}/{k8s_serviceaccount}' 형식의 Vault Entity Alias 사용
vault write -ns=shinhan auth/kubernetes/role/vsoperator \
bound_service_account_names=vso-account \
bound_service_account_namespaces=app \
ttl=1d \
max_ttl=1d \
alias_name_source=serviceaccount_name

```



## 5. **Kubernetes-Helm 작업 :** vault-secrets-operator **설치**

```bash
# 1. Helm : vault-secrets-operator 설치 
# 1-1. vault-secrets-operator-0.4.3.tgz 파일 확인
ll
total 28
-rw-rw-r-- 1 ec2-user ec2-user   588 Jan 16 07:45 sa-vault-account.yaml
-rw-rw-r-- 1 ec2-user ec2-user    84 Jan 16 07:54 sa-vso-account.yaml
-rw-r--r-- 1 ec2-user ec2-user 18816 Jan 16 04:48 vault-secrets-operator-0.4.3.tgz
# 1-2. Helm Install 수행
helm install --create-namespace --namespace vsoperator vault-secrets-operator vault-secrets-operator-0.4.3.tgz
NAME: vault-secrets-operator
LAST DEPLOYED: Tue Jan 16 08:32:32 2024
NAMESPACE: vsoperator
STATUS: deployed
REVISION: 1

# 2. Helm 설치 확인
helm list -n vsoperator
NAME                    NAMESPACE       REVISION        UPDATED                                 STATUS   CHART                           APP VERSION
vault-secrets-operator  vsoperator      1               2024-01-16 08:32:32.269181129 +0000 UTC deployed vault-secrets-operator-0.4.3    0.4.3

# 3. VSO CRD 생성 확인
# 생성되는 CRDs : hcpauths, hcpvaultsecretsapps, vaultauths, vaultconnections, vaultdynamicsecrets, vaultpkisecrets, vaultstaticsecrets
# Vault Enterprise : vaultauths, vaultconnections, vaultdynamicsecrets, vaultpkisecrets, vaultstaticsecrets 사용
kubectl get Customresourcedefinitions
NAME                                         CREATED AT
cninodes.vpcresources.k8s.aws                2023-12-22T00:39:06Z
eniconfigs.crd.k8s.amazonaws.com             2023-12-22T00:39:03Z
hcpauths.secrets.hashicorp.com               2024-01-16T08:34:08Z
hcpvaultsecretsapps.secrets.hashicorp.com    2024-01-16T08:34:08Z
policyendpoints.networking.k8s.aws           2023-12-22T00:39:03Z
securitygrouppolicies.vpcresources.k8s.aws   2023-12-22T00:39:06Z
vaultauths.secrets.hashicorp.com             2024-01-16T08:34:08Z
vaultconnections.secrets.hashicorp.com       2024-01-16T08:34:08Z
vaultdynamicsecrets.secrets.hashicorp.com    2024-01-16T08:34:08Z
vaultpkisecrets.secrets.hashicorp.com        2024-01-16T08:34:08Z
vaultstaticsecrets.secrets.hashicorp.com     2024-01-16T08:34:08Z

```



## 6. **Kubernetes 작업 : VSO 적용**

```yaml
# 1. VaultConnection Yaml 작성
# Vault 서버 접속 정보 지정
tee vaultconnection.yaml <<EOF
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultConnection
metadata:
  namespace: app
  name: vault-connection
spec:
  address: http://47.128.149.217:8200  # Vault 접속 주소 변경 필요
  # https 사용하는 경우, skipTLSVerify 설정 가능
  # skipTLSVerify: true
EOF

# 1-1. VaultConnection 적용
kubectl apply -f vaultconnection.yaml
vaultconnection.secrets.hashicorp.com/vault-connection created

# 1-2. vaultconnection 생성 확인
kubectl get -n app vaultconnection vault-connection
NAME               AGE
vault-connection   39s

# 1-3. vaultconnection : Events Message 확인
kubectl describe -n app vaultconnection vault-connection
Name:         vault-connection
Namespace:    app
Labels:       <none>
Annotations:  <none>
API Version:  secrets.hashicorp.com/v1beta1
Kind:         VaultConnection
Metadata:
  Creation Timestamp:  2024-01-16T08:48:51Z
...(생략)
Spec:
  Address:          <http://47.128.149.217:8200>
  Skip TLS Verify:  false
Status:
  Valid:  true
Events:
  Type    Reason    Age   From             Message
  ----    ------    ----  ----             -------
  Normal  Accepted  59s   VaultConnection  VaultConnection accepted

```

```yaml
# 2. VaultAuth Yaml 작성
# Vault 인증 정보 지정
tee vaultauth.yaml <<EOF
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultAuth
metadata:
  namespace: app
  name: vaultauth
spec:
  vaultConnectionRef: vault-connection	# 위에서 생성한 VaultConnection 이름
  method: kubernetes	# Vault auth mehtod 타입 지정
  mount: kubernetes		# Vault auth mehtod 활성화한 path 지정
  kubernetes:
    role: vsoperator	# Vault Kubernetes Auth의 Role 이름 지정
    serviceAccount: vso-account	# Kubernetes Role에 연결된 Service account 지정 
  namespace: "shinhan"	# Vault Kubernetes Auth 활성화된 namespace 지정
EOF

# 2-1. VaultAuth 적용
kubectl apply -f vaultauth.yaml
vaultauth.secrets.hashicorp.com/vaultauth created

# 2-2. VaultAuth 생성 확인
kubectl get -n app vaultauth vaultauth
NAME        AGE
vaultauth   60s

# 2-3. VaultAuth : Events Message 확인
kubectl describe -n app vaultauth vaultauth
Name:         vaultauth
Namespace:    app
Labels:       <none>
Annotations:  <none>
API Version:  secrets.hashicorp.com/v1beta1
Kind:         VaultAuth
Metadata:
  Creation Timestamp:  2024-01-16T09:01:15Z
...(생략)
Spec:
  Kubernetes:
    Role:                      vsoperator
    Service Account:           vso-account
    Token Expiration Seconds:  600
  Method:                      kubernetes
  Mount:                       kubernetes
  Namespace:                   shinhan
  Vault Connection Ref:        vault-connection
Status:
  Error:
  Valid:  true
Events:
  Type    Reason    Age   From       Message
  ----    ------    ----  ----       -------
  Normal  Accepted  71s   VaultAuth  Successfully handled VaultAuth resource request

```

```yaml
# 3. VaultStaticSecret Yaml 작성
# Vault에서 읽어올 KV-v2 Secret 경로 및 refresh 주기 설정 
tee vault-static-secret-kv2.yaml <<EOF
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  namespace: app
  name: vault-static-kv2-secret
spec:
  vaultAuthRef: vaultauth
  namespace: shinhan
  mount: kvv2
  type: kv-v2
  path: myapp/config
  refreshAfter: 10s
  # k8s secret 생성 값 지정
  destination:
    create : true
    name: staticsecret-kv2
EOF

# 3-1. VaultStaticSecret 적용
kubectl apply -f vault-static-secret-kv2.yaml
vaultstaticsecret.secrets.hashicorp.com/vault-static-kv2-secret created

# 3-2. VaultStaticSecret 생성 확인
kubectl get -n app VaultStaticSecret vault-static-kv2-secret
NAME                      AGE
vault-static-kv2-secret   49s

# 3-3. VaultAuth Events Message 확인
kubectl describe -n app VaultStaticSecret vault-static-kv2-secret
Name:         vault-static-kv2-secret
Namespace:    app
Labels:       <none>
Annotations:  <none>
API Version:  secrets.hashicorp.com/v1beta1
Kind:         VaultStaticSecret
Metadata:
  Creation Timestamp:  2024-01-16T09:17:59Z
...(생략)
Spec:
  Destination:
    Create:          true
    Name:            staticsecret-kv2
  Hmac Secret Data:  true
  Mount:             kvv2
  Namespace:         shinhan
  Path:              myapp/config
  Refresh After:     10s
  Type:              kv-v2
  Vault Auth Ref:    vaultauth
Status:
  Last Generation:  1
  Secret MAC:       fXffFh4GhyaOASMfJLvHzM560/6AXWEYOJPHZLyyeEg=
Events:
  Type    Reason         Age       From               Message
  ----    ------         ----      ----               -------
  Normal  SecretSynced   1m44s     VaultStaticSecret  Secret synced

```

```bash
# 4. 생성된 k8s secret 값 확인
# 4-1. _raw 값 확인
kubectl get secrets -n app staticsecret-kv2 -o jsonpath="{.data._raw}" | base64 -d; echo
{"data":{"password":"suP3rsec(et","username":"appuser"},"metadata":{"created_time":"2024-01-16T07:42:48.4834714Z","custom_metadata":null,"deletion_time":"","destroyed":false,"version":1}}

# 4-2. password 값 확인
kubectl get secrets -n app staticsecret-kv2 -o jsonpath="{.data.password}" | base64 -d; echo
suP3rsec(et

# 4-3. username 값 확인
kubectl get secrets -n app staticsecret-kv2 -o jsonpath="{.data.username}" | base64 -d; echo
appuser

```



## 7. **Vault 작업 : KV 값 변경**

```bash
# kvv2/myapp/config 값 변경
vault kv put -ns=shinhan kvv2/myapp/config \
username="appuser-v2" \
password="suP3rsec(et-v2"

```



## 8. **Kubernetes : Secret 값 변경 확인**

```bash
# 1-1. _raw 값 확인
kubectl get secrets -n app staticsecret-kv2 -o jsonpath="{.data._raw}" | base64 -d; echo
{"data":{"password":"suP3rsec(et-v2","username":"appuser-v2"},"metadata":{"created_time":"2024-01-16T09:19:20.890456382Z","custom_metadata":null,"deletion_time":"","destroyed":false,"version":2}}

# 1-2. password 값 확인
kubectl get secrets -n app staticsecret-kv2 -o jsonpath="{.data.password}" | base64 -d; echo
suP3rsec(et-v2

# 1-3. username 값 확인
kubectl get secrets -n app staticsecret-kv2 -o jsonpath="{.data.username}" | base64 -d; echo
appuser-v2

# 1-4. VaultAuth : Events Message (SecretRotated, SecretSync) 확인
kubectl describe -n app VaultStaticSecret vault-static-kv2-secret
Name:         vault-static-kv2-secret
Namespace:    app
Labels:       <none>
Annotations:  <none>
API Version:  secrets.hashicorp.com/v1beta1
Kind:         VaultStaticSecret
Metadata:
  Creation Timestamp:  2024-01-16T09:17:59Z
...(생략)
Spec:
  Destination:
    Create:          true
    Name:            staticsecret-kv2
  Hmac Secret Data:  true
  Mount:             kvv2
  Namespace:         shinhan
  Path:              myapp/config
  Refresh After:     10s
  Type:              kv-v2
  Vault Auth Ref:    vaultauth
Status:
  Last Generation:  1
  Secret MAC:       fXffFh4GhyaOASMfJLvHzM560/6AXWEYOJPHZLyyeEg=
Events:
  Type    Reason         Age                   From               Message
  ----    ------         ----                  ----               -------
  Normal  SecretSynced   3m44s                 VaultStaticSecret  Secret synced
  Normal  SecretRotated  2m19s                 VaultStaticSecret  Secret synced
  Normal  SecretSync     19s (x23 over 3m35s)  VaultStaticSecret  Secret sync not required

```
