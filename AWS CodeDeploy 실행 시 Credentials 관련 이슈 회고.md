# AWS CodeDeploy 실행 시 Credentials 관련 이슈 회고

## 1. 에러 내용
```
# /var/log/aws/codedeploy-agent/codedeploy-agent.log

InstanceAgent::Plugins::CodeDeployPlugin::CommandPoller: Cannot reach InstanceService: Aws::CodeDeployCommand::Errors::AccessDeniedException
```

## 2. 이슈 현상
기존 AWS CodeDeploy를 통해 배포를 진행하던 EC2 인스턴스의 AMI 생성 시 순단이 발생하며 인스턴스가 재부팅 되었고 이후 배포 진행 시 CodeDeploy에서 해당 인스턴스로 접근을 하지 못하는 현상이 발생하게됨

## 3. 이슈 원인
해당 이슈는 AWS의 Credential Loading 순서와 관련된 이슈로 Host Agent의 경우 기본적으로 Root User로 실행되며 Instance Profile을 사용하지만 Instance Profile보다 우선순위의 Root Credential을 설정한 경우 예외가 발생하게됨

## 4. 조치 사항
Instance Profile 사용을 위해 Loading 우선 순위로 설정된 root 디렉토리 내부의 .aws/credentials 파일 삭제 후 codedeploy-agent 재실행
```
rm -rf /root/.aws/credentials
# 또는
rm -rf /home/{user}/.aws/credentials

systemctl restart codedeploy-agent
```

## 5. 참조 링크
https://repost.aws/ko/questions/QUUQG8erWXTj6KlQ1Asa4siw/aws-code-deploy-cannot-reach-instance-service   
https://stackoverflow.com/questions/37721601/aws-code-deploy-deployment-failed