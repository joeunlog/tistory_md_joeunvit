EKS cluster를 생성하는 다양한 방법이 있지만, 그 중 eksctl을 사용하여 간단하게 테스트용 cluster를 생성해보고자 합니다.

**테스트 환경**

-   AWS EC2 Instance (Amazon Linux)

# Install eksctl

[eksctl 설치하기 - AWS 공식 문서](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html)  

위 링크를 참조하여 본인의 환경에 맞는 방법으로 eksctl을 설치합니다.  
저는 Amazon Linux 환경이기 때문에, 아래 절차대로 설치했습니다.

## eksctl binary 추출

```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
```

## eksctl을 어디서든 사용할 수 있도록 바이너리를 기본 디렉토리로 이동

```
sudo mv /tmp/eksctl /usr/local/bin
```

## eksctl 동작 확인

```
eksctl version
```

# AWS IAM User 생성하기

eksctl을 이용하여 EKS cluster를 생성하기 위해서는 아래의 두 가지가 필요합니다.

-   EKS Cluster 생성 권한이 있는 AWS IAM User
-   해당 User의 Access key ID, Secret access key

EKS cluster를 만들기 위해서는 VPC, EC2, EBS 등 여러 가지 리소스가 필요하며, 해당 리소스들을 생성할 수 있는 권한을 가진 사용자를 등록해야 eksctl로 cluster 생성이 가능합니다.

테스트 환경에서는 Admin 권한을 가진 사용자를 따로 생성하여 진행했습니다.

<div class="warning" style="background-color:#87cefa;"><em>
추후 어떤 권한이 필요한지 파악해서 특정 권한만 활성화시킨 사용자로 테스트 해 볼 예정이고, 이번 테스트에서는 admin으로 진행합니다.
</em></div>

## AWS IAM 서비스 접속


AWS Console의 상단부에서 `IAM`을 검색하고, 검색 결과 중 `IAM`을 클릭합니다.

[##_Image|kage@PFLWJ/btrBqaF9PQh/ktW0a2xD3HgRv41fTPUgOK/img.png|CDM|1.3|{"originWidth":1720,"originHeight":1296,"style":"alignCenter"}_##]

왼쪽 탐색바에서 `액세스 관리` - `사용자`를 클릭하고, `사용자 추가` 버튼을 클릭합니다.

[##_Image|kage@D7p8Y/btrBoEBaWJq/9Z2kkCvnwmHX7AzxSQis2k/img.png|CDM|1.3|{"originWidth":1720,"originHeight":1296,"style":"alignCenter"}_##]

`사용자 이름`을 입력하고, `액세스 키`와 `암호`를 체크하여 활성화합니다.

> 콘솔 비밀번호와 비밀번호 재설정 필요 항목은 위와 같이 체크해두면 사용자 생성 완료 후에 자동 생성된 암호를 발급받게 되고, 최초 로그인시 새로운 비밀번호를 생성하게 됩니다.  
> 보안적인 측면에서 저는 위와 같이 설정한 후, 해당 사용자로 콘솔 최초 로그인 시 비밀번호 변경과 함께 MFA(Multi Factor Authentication)을 설정하는 것을 선호합니다.

# 테스트 환경에 aws configure 적용하기