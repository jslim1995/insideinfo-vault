# Gitlab - Vault(JWT) 연동

# Vault Server

참고 : [https://gitlab.com/gitlab-com/support/support-training/-/blob/master/Support Specific Trainings/Vault.md#configuring-jwt-authentication](https://gitlab.com/gitlab-com/support/support-training/-/blob/master/Support%20Specific%20Trainings/Vault.md#configuring-jwt-authentication)

```bash
# namespace 활성화
vault namespace create gitlab_demo

# KV Engine 활성화
vault secrets enable -ns=gitlab_demo -path=kv kv-v2

# KV data 생성
vault kv put -ns=gitlab_demo kv/secret password=passwordForGitlab

# KV data 조회 정책 생성
vault policy write -ns=gitlab_demo kv_gitlab_policy - <<EOF
path "kv/data/secret" {
  capabilities = [ "read" ]
}  
EOF

# jwt auth 활성화
vault auth enable -ns=gitlab_demo jwt

# auth check
vault auth list -ns=gitlab_demo 

# jwt config 설정
# jwks_url, oidc_discovery_url 옵션 중 하나만 설정 가능
# jwks_url : gitlab의 jwks 주소
# oidc_discovery_url : GitLab의 주소(GitLab 서버가 HTTPS로 통신해야 정상적으로 인증 가능)
# bound_issuer : gitlab runner에 등록된 GitLab instance URL
vault write -ns=gitlab_demo auth/jwt/config \
jwks_url="http://172.32.3.64:8100/-/jwks" \
# oidc_discovery_url="http://172.32.3.64:8100" \
bound_issuer="http://172.32.3.64:8100"

# jwt role 생성
# kv data 조회 정책을 보유한 jwt gitlab role 생성
vault write -ns=gitlab_demo auth/jwt/role/gitlab_role - <<EOF
{
  "role_type": "jwt",
  "policies": ["kv_gitlab_policy"],
  "token_explicit_max_ttl": 60,
  "user_claim": "project_path",
  "bound_claims_type": "string",
  "bound_claims": {
    "project_id": "4",
    "ref": "main",
    "ref_type": "branch"
  }
}
EOF
```

### jwt gitlab role에 적용될 프로젝트

Project ID : `4`

![Untitled](https://github.com/jslim1995/insideinfo-vault/assets/100335118/b84f68e8-b11d-43c9-b1b6-fe749d9693c5)

### claim 목록

[Authenticating and reading secrets with HashiCorp Vault | GitLab](https://docs.gitlab.com/ee/ci/examples/authenticating-with-hashicorp-vault/)

# GitLab

서버 주소 : `http://3.99.145.122:8100`

접속 정보 : `root` / `insideinfo`

Project 경로 : `vault_team/jwt_test`

## CI/CD 환경 변수 설정

Project → Settings → CI/CD → Variables 에서 환경 변수 추가

VAULT_ADDR : `http://15.222.62.110:8200`

![Untitled 1](https://github.com/jslim1995/insideinfo-vault/assets/100335118/d226d895-d2db-407b-bf0b-e24b4c267a0d)

## pipeline 구성 및 빌드

GitLab Project → Build → Pipeline Editor 에서 Pipeline 구성

GitLab Pipeline에서 `JWT`를 발급 받고 `script` 블록에서 `Vault command`혹은 `Vault API`를 사용하여 `Vault JWT Auth`로 로그인하여 `Vault Token`을 발급 받은 후 `Vault KV` 값을 가져오거나, GitLab에서 제공하는 `secrets:vault`를 통해 Vault KV 값을 가져올 수 있다.

단, `secrets:vault`는 GitLab License 필요([https://docs.gitlab.com/ee/ci/yaml/index.html#secrets](https://docs.gitlab.com/ee/ci/yaml/index.html#secrets))

### 1. Vault Command 사용

Pipeline에서 `JWT`를 발급 받아 `script` 블록에서 `Vault Command`를 사용하여 Vault 토큰 발급 및 `Vault KV` 값 조회

사용 조건 (택 1)

1. GitLab-Runner가 `executor : docker` 로 지정되어 있으며, `hashicorp/vault` 이미지를 호출할 수 있는 상태
2. GitLab-Runner가 `excutor : shell`로 지정되어 있으며, GitLab-Runner서버에 `Vault Package`가 설치되어 있는 상태

[OpenID Connect (OIDC) Authentication Using ID Tokens | GitLab](https://docs.gitlab.com/ee/ci/secrets/id_token_authentication.html#manual-id-token-authentication)

```yaml
## Vault Command 사용 (Runner excutor:shell)
## https://docs.gitlab.com/ee/ci/secrets/id_token_authentication.html#manual-id-token-authentication
vault_command:
  # image: hashicorp/vault             # Vault 이미지 지정 / excutor : shell 설정일 경우 주석 처리
  stage: test
  variables:                         # 환경변수 지정
    # VAULT_ADDR: http://15.222.62.110:8200 # CI/CD 환경 변수 값 사용
    VAULT_NAMESPACE: gitlab_demo
  id_tokens:
    VAULT_ID_TOKEN:                  # JWT 값이 저장 될 환경 변수
      aud: http://15.222.62.110:8200 # Vault 주소
  script:
    - export VAULT_TOKEN=$(vault write -field=token auth/jwt/login jwt=$VAULT_ID_TOKEN role=gitlab_role)
    - vault kv get -field=password kv/secret

stages:
  - test
```

![Untitled 2](https://github.com/jslim1995/insideinfo-vault/assets/100335118/3ec78426-ac1a-4140-8a38-1bff6a25a6d0)

### 2. Vault API 사용

Pipeline에서 `JWT`를 발급 받아 `script` 블록에서 `Vault API`를 사용하여 Vault 토큰 발급 및 `Vault KV` 값 조회

[OpenID Connect (OIDC) Authentication Using ID Tokens | GitLab](https://docs.gitlab.com/ee/ci/secrets/id_token_authentication.html#manual-id-token-authentication)

```yaml
## API 사용 시
vault_api:
  stage: test
  variables:                                # 환경변수 지정
    # VAULT_ADDR: http://15.222.62.110:8200 # CI/CD 환경 변수 값 사용
    VAULT_NAMESPACE: gitlab_demo
  id_tokens:
    VAULT_ID_TOKEN:                          # JWT 값이 저장 될 환경 변수
      aud: http://15.222.62.110:8200         # Vault 주소
  script:
    - export VAULT_TOKEN=$(curl -X PUT -H "X-Vault-Namespace:${VAULT_NAMESPACE}/" -H "X-Vault-Request:true" -d "{\"jwt\":\"${VAULT_ID_TOKEN}\",\"role\":\"gitlab_role\"}" ${VAULT_ADDR}/v1/auth/jwt/login | jq -r .auth.client_token)
    - curl -H "X-Vault-Token:${VAULT_TOKEN}" -H "X-Vault-Request:true" -H "X-Vault-Namespace:${VAULT_NAMESPACE}/" ${VAULT_ADDR}/v1/kv/data/secret | jq -r .data.data.password

stages:
  - test
```

![Untitled 3](https://github.com/jslim1995/insideinfo-vault/assets/100335118/78e9625c-d1c4-4c8f-9779-7459a73b7728)

### 3. GitLab secrets:vault 사용

Pipeline에서 `JWT`를 발급 받아 `secrets:vault`를 사용하여 Vault 토큰 발급 및 `Vault KV` 값을 조회하여 환경 변수에 저장

환경 변수에 저장된 `Vault KV` 값은 마스킹 처리

사용 조건 : GitLab License 필요

[Tutorial: Update HashiCorp Vault configuration to use ID Tokens | GitLab](https://docs.gitlab.com/ee/ci/secrets/convert-to-id-tokens.html#kv-secrets-engine-v2)

[CI/CD YAML syntax reference | GitLab](https://docs.gitlab.com/ee/ci/yaml/index.html#secretsvault)

```yaml
## GitLab License가 있을 경우 사용 가능
## https://docs.gitlab.com/ee/ci/yaml/index.html#secretsvault
secrets_vault:
  stage: test
  variables:                         # 환경변수 지정
    VAULT_SERVER_URL: http://15.222.62.110:8200
    VAULT_AUTH_PATH: jwt
    VAULT_AUTH_ROLE: gitlab_role
    VAULT_NAMESPACE: gitlab_demo
  id_tokens:	                       # JWT 발급
    VAULT_ID_TOKEN:
      aud: http://15.222.62.110:8200
  secrets:
    PASSWORD_CASE1:                  # Vault KV 조회 방법 1.
      vault:
        engine:
          name: kv-v2                # KV ENGINE 명
          path: kv                   # KV ENGINE 경로
        field: password              # KV KEY 값
        path: secret                 # KV PATH 값 
      file: false
    PASSWORD_CASE2:                  # Vault KV 조회 방법 2.
      vault: secret/password@kv      # <KV_PATH>/<KV_FIELD>@<KV_ENGINE_PATH>
      file: false
  script:
    - echo $PASSWORD_CASE1 | tee /tmp/secret1
    - echo $PASSWORD_CASE2 | tee /tmp/secret2

stages:
  - test
```

![Untitled 4](https://github.com/jslim1995/insideinfo-vault/assets/100335118/2bf522bb-4024-4c42-912e-720bced35a6b)

# 참고

### Manual ID Token authentication

[OpenID Connect (OIDC) Authentication Using ID Tokens | GitLab](https://docs.gitlab.com/ee/ci/secrets/id_token_authentication.html#manual-id-token-authentication)

### HashiCorp Vault authentication

[Authenticating and reading secrets with HashiCorp Vault | GitLab](https://docs.gitlab.com/ee/ci/examples/authenticating-with-hashicorp-vault/)

### Tutorial

[Tutorial: Update HashiCorp Vault configuration to use ID Tokens | GitLab](https://docs.gitlab.com/ee/ci/secrets/convert-to-id-tokens.html)

### secrets:vault

[CI/CD YAML syntax reference | GitLab](https://docs.gitlab.com/ee/ci/yaml/index.html#secretsvault)
