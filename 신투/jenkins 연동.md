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

# ssh otp policy 생성
$ vault policy write -ns=jenkins kv_jenkins_policy -<<EOF
path "kv/data/secret" {
  capabilities = ["read"]
}
EOF
Success! Uploaded policy: kv_jenkins_policy
```

### Entity 생성 및 Token 발급

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

# Token 발급
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

![Untitled](https://github.com/jslim1995/insideinfo-vault/assets/100335118/13a81cd6-d872-45f7-8540-3ee24bb845ba)

### Vault Plugin 설치

Available plugins 탭에서 HashiCorp vault 설치 후 Jenkins 재기동

![Untitled 1](https://github.com/jslim1995/insideinfo-vault/assets/100335118/908f0ae2-afcf-4baa-a551-43f708577bbe)

## Credentials 추가

### Credentials 목록

Jenkins 관리 - Security - Credentials

![Untitled 2](https://github.com/jslim1995/insideinfo-vault/assets/100335118/08cc7892-08f9-4191-8424-cfe6e1b75a43)

### add Credentials

![Untitled 3](https://github.com/jslim1995/insideinfo-vault/assets/100335118/0a2d866c-7380-4f20-8213-7c866100c9fb)

![Untitled 4](https://github.com/jslim1995/insideinfo-vault/assets/100335118/b0e0d386-1094-452e-b943-f853ef5d858f)

> Kind : `Vault Token Credential`
>
> Token : kv 조회 권한이 있는 Vault Token
>
> Namespace : Vault Namespace 명 (현재 Jenkins 측 이슈로 적용 불가)
>
> ID : `vault-jenkins-app_jenkins-token` (Jenkins에서 사용하는 Credential 구분 ID) - Vault Token관리를 위해 네이밍 규칙 필요 ‘`vault-<NAMESPACE>-<ENTITY_NAME>-token`’ 

### 생성된 Credentials

생성된 ID(`vault-jenkins-app_jenkins-token`) 확인

![Untitled 5](https://github.com/jslim1995/insideinfo-vault/assets/100335118/87d47dff-d56f-4601-9318-dc6a6b088b17)

## System 설정

jenkins 관리 - System Configuration - System

![Untitled 6](https://github.com/jslim1995/insideinfo-vault/assets/100335118/2c0c597e-a1a6-4776-8373-af4b5727bae3)

### Vault Plugin Global 설정

![Untitled 7](https://github.com/jslim1995/insideinfo-vault/assets/100335118/6e6f5274-808c-4de3-894a-4e5972a04b75)

Vault URL : `http://15.222.62.110:8200`

설정 후 저장

## Pipline 생성

새로운 Item 클릭

![Untitled 8](https://github.com/jslim1995/insideinfo-vault/assets/100335118/71752f59-3c61-4184-8937-836a9a878dd1)

### Pipeline 생성

item name 지정 후 Pipeline 선택

![Untitled 9](https://github.com/jslim1995/insideinfo-vault/assets/100335118/87043a45-0634-4d07-810e-7d3d902f40eb)

### Script 작성 (Vault Plugin 사용)

사용 예시 : [https://plugins.jenkins.io/hashicorp-vault-plugin/#plugin-content-usage-via-jenkinsfile](https://plugins.jenkins.io/hashicorp-vault-plugin/#plugin-content-usage-via-jenkinsfile)

사용 가능한 변수 목록 : [https://www.jenkins.io/doc/pipeline/steps/hashicorp-vault-plugin/](https://www.jenkins.io/doc/pipeline/steps/hashicorp-vault-plugin)

- `configuration` (optional)
  → `configuration`의 각 변수는 Jenkins에서 설정한 Vault Plugin global 설정 값보다 높은 우선순위를 가짐
  - `engineVersion : int` - KV 엔진 버전 지정 (미 지정 시 Vault Plugin Global 설정 값 사용)
  - `vaultCredentialId : String` - Jenkins Creds ID 지정 (미 지정 시 Vault Plugin Global 설정 값 사용)
  - `vaultNamespace : String` - Vault Namespace 지정 (미 지정 시 Vault Plugin Global 설정 값 사용)
  - `vaultUrl : String` - Vault 서버 URL 지정 (미 지정 시 Vault Plugin Global 설정 값 사용)
- `vaultSecrets`
  - `path : String` - Vault KV 경로 지정
  - `secretValues`
    - `vaultKey : String` - Vault KV Key 값 지정
    - `envVar : String` - withVault 함수 내에서 사용할 환경 변수 명 지정
  - `engineVersion : int` - KV 엔진 버전 지정 (미 지정 시 configuration 설정 값 사용)

```bash
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
                        [
                            path: 'kv/secret',
                            secretValues: [
                                [envVar: 'PASSWORD_ENV', vaultKey: 'password']
                            ]
                        ]
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

![Untitled 10](https://github.com/jslim1995/insideinfo-vault/assets/100335118/bc94ec72-dbed-41b1-a567-64c97ab19640)

2. Sample Step 설정

`withVault: Vault Plugin` 선택 후 Vault Plugin - Vault Credential 설정

![Untitled 11](https://github.com/jslim1995/insideinfo-vault/assets/100335118/5fe0fddf-8b44-4d60-a7ed-a685055e67ea)

3. Advanced 설정

 `Advanced` 클릭 후 Vault Namespace : `jenkins` 지정

![Untitled 12](https://github.com/jslim1995/insideinfo-vault/assets/100335118/e781f6ba-99fe-4a82-addb-63a4937f6a27)

4. Vault Secret 설정

`Vault Secret Path`, `Evironment Variable` 및 `Key name` 지정

![Untitled 13](https://github.com/jslim1995/insideinfo-vault/assets/100335118/a9c9b6e1-8483-455a-9834-57480a4de3a8)

5. 하단의 `Generate Pipeline Script` 클릭하여 withVault 함수 값 확인

![Untitled 14](https://github.com/jslim1995/insideinfo-vault/assets/100335118/6ff02e70-54fc-4e9d-8e9e-317618628f2b)

6. Jenkins Pipeline에 적용

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

![Untitled 15](https://github.com/jslim1995/insideinfo-vault/assets/100335118/1d1cd4e8-12ec-41dc-a527-c4e5fc524af0)

### 빌드 결과 확인

Logs를 클릭하여 Stage Logs 확인

![Untitled 16](https://github.com/jslim1995/insideinfo-vault/assets/100335118/dc18458b-8420-4e07-9950-5e1bf16ca4db)

jenkins 서버에 생성된 `/tmp/kv-read-result.txt` 파일 확인

![Untitled 17](https://github.com/jslim1995/insideinfo-vault/assets/100335118/7831bfec-e5c6-47f2-8349-ab872736bac1)
