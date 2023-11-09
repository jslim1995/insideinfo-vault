## 1. 사전작업

### 1. Vault와 Azure 연동을 위한 APP 등록

![Untitled](https://github.com/jslim1995/insideinfo-vault/assets/100335118/1866710c-1fdd-4907-96fa-839e93491899)

1. Entra ID 선택

![Untitled 1](https://github.com/jslim1995/insideinfo-vault/assets/100335118/0a844e43-8396-41e6-8eb0-df058b50139e)

2. 앱등록 → 새등록

![Untitled 2](https://github.com/jslim1995/insideinfo-vault/assets/100335118/644ecb19-a178-4df2-a1d0-da8afc6f78f2)

3. 이름입력→ 이 조직의 디렉터리 계정만 선택 → 등록

![Untitled 3](https://github.com/jslim1995/insideinfo-vault/assets/100335118/715e8da3-c345-4b0a-8630-57fe838621cf)

4. App 생성 확인

### 2. APP의 Client Secret 생성

![Untitled 4](https://github.com/jslim1995/insideinfo-vault/assets/100335118/8342f1fa-fad7-4ce7-a7ae-e06b9e7028cc)

1. 인증서및 암호 → 새 클라이언트암호

![Untitled 5](https://github.com/jslim1995/insideinfo-vault/assets/100335118/919390a5-f610-41fe-8fae-0a195ea357d2)

2. 설명 입력 → 추가

![Untitled 6](https://github.com/jslim1995/insideinfo-vault/assets/100335118/d300c1c6-06a6-4c66-9407-769105969f8b)

3. 값 확인 후 저장 VN78Q~~4YWEnBJqRXvmvPkNwxE75zGP6MbHGccRM

### 3. API 권한 설정

![Untitled 7](https://github.com/jslim1995/insideinfo-vault/assets/100335118/b364afdf-6afe-44c6-bd12-aef921db360d)

1. API 사용권한 → 권한추가

![Untitled 8](https://github.com/jslim1995/insideinfo-vault/assets/100335118/46e5b3e3-0633-4cc9-930f-5f379d705d94)

2. Microsoft Graph 선택

![Untitled 9](https://github.com/jslim1995/insideinfo-vault/assets/100335118/3e4c3dde-72ed-4115-b5bf-9c6646741916)

3. 위임된 권한 및 어플리케이션 사용 권한 모두 설정

![Untitled 10](https://github.com/jslim1995/insideinfo-vault/assets/100335118/64c47dd6-1dfa-481f-b8be-0c23b117196c)

4. 권한 선택 → 권한 추가

| Permissions name | type |
| --- | --- |
| Application.Read.All | Delegated |
| Application.ReadWrite.All | Delegated |
| Directory.AccessAsUser.All | Delegated |
| Directory.Read.All | Delegated |
| Directory.ReadWrite.All | Delegated |
| Group.Read.All | Delegated |
| Group.ReadWrite.All | Delegated |
| GroupMember.Read.All | Delegated |
| GroupMember.ReadWrite.All | Delegated |

![Untitled 11](https://github.com/jslim1995/insideinfo-vault/assets/100335118/f8136688-f829-4594-acf5-2e9a8f50ee77)

5. 구성된 결과 확인

![Untitled 12](https://github.com/jslim1995/insideinfo-vault/assets/100335118/b5b806f2-dc61-4aed-98d0-44ca6746a1a4)

6. 기본 디렉터리에 대한 관리자 동의 허용 → 확인

![Untitled 13](https://github.com/jslim1995/insideinfo-vault/assets/100335118/06881ae5-d0a4-42aa-92e3-ac7c41e55c75)

7. 권한 확인

### 4. Subscription ID 확인

![Untitled 14](https://github.com/jslim1995/insideinfo-vault/assets/100335118/3bb580eb-e479-4e8c-9289-139cac654851)

1. 구독 선택

![Untitled 15](https://github.com/jslim1995/insideinfo-vault/assets/100335118/50f73da9-ca53-491f-b538-acf270245ae4)

구독 ID 확인

<img width="466" alt="Untitled 16" src="https://github.com/jslim1995/insideinfo-vault/assets/100335118/0653207a-6e05-4249-9a0a-7250e4a50c59">

## Vault AZURE 연동

```bash
# 1. Vault Azure Secret Enable
vault secrets enable -ns=uplus azure
Success! Enabled the azure secrets engine at: azure/

# 2. Vault Azure Secret Engine 확인
vault secrets list -ns=uplus 

# 3. Vault Azure 계정 구성
vault write -ns=uplus azure/config \
subscription_id=f2070420-f3e5-4f26-8f75-00270c257ac0 \
tenant_id=f2ae175d-6aa0-417e-8b62-2229585d1505 \
client_id=c7e00122-4855-49ee-be37-84b8bf604379 \
client_secret=hSo8Q~19Lau4zqlc6MjGidgHU8XRyJeqIBM8_c-p \
use_microsoft_graph_api=true
Success! Data written to: azure/config

# 4. Azure Role 생성
vault write -ns=uplus azure/roles/azure-object-role \
application_object_id=358d28d5-67e2-42b2-add8-0089dc0b3fba \
ttl=1m

# 5. 생성된 Role List 확인
vault list -ns=uplus azure/roles/
Keys
----
azure-object-role

# 6. Role 정보 확인
vault read -ns=uplus azure/roles/azure-object-role
Key                      Value
---                      -----
application_object_id    358d28d5-67e2-42b2-add8-0089dc0b3fba
azure_groups             <nil>
azure_roles              <nil>
max_ttl                  0s
permanently_delete       false
persist_app              false
ttl                      1m

# 7. Policy 생성
## 사용자가 aws Credential을 요청 권한을 주기 위한 policy 생성
vault policy write -ns=uplus azure_policy -<<EOF
path "azure/creds/azure-object-role" {
  capabilities = ["read"]
}
EOF

# 8. Entity 생성
# 사용자가 vault 로그인 시 entity 생성이 됨, 테스트에서는 미리 생성 해서 생성한 정책을 적용하여, 사용자가 로그인 했을때 엔티티가 연결되어 해당 정책에 맞게 동작
vault write -ns=uplus -format=json identity/entity \
name=azure@hashicorp.com \
policies=azure_policy \
| jq -r ".data.id" > azure_entity_id.txt

# 9. Accessor 확인
vault auth list -ns=uplus -format=json \
| jq -r '.["okta/"].accessor' > auth_okta_accessor.txt

# 1. Entity Alias 생성
vault write -ns=uplus identity/entity-alias \
name="azure@hashicorp.com" \
canonical_id=$(cat azure_entity_id.txt) \
mount_accessor=$(cat auth_okta_accessor.txt)
```

## Azure Credential 발급

```bash
# 1. Vault Login
# 적용된 정책 확인
vault login -method=okta -ns=uplus username=azure@hashicorp.com
Password (will be hidden):
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                    Value
---                    -----
token                  hvs.CAESIMthuwVcem7HV8XKrRlqV9vYYiH87UG7urr0emLsTnNaGicKImh2cy5oVk1jVk9WandMMlIzaHlaNlVTanN4S1cuaUFKYzIQ4AM
token_accessor         tCtuNg0fCqVCPIydDg2hdkW2.iAJc2
token_duration         768h
token_renewable        true
token_policies         ["default"]
identity_policies      ["azure_policy"]
policies               ["azure_policy" "default"]
token_meta_policies    n/a
token_meta_username    azure@hashicorp.com

# 2. Azure Credential 발급
## Azure 에서 인증서 및 암호 확인
## 앞에서 등록한 Service Principal (Application)에 발급된 키 확인
vault read -ns=uplus azure/creds/azure-object-role
Key                Value
---                -----
lease_id           azure/creds/azure-object-role/6hzE2pHpCtWAlno9ZsPHaC3z.z3Pni
lease_duration     1h
lease_renewable    true
client_id          c7e00122-4855-49ee-be37-84b8bf604379
client_secret      hQ.8Q~VqJNr9awOqJOqgU7U1om6vPAiqcJKgcbNo
```
