# Jenkins Plugin 수동 구성

# 방법 1. CLI

## plugin 다운로드

`jenkins-plugin-manager` 를 사용하여 `jenkins_plugin`을 한번에 다운로드

```bash
# jenkins plugin manager war 파일 다운로드
# https://github.com/jenkinsci/plugin-installation-manager-tool/releases/tag/2.12.13
$ curl -OL https://github.com/jenkinsci/plugin-installation-manager-tool/releases/download/2.12.13/jenkins-plugin-manager-2.12.13.jar

# jenkins 기본 plugin 및 vault plugin 목록 파일 생성
# https://github.com/jenkinsci/jenkins/blob/master/core/src/main/resources/jenkins/install/platform-plugins.json 에서 "suggested": true 인 옵션 목록 + vault plugin
tee plugins.txt -<<EOF
ant:latest
antisamy-markup-formatter:latest
build-timeout:latest
cloudbees-folder:latest
configuration-as-code:latest
credentials-binding:latest
email-ext:latest
git:latest
github-branch-source:latest
gradle:latest
ldap:latest
mailer:latest
matrix-auth:latest
pam-auth:latest
pipeline-github-lib:latest
pipeline-stage-view:latest
ssh-slaves:latest
timestamper:latest
workflow-aggregator:latest
ws-cleanup:latest
hashicorp-vault-plugin:latest
hashicorp-vault-pipeline:latest
EOF

# jenkins plugins 파일 다운로드
$ java -jar jenkins-plugin-manager-*.jar --war jenkins.war --plugin-file plugins.txt -d jenkins_plugins/

# jenkins_plugins 폴더 압축
$ tar -cvf jenkins_plugins.tar jenkins_plugins

# private 서버로 파일 전송
$ sudo scp -i jinsu.pem jenkins.war jenkins_plugins.tar rocky@172.32.3.227:/home/rocky

# private 서버 접속
$ sudo ssh -i jinsu.pem rocky@172.32.3.227
```

## Plugin 배포

### Jenkins가 Package로 구성 된 경우

```bash
# jenkins.service 파일에서 JENKINS_HOME 환경 변수 값 조회
$ systemctl show jenkins.service | grep 'JENKINS_HOME'
Environment=JENKINS_HOME=/var/lib/jenkins JENKINS_WEBROOT=%C/jenkins/war JAVA_OPTS=-Djava.awt.headless=true JENKINS_PORT=8080

# JENKINS_HOME 환경 변수 설정
$ export JENKINS_HOME=/var/lib/jenkins

# JENKINS_HOME 환경 변수 확인
$ echo $JENKINS_HOME

# jenkins_plugins 폴더 압축 해제
$ tar -xvf jenkins_plugins.tar

# jenkins plugin install
$ cp jenkins_plugins/* $JENKINS_HOME/plugins

# jenkins 재시작
$ sudo systemctl restart jenkins.service
```

### Jenkins가 war로 구성된 경우

```bash
# JENKINS_HOME 환경 변수 설정
$ export JENKINS_HOME=$HOME/.jenkins

# JENKINS_HOME 환경 변수 설정
$ export JENKINS_HOME=/var/lib/jenkins

# jenkins_plugins 폴더 압축 해제
$ tar -xvf jenkins_plugins.tar

# jenkins plugin install
$ cp jenkins_plugins/* $JENKINS_HOME/plugins

# jenkins-cli.jar 다운로드
$ curl -O localhost:8080/jnlpJars/jenkins-cli.jar

# jenkins 재시작
$ java -jar jenkins-cli.jar -s http://localhost:8080/ -auth admin:password restart
```

# 방법 2. UI

## Plugin 다운로드

[Index of /download/plugins](https://updates.jenkins-ci.org/download/plugins/)

Jenkins plugin 다운로드 사이트에서 `hashicorp-vaulg-plugin.hpi`,  `hashicorp-vault-pipeline.hpi`파일 다운로드

![Untitled](https://github.com/jslim1995/insideinfo-vault/assets/100335118/b811f744-e915-4255-aac8-996a758352d5)

## Plugin 배포

jenkins 서버 접속 후 Dashboard → Manage Jenkins → Plugins → Advanced settings → Deploy Plugin 에서 `hashicorp-vaulg-plugin.hpi`,  `hashicorp-vault-pipeline.hpi`파일 업로드 후 Deploy

### 1. Plugin Advanced settings 이동

![Untitled 1](https://github.com/jslim1995/insideinfo-vault/assets/100335118/a22fac60-5c28-4e3c-a6f4-ebfceb85abea)

### 2. `hashicorp-vaulg-plugin.hpi` 업로드 및 배포

의존성이 있으므로 `hashicorp-vaulg-plugin` 먼저 배포

![Untitled 2](https://github.com/jslim1995/insideinfo-vault/assets/100335118/b5540edb-53f4-4de7-ae67-7dd7996f5345)

### 3. `hashicorp-vault-pipeline.hpi` 업로드 및 배포

![Untitled 3](https://github.com/jslim1995/insideinfo-vault/assets/100335118/f58fdd88-8a17-4fd9-80cd-e80c3e7bfff5)

### 4. jenkins 재기동

‘설치가 끝나고 실행중인 작업이 없으면 Jenkins 재시작’ 클릭

![Untitled 4](https://github.com/jslim1995/insideinfo-vault/assets/100335118/168be03d-ee07-4c51-93bc-ff8ce5cb2409)

### 5. vault plugin 설치 확인

Dashboard → Jenkins 관리 → Plugins → Installed plugins에서 vault 검색

![제목 없음](https://github.com/jslim1995/insideinfo-vault/assets/100335118/964795bc-e824-427e-957a-48e4d6b0e329)
