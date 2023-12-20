# aws-sm

# 개요

<aside>
💡 vault에 저장된 kv 값을 aws-secret-manager의 보안 암호 값에 단방향 동기화

(aws-secrets-manager에서 값 변경 시 vault의 kv값은 변경되지 않음)

- [현재는 KV-v2 엔진만 지원](https://developer.hashicorp.com/vault/api-docs/system/secrets-sync#set-association)
</aside>

특정 상황에서는 Vault에서 직접 비밀을 가져오는 것이 불가능하거나 비실용적입니다. 이 문제를 해결하기 위해 Vault는 일부 클라이언트가 더 쉽게 액세스할 수 있는 다양한 대상으로 KVv2 비밀에 대한 단방향 동기화를 유지할 수 있습니다. 이를 통해 Vault는 기록 시스템으로 유지되지만 신뢰할 수 있는 라스트마일 전달 시스템 역할을 하는 다양한 외부 시스템에 비밀의 하위 집합을 캐시할 수 있습니다.

Vault KVv2 비밀 엔진에서 외부 대상으로 연결된 비밀은 지속적인 프로세스에 의해 적극적으로 관리됩니다. Vault에서 비밀 값이 업데이트되면 대상에서도 비밀이 업데이트됩니다. Vault에서 비밀을 삭제하면 외부 시스템에서도 삭제됩니다. 이 프로세스는 비동기식이며 이벤트 기반입니다. Vault는 몇 초 안에 수정 사항을 적절한 대상에 자동으로 전파합니다.

![Untitled](https://github.com/jslim1995/insideinfo-vault/assets/100335118/467c48bd-515e-4b09-b2c9-7836262a0304)

Vault Enterprise Secrets Sync는 Vault와 AWS Secrets Manager 간의 비밀을 자동으로 동기화하므로 개발자는 Vault에서 비밀을 사용하는 방법을 배울 필요가 없습니다. 운영자는 비밀 수명주기 관리를 위해 Vault Enterprise의 강력한 기능을 계속 활용할 수 있지만 개발자는 플랫폼 기본 도구 및 기술을 통해 secret을 사용할 수 있습니다.

![Untitled 1](https://github.com/jslim1995/insideinfo-vault/assets/100335118/cc39fd95-042a-47d3-b6f9-6e6e7d5a4a3c)

동기화 **destinations은** 비밀을 동기화할 수 있는 위치를 나타냅니다. 현재 대상에는 AWS Secrets Manager, Azure Key Vault, GCP Secret Manager, GitHub 저장소 비밀 및 Vercel 프로젝트 환경 변수 등 5가지 유형이 있습니다. 하나의 비밀은 여러 대상과 연결될 수 있습니다. 예를 들어 CI 워크플로에 사용되는 Vault Secret로 저장된 Jira 토큰은 여러 GitHub 저장소에 동기화될 수 있으며, 각 저장소는 고유한 Vault 동기화 대상으로 표시됩니다.

비밀 **associations****은** 기존 Vault KV 비밀과 동기화 대상 간의 링크를 나타냅니다. 연결은 세분화되어 있으며 각 비밀에는 연결이 있습니다. 많은 associations가 destination를 공유할 수 있습니다.

# 환경

- vault : 1.15.0+ent

# AWS 구성

`Secrets Sync`를 사용하는 데 필요한 권한을 포함한 계정 생성

### AWS-SM-Policy

`secretsmanager` 관련 권한 생성

```bash
{
    "Version": "2012-10-17",
    "Statement": [
		{
            "Sid": "Statement1",
            "Effect": "Allow",
            "Action": [
                "secretsmanager:Create*",
                "secretsmanager:Update*",
                "secretsmanager:Delete*",
                "secretsmanager:TagResource"
            ] ,
            "Resource": "arn:aws:secretsmanager:*:*:secret:vault*"
        }
    ]
}
```

### Access Key, Secret Key

AWS-SM-Policy를 할당 받은 사용자의 `Access Key`, `Secret Key`

```bash
# Access Key
AKIAWEICIKPSBWCUS6XD
# Secret Key
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
```

---

# Vault 구성

### KV 엔진 구성

```bash
# KV V2 엔진 생성
$ vault secrets enable -path=my-kv kv-v2

Success! Enabled the kv-v2 secrets engine at: my-kv/

# KV 값 생성
$ vault kv put -mount=my-kv my-secret foo='bar'

==== Secret Path ====
my-kv/data/my-secret

======= Metadata =======
Key                Value
---                -----
created_time       2023-10-04T01:33:47.161455425Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1

# KV 값 조회
$ vault kv get -mount=my-kv my-secret

==== Secret Path ====
my-kv/data/my-secret

======= Metadata =======
Key                Value
---                -----
created_time       2023-10-04T01:33:47.161455425Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1

=== Data ===
Key    Value
---    -----
foo    bar
```

### sync/aws-sm 구성

```bash
# AWs Secrets Manager 대상 생성
$ vault write sys/sync/destinations/aws-sm/test \
access_key_id=AKIAWEICIKPSBWCUS6XD \
secret_access_key=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

Key                   Value
---                   -----
connection_details    map[access_key_id:***** secret_access_key:*****]
name                  test
type                  aws-sm

# 연관 설정 (현재 KV-v2 엔진만 지원)
$ vault write -f /sys/sync/destinations/aws-sm/test/associations/set \
mount=my-kv \
secret_name=my-secret

Key                   Value
---                   -----
associated_secrets    map[kv_6f196233/my-secret:map[accessor:kv_6f196233 secret_name:my-secret sync_status:SYNCED updated_at:2023-10-04T04:18:07.060590584Z]]
store_name            test
store_type            aws-sm
```

### 값 확인

AWS Secrets Manager - 보안 암호 - vault/kv_6f196233/my-secret - 보안 암호 값 - 보안 암호 값 검색

![Untitled 2](https://github.com/jslim1995/insideinfo-vault/assets/100335118/1d4d8b20-a864-43ad-810a-8e015e62a747)

# 참고 페이지

### Vault Doc

[AWS Secrets Manager - Secrets Sync Destination | Vault | HashiCorp Developer](https://developer.hashicorp.com/vault/docs/sync/awssm)

### Vault API Doc

[/sys/sync - HTTP API | Vault | HashiCorp Developer](https://developer.hashicorp.com/vault/api-docs/system/secrets-sync)

### Vault Tutorial

[Vault Enterprise Secrets Sync | Vault | HashiCorp Developer](https://developer.hashicorp.com/vault/tutorials/enterprise/secrets-sync)
