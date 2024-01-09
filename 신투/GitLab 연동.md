# Vault 연동 (KV)

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
allowed_entity_aliases=app_jenkins orphan=true token_period=10d token_bound_cidrs=43.201.98.46/32
Success! Data written to: auth/token/roles/app_jenkins_token_role

# Token 발급
$ vault token create -ns=jenkins \
-role=app_jenkins_token_role \
-entity-alias=app_jenkins \
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
accessor                       XoTxJzXkUOicMrVIL0vVfejg.iRXiL
creation_time                  1698284357
creation_ttl                   240h
display_name                   token
entity_id                      c75fe501-63e4-97ae-21b1-6cab0d67633a
expire_time                    2023-11-05T01:39:17.929679193Z
explicit_max_ttl               0s
external_namespace_policies    map[]
id                             hvs.CAESIFcyOIYN-SAEOpRwANNTZrmmoxxap8J5STNMgN6K5NA6GicKImh2cy5uU1pMeFJHTnl3SlAwVDZjZHZQQnVyYmwuaVJYaUwQpQU
identity_policies              [kv_jenkins_policy]
issue_time                     2023-10-26T01:39:17.929688972Z
meta                           <nil>
namespace_path                 jenkins/
num_uses                       0
orphan                         true
path                           auth/token/create/app_jenkins_token_role
policies                       []
renewable                      true
role                           app_jenkins_token_role
ttl                            239h58m25s
type                           service
```

# Jenkins Server

## Vault Plugin 설치

jenkins 관리 - System Configuration - Plugins

![Untitled](https://github.com/jslim1995/insideinfo-vault/assets/100335118/25cbb917-18a9-4a66-a30d-ebff01e9c586)

### Vault Plugin 설치

Available plugins 탭에서 HashiCorp vault 설치 후 Jenkins 재기동

![Untitled 1](https://github.com/jslim1995/insideinfo-vault/assets/100335118/3487ec34-0d3d-4a14-a9d6-1c53e8e69dbf)

## Credentials 추가

### Credentials 목록

Jenkins 관리 - Security - Credentials

![Untitled 2](https://github.com/jslim1995/insideinfo-vault/assets/100335118/d731a54a-e7e1-4a25-8998-b36703fa8aaa)

### add Credentials

![Untitled 3](https://github.com/jslim1995/insideinfo-vault/assets/100335118/4daacb26-3084-45da-b802-d4e7a7182b79)

![Untitled 4](https://github.com/jslim1995/insideinfo-vault/assets/100335118/a8314163-dd75-45cc-88fa-bbc8e9e879a8)

> Kind : Vault Token Credential
>
> Token : kv 조회 권한이 있는 Token
>
> Namespace : Vault Namespace 명 (적용 안됨)
>
> ID : Jenkins에서 사용하는 Credential 구분ID

### 생성된 Credentials

![Untitled 5](https://github.com/jslim1995/insideinfo-vault/assets/100335118/99b99836-5ed2-400f-8ce1-32fe8a64da02)

## System 설정

jenkins 관리 - System Configuration - System

![Untitled 6](https://github.com/jslim1995/insideinfo-vault/assets/100335118/44eefca3-3a69-4ae2-9f94-d6bdf0947b97)

### Vault Plugin 설정

![Untitled 7](https://github.com/jslim1995/insideinfo-vault/assets/100335118/99c5bad5-3027-48c6-be2b-56a5062b217c)

Vault URL : `http://172.32.0.75:8200`
Vault Credential : `jenkins vault token for kv read`
Vault Namespace : `jenkins`

설정 후 저장

## Pipline 생성

새로운 Item 클릭

![Untitled 8](https://github.com/jslim1995/insideinfo-vault/assets/100335118/eda8c626-4d0b-4d2f-9f3b-2564b04ec53e)

### Pipeline 생성

item name 지정 후 Pipeline 선택

![Untitled 9](https://github.com/jslim1995/insideinfo-vault/assets/100335118/01152968-2f9f-403b-9c57-ca8cf41114e5)

Pipeline Script 작성은 아래 3가지 방법 중 하나 선택

### Script 생성 (Vault Pipeline Plugin ****global 사용****)

[Hashicorp Vault Pipeline](https://plugins.jenkins.io/hashicorp-vault-pipeline/)

```bash
pipeline {
    agent any
    environment {
        PASSWORD_ENV = vault path: 'kv/secret', key: 'password'
    }
    stages {
        stage("read vault key") {
            steps {
                sh '''
                    echo $PASSWORD_ENV | tee /tmp/kv-read-result.txt
                '''
            }
        }
    }
}
```

### Script 생성 (Vault Pipeline Plugin ****pipeline specific 사용****)

```bash
pipeline {
    agent any
    environment {
        PASSWORD_ENV = vault path: 'kv/secret', key: 'password', vaultUrl: 'http://172.32.0.75:8200', credentialsId: 'jenkins-vault-kv-token', engineVersion: "2"
    }
    stages {
        stage("read vault key") {
            steps {
                sh '''
                    echo $PASSWORD_ENV | tee /tmp/kv-read-result.txt
                '''
            }
        }
    }
}
```

### Script 생성 (Vault Plugin 사용)

1. Script 아래에 있는 `Pipeline Syntax` 선택

![Untitled 10](https://github.com/jslim1995/insideinfo-vault/assets/100335118/5b141763-d60c-4dc1-b4dd-8ce01370683f)

2. Sample Step : `withVault: Vault Plugin` 선택 및 Vault Plugin 설정

![Untitled 11](https://github.com/jslim1995/insideinfo-vault/assets/100335118/34e82ec9-7a72-4343-9aff-3c8bc378b010)

3. Advanced 클릭 후 Vault Namespace : `jenkins` 지정

![Untitled 12](https://github.com/jslim1995/insideinfo-vault/assets/100335118/e79cd14c-8bc5-4812-982c-f709a2eb6ef4)

4. `Add a vault secret` 클릭 후 `Vault Secret Path` 지정 및 `Add a Key/value pair` 클릭 후 `Evironment Variable` 및 `Key name` 지정

![Untitled 13](https://github.com/jslim1995/insideinfo-vault/assets/100335118/194c9474-6586-4c66-91cc-b87bcf989b34)

5. 하단의 `Generate Pipeline Script` 클릭하여 withVault 함수 값 확인

![Untitled 14](https://github.com/jslim1995/insideinfo-vault/assets/100335118/6d0cd3ff-128f-4ada-a31a-4eea9164d1cf)

```bash
pipeline {
    agent any
    stages {
        stage('Vault') {
            steps {
                withVault(configuration: [disableChildPoliciesOverride: false, engineVersion: 2, timeout: 60, vaultCredentialId: 'jenkins-vault-kv-token', vaultNamespace: 'jenkins', vaultUrl: 'http://172.32.0.75:8200'], vaultSecrets: [[engineVersion: 2, path: 'kv/secret', secretValues: [[envVar: 'PASSWORD_ENV', vaultKey: 'password']]]]) {
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

![Untitled 15](https://github.com/jslim1995/insideinfo-vault/assets/100335118/268827a3-6865-4308-9d55-cc766b1469bf)

### 빌드 결과 확인

Logs를 클릭하여 Stage Logs 확인

![Untitled 16](https://github.com/jslim1995/insideinfo-vault/assets/100335118/a309b6c3-43bb-4240-828c-5abde87cbb16)

jenkins 서버에 생성된 `/tmp/kv-read-result.txt` 파일 확인

![Untitled 17](https://github.com/jslim1995/insideinfo-vault/assets/100335118/3789c88f-d2b6-499b-9cfa-8292dc962234)
