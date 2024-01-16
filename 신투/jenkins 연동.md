# [PoC] Jenkins - Vault 연동 (KV)

# Vault Server

### KV Engine 설정

```bash
# namespace 생성
$ vault namespace create jenkins
Key                Value
---                -----
custom_metadata    map[]
id                 dACKF
path               jenkins/

# KV secret 활성화
$ vault secrets enable -path=kv -ns=jenkins -version=2 kv
Success! Enabled the kv secrets engine at: kv/

# KV 데이터 생성
$ vault kv put -ns=jenkins kv/secret password=1234
= Secret Path =
kv/data/secret

======= Metadata =======
Key                Value
---                -----
created_time       2023-10-26T01:02:38.725834966Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1

# KV read policy 생성
$ vault policy write -ns=jenkins kv_jenkins_policy -<<EOF
path "kv/data/secret" {
  capabilities = ["read"]
}
EOF
Success! Uploaded policy: kv_jenkins_policy
```

### Entity 생성 및 Token 발급

최소한의 Vault 권한을 부여한 Vault Token을 발급 받고, Jenkins Credentials에 등록하여 사용.

Jenkins Credentials를 추가 등록 시 Entity를 추가 생성하여 Token 발급

```bash
# Entity 생성
$ vault write -ns=jenkins identity/entity name=app_jenkins policies=kv_jenkins_policy
Key        Value
---        -----
aliases    <nil>
id         c75fe501-63e4-97ae-21b1-6cab0d67633a
name       app_jenkins

# token auth accessor 조회
$ vault auth list -ns=jenkins
Path      Type        Accessor                  Description                Version
----      ----        --------                  -----------                -------
token/    ns_token    auth_ns_token_0633be57    token based credentials    n/a

# Entity Alias 생성
$ vault write -ns=jenkins identity/entity-alias name=app_jenkins \
canonical_id=c75fe501-63e4-97ae-21b1-6cab0d67633a \
mount_accessor=auth_ns_token_0633be57
Key             Value
---             -----
canonical_id    c75fe501-63e4-97ae-21b1-6cab0d67633a
id              141627ba-fa64-9307-c9b5-45703faffc77

# Token role 생성
$ vault write -ns=jenkins auth/token/roles/app_jenkins_token_role \
allowed_entity_aliases=app_jenkins \
orphan=true \
token_period=10d \
token_bound_cidrs=35.183.111.12/32
Success! Data written to: auth/token/roles/app_jenkins_token_role

# Token 발급 - 발급된 토큰은 Jenkins Credential에 등록하여 사용
$ vault token create -ns=jenkins \
-role=app_jenkins_token_role \
-entity-alias=app_jenkins \
-display-name=app_jenkins \
-no-default-policy=true
Key                  Value
---                  -----
token                hvs.CAESIFcyOIYN-SAEOpRwANNTZrmmoxxap8J5STNMgN6K5NA6GicKImh2cy5uU1pMeFJHTnl3SlAwVDZjZHZQQnVyYmwuaVJYaUwQpQU
token_accessor       XoTxJzXkUOicMrVIL0vVfejg.iRXiL
token_duration       240h
token_renewable      true
token_policies       []
identity_policies    []
policies             []

# Token 확인
$ vault token lookup hvs.CAESIFcyOIYN-SAEOpRwANNTZrmmoxxap8J5STNMgN6K5NA6GicKImh2cy5uU1pMeFJHTnl3SlAwVDZjZHZQQnVyYmwuaVJYaUwQpQU
Key                            Value
---                            -----
accessor                       ZmHxmv7U9d7Zb1IZHp8p70Ac.POAFz
bound_cidrs                    [35.183.111.12]
creation_time                  1705396061
creation_ttl                   240h
display_name                   token-app-jenkins
entity_id                      18448201-ca44-d198-e4c2-782caa97e7b9
expire_time                    2024-01-26T09:07:41.68171967Z
explicit_max_ttl               0s
external_namespace_policies    map[]
id                             hvs.CAESIOKiEpC6XKzLs2g15qzW6kltyjhi-LhJ_KHpBor2z4-uGigKImh2cy5yWTMxZ0RQVzloODhyaEdrTExTRmlLbnkuUE9BRnoQl-UB
identity_policies              [kv_jenkins_policy]
issue_time                     2024-01-16T09:07:41.681726816Z
meta                           <nil>
namespace_path                 jenkins/
num_uses                       0
orphan                         true
path                           auth/token/create/app_jenkins_token_role
policies                       []
renewable                      true
role                           app_jenkins_token_role
ttl                            239h59m51s
type                           service
```

# Jenkins Server

## Vault Plugin 설치

jenkins 관리 - System Configuration - Plugins

![Untitled](https://github.com/jslim1995/insideinfo-vault/assets/100335118/63e2a544-a911-442f-87dc-cb55b977abd6)

### Vault Plugin 설치

Available plugins 탭에서 HashiCorp vault 설치 후 Jenkins 재기동

![Untitled 1](https://github.com/jslim1995/insideinfo-vault/assets/100335118/4c0fd4f6-d79b-460f-8324-049d548b59f0)

## Credentials 추가

Jenkins Credentials는 Jenkins Pipeline에서 Vault Token을 사용하기 위해 구성

Vault Token에 최소 권한을 부여하기 위해 Jenkins Pipeline별로 다른 Vault Token이 적용된 Jenkins Credentials 사용 권장

### Credentials 목록

Jenkins 관리 - Security - Credentials

![Untitled 2](https://github.com/jslim1995/insideinfo-vault/assets/100335118/cdbcf485-5054-47ee-a35c-2a5cb1021d25)

### add Credentials

![Untitled 3](https://github.com/jslim1995/insideinfo-vault/assets/100335118/a4b91694-1034-4300-a765-b13458b020ef)

![Untitled 4](https://github.com/jslim1995/insideinfo-vault/assets/100335118/c71063f1-4b87-4ebe-9760-693eed2c70ab)

> Kind : `Vault Token Credential`
>
> Token : Pipeline에서 사용 할 최소의 권한이 부여된 Vault Token
>
> Namespace : Vault Namespace 명 (현재 Jenkins 측 이슈로 적용 불가)
>
> ID : `vault-jenkins-app_jenkins-token` (Jenkins에서 사용하는 Credential 구분 ID) - Vault Token관리를 위해 네이밍 규칙 필요 ex) `vault-<NAMESPACE>-<ENTITY_NAME>-token`

### 생성된 Credentials

생성된 ID(`vault-jenkins-app_jenkins-token`) 확인

해당 Credential ID는 Pipeline에서 호출하여 사용

![Untitled 5](https://github.com/jslim1995/insideinfo-vault/assets/100335118/3963536a-41ae-44fc-a7fe-164b0ad91a47)

## System 설정

jenkins 관리 - System Configuration - System

![Untitled 6](https://github.com/jslim1995/insideinfo-vault/assets/100335118/5bc6a6c8-ec20-4b9a-8762-0f9a5f758c9b)

### Vault Plugin Global 설정

Vault Plugin Global 설정 값 미 지정 시 Script에서 변수 지정 필요

Vault Credential, Vault Namespace는 Pipeline 별로 다른 값을 사용할 수 있도록 미 지정

![Untitled 7](https://github.com/jslim1995/insideinfo-vault/assets/100335118/41eaed5d-9fb1-4358-9c3e-ed1d0801eb8f)

> Vault URL : `http://15.222.62.110:8200`
>
> Vault Credential : 미 지정 (Pipeline - Script의 configuration에서 구성)
>
> Vault Namespace : 미 지정 (Pipeline - Script의 configuration에서 구성)
>
> K/V Engine Version : 2 (기본 값)

설정 후 저장

## Pipline 생성

새로운 Item 클릭

![Untitled 8](https://github.com/jslim1995/insideinfo-vault/assets/100335118/fedc8bc4-6262-4cd1-b3e9-4383281499a4)

### Pipeline 생성

item name 지정 후 Pipeline 선택

![Untitled 9](https://github.com/jslim1995/insideinfo-vault/assets/100335118/a42da33e-3d8a-444e-9bc0-1a5118530512)

### Script 작성 (Vault Plugin 사용)

사용 예시 : [https://plugins.jenkins.io/hashicorp-vault-plugin/#plugin-content-usage-via-jenkinsfile](https://plugins.jenkins.io/hashicorp-vault-plugin/#plugin-content-usage-via-jenkinsfile)

```groovy
// optional configuration, if you do not provide this the next higher configuration
// (e.g. folder or global) will be used
def configuration = [vaultUrl: 'http://my-very-other-vault-url.com',
                     vaultCredentialId: 'my-vault-cred-id',
                     engineVersion: 1]

// define the secrets and the env variables
// engine version can be defined on secret, job, folder or global.
// the default is engine version 2 unless otherwise specified globally.
def secrets = [
    [path: 'secret/testing', engineVersion: 1, secretValues: [
        [envVar: 'testing', vaultKey: 'value_one'],
        [envVar: 'testing_again', vaultKey: 'value_two']]],
    [path: 'secret/another_test', engineVersion: 2, secretValues: [
        [vaultKey: 'another_test']]]
]

// inside this block your credentials will be available as env variables
withVault([configuration: configuration, vaultSecrets: secrets]) {
    sh 'echo $testing'
    sh 'echo $testing_again'
    sh 'echo $another_test'
}
```

사용 가능한 변수 목록 : [https://www.jenkins.io/doc/pipeline/steps/hashicorp-vault-plugin/](https://www.jenkins.io/doc/pipeline/steps/hashicorp-vault-plugin)

- `configuration` (옵션)
  → `configuration`의 각 변수는 Jenkins에서 설정한 Vault Plugin global 설정 값보다 높은 우선순위를 가짐
  - `engineVersion : int` - KV 엔진 버전 지정 (미 지정 시 Vault Plugin Global 설정 값 사용)
  - `vaultCredentialId : String` - Jenkins Creds ID 지정 (미 지정 시 Vault Plugin Global 설정 값 사용)
  - `vaultNamespace : String` - Vault Namespace 지정 (미 지정 시 Vault Plugin Global 설정 값 사용)
  - `vaultUrl : String` - Vault 서버 URL 지정 (미 지정 시 Vault Plugin Global 설정 값 사용)
- `vaultSecrets` (필수)
  - `path : String` - Vault KV 경로 지정
  - `secretValues`
    - `vaultKey : String` - Vault KV Key 값 지정
    - `envVar : String` - withVault 함수 내에서 사용할 환경 변수 명 지정
  - `engineVersion : int` - KV 엔진 버전 지정 (미 지정 시 configuration 설정 값 사용)

### Jenkins Pipeline에 적용

```groovy
pipeline {
    agent any
    stages {
        stage('Vault') {
            steps {
                script {
                    // withVault 함수에 필요한 변수 설정
                    def configuration = [
                        vaultCredentialId: 'vault-jenkins-app_jenkins-token',
                        vaultNamespace: 'jenkins'
                    ]
                    def secrets = [
                        [path: 'kv/secret', secretValues: [
                            [envVar: 'PASSWORD_ENV', vaultKey: 'password']]]
                    ]

                    withVault([configuration: configuration, vaultSecrets: secrets]) {
                        sh '''
                            echo $PASSWORD_ENV | tee /tmp/kv-read-result.txt
                        '''
                    }
                }
            }
        }
    }
}
```

### (Option) Pipeline Syntax를 통한 Script 작성

1. Script 아래에 있는 `Pipeline Syntax` 선택

![Untitled 10](https://github.com/jslim1995/insideinfo-vault/assets/100335118/37b02329-29f3-46e8-9263-af64df95ffe5)

1. Sample Step 설정

   `withVault: Vault Plugin` 선택

   Vault Plugin

   - Vault URL : Vault Plugin Global 설정 값 사용
   - Vault Credential : 이전 단계에서 설정한 Jenkins Credential 지정

![Untitled 11](https://github.com/jslim1995/insideinfo-vault/assets/100335118/f7624bde-5cfd-4f3e-a531-fb847fb8d9dc)

1. Advanced 설정

   `Advanced` 클릭 후 Vault Namespace : `jenkins` 지정

![Untitled 12](https://github.com/jslim1995/insideinfo-vault/assets/100335118/274c38ad-c489-45ae-8450-d56bc7ea2836)

1. Vault Secret 설정

   `Vault Secret Path`, `Evironment Variable` 및 `Key name` 지정

![Untitled 13](https://github.com/jslim1995/insideinfo-vault/assets/100335118/c3d0ec2e-2f97-4aff-82d0-b6725051eccc)

1. 하단의 `Generate Pipeline Script` 클릭하여 withVault 함수 값 확인

![Untitled 14](https://github.com/jslim1995/insideinfo-vault/assets/100335118/7836d952-b9cd-4dfb-8a53-0be64dce0e8e)

1. Jenkins Pipeline에 적용

```bash
pipeline {
    agent any
    stages {
        stage('Vault') {
            steps {
                withVault(configuration: [disableChildPoliciesOverride: false, timeout: 60, vaultCredentialId: 'vault-kv-token', vaultNamespace: 'jenkins'], vaultSecrets: [[path: 'kv/secret', secretValues: [[envVar: 'PASSWORD_ENV', vaultKey: 'password']]]]) {
                    // some block
                    sh '''
                        echo $PASSWORD_ENV | tee /tmp/kv-read-result.txt
                    '''
                }
            }
        }
    }
}
```

## Build

생성한 pipline을 빌드 실행

![Untitled 15](https://github.com/jslim1995/insideinfo-vault/assets/100335118/5407a918-67d0-4282-8396-8978d2b476a2)

### 빌드 결과 확인

Logs를 클릭하여 Stage Logs 확인

![Untitled 16](https://github.com/jslim1995/insideinfo-vault/assets/100335118/d4c23331-49ab-4dff-91da-b8c3f5d45b0b)

jenkins 서버에 생성된 `/tmp/kv-read-result.txt` 파일 확인

![Untitled 17](https://github.com/jslim1995/insideinfo-vault/assets/100335118/52d2c8e6-ab34-4af3-b041-a9ab54240f80)
