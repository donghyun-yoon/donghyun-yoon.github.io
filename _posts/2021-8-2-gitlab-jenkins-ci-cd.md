---
layout: post
title: "[DevOps] AWS EC2에서 Docker를 통한 GitLab-Jenkins Pipeline 자동배포(CI/CD)"
categories: devops
tags: [devops ,jenkins, gitlab, AWS, ec2]
date: 2021-08-02
---

> 도커(Docker)가 깔려있다는 가정하에 진행합니다.
>
> 꼭 도커에서 작업할 필요없이 ec2에서 jenkins, local에서 jenkins를 다운받아 사용하셔도 됩니다.

<br/>

![](/assets/img/DevOps/CI_CD_Flow.png)

<br/>

제가 이번에 배포하게 될 흐름입니다.    
순서대로 진행되기때문에 참고하시면 좋을꺼같습니다.

제가 찾아본 서버에 배포하는 방법은
1. (ec2에 젠킨스와 결과물 같이 올릴경우) docker의 경우 -v 옵션으로 결과물 폴더를 mount해서 사용하는 방법
2. (ec2에 젠킨스와 도커가 있을경우) jenkins에서 빌드된 결과물을 docker로 바로 실행
3. (젠킨스가 실행되는 곳과 다른 원격서버일 경우) SSH명령을 통해 결과물 전달
4. (젠킨스가 실행되는 곳과 다른 원격서버일 경우) 빌드된 결과물을 Docker Hub에 올려서 결과물 전달

저는 declarative pipeline 사용하였고 SSH를 통해 원격 서버에 배포해줍니다.

SSH를 이용한 이유는 현재 프로젝트는 ec2 한개로 서버로 사용하지만 추후에는 젠킨스 빌드용 서버 배포용 서버를 따로 사용하는 경우도 있다고 생각하여 SSH를 이용하기로 하였습니다.

<br/>


## Docker에서 젠킨스 다운

도커 위에 젠킨스를 다운로드 받습니다.

`docker pull jenkins/jenkins`    
(도커허브를 이용하여 사용하실분들은 도커가 깔려있는 젠킨스를 다운 받으시면 됩니다.)

`docker run -d -u root -p 9090:8080 --name=jenkins jenkins/jenkins`    
(저는 9090포트로 접속하겠다고 설정하였고 도커의 옵션들은 추후 포스팅할 예정입니다.)

이제 브라우저에서 public ipv4:9090 or 도메인명:9090 or 로컬에서 접속하고 계시다면 localhost:9090 으로 접속합니다.

## Jenkins 초기 설정

<br/>

![](/assets/img/DevOps/젠킨스_초기화면_패스워드.png)

<br/>

`docker exec -it jenkins cat /var/jenkins_home/secrets/initialAdminPassword` 명령어로 Administrator password 확인가능합니다.

아래 사진은 도커가 아닌 cmd창에서 패스워드 확인하는 모습입니다.

<br/>

![](/assets/img/DevOps/jenkins%20패스워드%20확인.png)

<br/>

그 후 Install suggested plugins을 선택합니다.

<br/>

![](/assets/img/DevOps/젠킨스_install_suggested_plugins.png)

<br/>

플러그인 설치가 완료된 후 Create User 창이 뜰것이고 본인의 계정명, 암호, 이름, 이메일 주소를 설정하고 Save and Continue를 클릭합니다. 앞으로 젠킨스를 로그인 할때 사용할 정보이므로 잘 기억하도록 합시다.

<br/>

![](/assets/img/DevOps/젠킨스_Create_User.png)

<br/>

젠킨스에 자동배포에 필요한 플러그인을 설치합시다.

Jenkins 관리 > 플러그인 관리 > 설치가능
- GitLap
- Generic Webhook Trigger
- Gitlab Authentication
- Gitlab API
- Gitlab Merge Request Builder
- Blue Ocean
- Config File Provider
- Publish Over SSH


<br/>

## GitLab Access Token 생성

<br/>

gitlab > settings > Access Tokens를 클릭하고

![](/assets/img/DevOps/깃랩_엑세스_토큰_생성.png)

<br/>

이름과 유효기간 그리고 scope 범위는 필요한 만큼 체크해서 생성해줍니다.

![](/assets/img/DevOps/깃랩_엑세스_토큰값.png)

<br/>

위에 네모박스에 토큰값이 생기는데 절대 잊어버리지 않게 기록해두록 합시다.

<br/>

## Jenkins + Gitlab 연동

<br/>

Jenkins 관리 > 시스템설정 > Gitlab

![](/assets/img/DevOps/젠킨스_웹훅_설정.png)

<br/>

Connerction name을 정해시고 Gitlab URL을 작성해주시고 키모양의 Add키를 눌러 새로운 api 토큰을 가지고 연결을 시켜줍니다.

<br/>

![](/assets/img/DevOps/젠킨스_api토큰_설정.png)

<br/>

아까 Gitlab에서 발급한 API 토큰 값을 넣고 Test Connertion을 통해 연결을 확인해 봅시다.

<br/>

## Pubilc over SSH

<br/>

**해당부분은 ssh를 사용하실 분만 설정해주시면 됩니다.**

Jenkins 관리 > 시스템 설정  > Publish over SSH

Key : .pem 키 안의 내용

Name : jenkins에서 사용하는 이름

username : ec2-user이름

고급에서 `Use password authentication, or use a different key`를 선택

Test Configuration을 실행하여 성공하는 것을 확인 해봅니다.

<br/>

## Jenkins Pipeline 생성

<br/>

![](/assets/img/DevOps/젠킨스_item_생성.png)

<br/>

![](/assets/img/DevOps/젠킨스_pipeline_생성.png)

<br/>

다양한 방법이 있지만 여기서는 Pipeline으로 진행하도록 하겠습니다.

<br/>

## Gitlab webhook으로 연결하기

<br/>

webhook이란 간단히 말하면 어떤 이벤트가 발생했을때 클라이언트에게 알려 주는 것    
즉, gitlab에서 설정한 어떤 변경사항이 생기게 되면 jenkins의 특정 item과 연동할 수 있다.

파이파인을 생성하면 `Build when a change is pushed to GitLab`을 클릭합니다.

<br/>

![](/assets/img/DevOps/젠킨스_깃랩_웹훅_설정.png)

<br/>

![](/assets/img/DevOps/젠킨스_깃랩_웹훅_설정_트리거.png)

<br/>

여기서 다양한 트리거를 생성할 수있습니다.    
그리고 `고급`을 선택하면  

<br/>

![](/assets/img/DevOps/젠킨스_시크릿토큰생성.png)

<br/>

`Secret token`을 생성해줍니다. 깃랩 웹훅 설정에 필요하므로 기억하도록 합시다.

그리고 `GitLab webhook URL`을 복사하여 Gitlab에서 Webhook 설정을 하도록합시다.

> locahost의 경우 `ngrok`을 사용하여 public url로 바꿔서 사용합시다!!

Gitlab > settings > webhook

<br/>

![](/assets/img/DevOps/깃랩_웹훅_설정.png)

<br/>

저는 간단하게 maseter 브랜치 push events가 발생했을때만 jenkins가 동작하도록 하였습니다.

<br/>

![](/assets/img/DevOps/깃랩_엡훅_테스트.png)

<br/>

Test를 통해서 연결을 확인할수있습니다.

<br/>

이제 Pipeline 문법에 따라 작성하면 됩니다.

```
pipeline {
    agent any
    
    tools {
        nodejs "nodejs"
    }

    parameters {
        string(name: 'GIT_URL', defaultValue: 'https://아이디:비밀번호@깃랩 레포지토리 주소.git', description: 'GIT_URL')
        booleanParam(name: 'VERBOSE', defaultValue: false, description: '')
    }

    environment {
        GIT_BUSINESS_CD = 'master'
        GIT_CREDENTIAL_ID = '아이디'
        VERBOSE_FLAG = '-q'
    }

    stages{
        stage('Preparation'){
            steps{
                script{
                    env.ymd = sh (returnStdout: true, script: ''' echo date '+%Y%m%d-%H%M%S' ''')
                }
                echo("params : ${env.ymd} " + params.tag)
            }
        }

        stage('Checkout'){
            steps{
                git(branch: "${env.GIT_BUSINESS_CD}",
                credentialsId: "${env.GIT_CREDENTIAL_ID}", url: params.GIT_URL, changelog: false, poll: false)
            }
        }

        stage('Build'){
            steps {
                sh "npm install"
                sh "npm run build"
            }
        }

        stage('SSH transfer'){
            steps([$class: 'BapSshPromotionPublisherPlugin']) {
				sshPublisher(
					continueOnError: false, failOnError: true,
					publishers: [
						sshPublisherDesc(
							configName: "Pubilc over SSH에서 설정한 Name",
							verbose: true,
							transfers: [
								sshTransfer(
									sourceFiles: "전송할 파일 경로",
                                    removePrefix: "파일에서 삭제할 경로가 있다면 작성", 
                                    remoteDirectory: "배포할 위치",
                                    execCommand: "원격지에서 실행할 커맨드" 
								)
							]
						)
					]
				)
			}
        }

}
```

<br/>

**`SSH를 통해 빌드된 결과물을 서버에 저장하고 execCommand로 docker-compose를 실행하게 되면 자동으로 배포가 됩니다.`**