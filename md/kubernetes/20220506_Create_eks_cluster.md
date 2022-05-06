EKS cluster를 생성하는 다양한 방법이 있지만, 그 중 eksctl을 사용하여 간단하게 테스트용 cluster를 생성해보고자 합니다.

---

**테스트 환경**

- AWS EC2 Instance (Amazon Linux)

---

# Install eksctl

[eksctl 설치하기 - AWS 공식 문서](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html)  

위 링크를 참조하여 본인의 환경에 맞는 방법으로 eksctl을 설치합니다.  
저는 Amazon Linux 환경이기 때문에, 아래 절차대로 설치했습니다.

## eksctl binary 추출

```sh
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
```

## eksctl을 어디서든 사용할 수 있도록 바이너리를 기본 디렉토리로 이동

```sh
sudo mv /tmp/eksctl /usr/local/bin
```

## eksctl 동작 확인

```sh
eksctl version
```

</br></br>

---
# AWS IAM User 생성하기

eksctl을 이용하여 EKS cluster를 생성하기 위해서는 아래의 두 가지가 필요합니다.

- EKS Cluster 생성 권한이 있는 AWS IAM User
- 해당 User의 Access key ID, Secret access key

EKS cluster를 만들기 위해서는 VPC, EC2, EBS 등 여러 가지 리소스가 필요하며, 해당 리소스들을 생성할 수 있는 권한을 가진 사용자를 등록해야 eksctl로 cluster 생성이 가능합니다.

테스트 환경에서는 Admin 권한을 가진 사용자를 따로 생성하여 진행했습니다.

<div class="warning" style="background-color:#87cefa; border-radius:4px; padding:8px; margin:8px;">
추후 어떤 권한이 필요한지 파악해서 특정 권한만 활성화시킨 사용자로 테스트 해 볼 예정이고, 이번 테스트에서는 admin으로 진행합니다.
</div>

</br></br>

![iam service](https://user-images.githubusercontent.com/78069372/167083317-d7fd2a35-bc7b-419c-ad60-a3d973901126.png)

AWS Console의 상단부에서 `IAM`을 검색하고, 검색 결과 중 `IAM`을 클릭합니다.

</br></br>

![create user click](https://user-images.githubusercontent.com/78069372/167083338-24061f80-ba60-404f-9628-ad19964fa1ed.png)

왼쪽 탐색바에서 `액세스 관리` - `사용자`를 클릭하고, `사용자 추가` 버튼을 클릭합니다.

</br></br>

![create user](https://user-images.githubusercontent.com/78069372/167085339-e88ebf89-b7f7-43b0-94f6-6e0307e311b2.png)

`사용자 이름`을 입력하고, `액세스 키`와 `암호`를 체크하여 활성화합니다.

<div class="warning" style="background-color:#87cefa; border-radius:4px; padding:8px; margin:8px;">
콘솔 비밀번호와 비밀번호 재설정 필요 항목은 위와 같이 체크해두면 사용자 생성 완료 후에 자동 생성된 암호를 발급받게 되고, 최초 로그인시 새로운 비밀번호를 생성하게 됩니다.
</div>

<div class="warning" style="background-color:#f08080; border-radius:4px; padding:8px; margin:8px;">
보안적인 측면에서 저는 위와 같이 설정한 후, 해당 사용자로 콘솔 최초 로그인 시 비밀번호 변경과 함께 MFA(Multi Factor Authentication)을 설정하는 것을 선호합니다.
</div>

`다음: 권한` 버튼을 클릭합니다.

</br></br>

![add policy](https://user-images.githubusercontent.com/78069372/167085579-315b902c-a85c-465e-94c1-f4a77ff9b8c5.png)

상단에서 `기존 정책 직접 연결`을 선택하고 `AdministratorAccess`를 체크합니다.
`다음: 태그`를 클릭합니다.

</br></br>

![tagging user](https://user-images.githubusercontent.com/78069372/167086026-9102a746-40e0-4784-ad01-e2188debed0e.png)

태그 작성 여부는 선택사항입니다.

<div class="warning" style="background-color:#87cefa; border-radius:4px; padding:8px; margin:8px;">
리소스에 대한 추적을 위해 소유자나 생성 날짜, 설명을 간단하게 적는 것을 추천 드립니다.
</div>

`다음: 검토`를 클릭합니다.

</br></br>

![user create finish](https://user-images.githubusercontent.com/78069372/167086385-aec5e390-cadd-4711-915f-a1d713be1ac1.png)

`사용자 세부 정보`와 `권한 요약`이 선택한 대로 잘 설정되었는지 확인하고 `사용자 만들기`를 클릭합니다.

</br></br>

![user key](https://user-images.githubusercontent.com/78069372/167086737-32564903-8772-42d1-869e-cafde9b51cdf.png)

사용자 생성이 완료되고 위와 같이 생성 결과값을 확인할 수 있습니다.
여기서 저희가 필요한 것은 `액세스 키`와 `비밀 액세스 키`입니다.
표시 버튼을 클릭하여 값을 확인할 수 있습니다.
`비밀번호` 항목은 해당 사용자를 이용하여 AWS Console에 접속시 필요한 비밀번호입니다.
`.csv 다운로드` 버튼을 클릭하여 해당 정보를 저장해놓고 사용하는 것을 추천 드립니다.

<div class="warning" style="background-color:#87cefa; border-radius:4px; padding:8px; margin:8px;">
액세스 키와 비밀 액세스키는 외부에 유출되면 안되는 정보입니다.
특히 현재 테스트와 같이 관리자 권한을 준 상태로 외부에 유출되게 되면, 타인이 해당 계정에 접속하여 리소스를 마구 생성하거나 기존 리소스를 지우는 식의 안 좋은 행동을 할 수 있습니다.
</div>

<div class="warning" style="background-color:#f08080; border-radius:4px; padding:8px; margin:8px;">
AWS는 액세스 키 노출 여부를 모니터링하고 있습니다.
실제로 노출된 경우에는 해당 액세스키에 대하여 즉각적으로 기본적인 조치를 취한 뒤, 해당 계정에 유출 여부를 이메일을 통해 알려주고 추가 조치를 취할 것을 권고합니다.
</div>

</br></br>

---

# 테스트 환경에 aws configure 적용하기

새로 생성한 사용자의 Access key ID와 Secret access key를 터미널에서 configure합니다.

```sh
aws configure
```

커맨드를 입력하고 순서대로 입력합니다.
- AWS Access Key ID
- AWS Secret Access Key
- Default region name
- Default output format

![aws configure](https://user-images.githubusercontent.com/78069372/167090626-2bcca3d7-69c8-4a53-81e2-11e5b50f932d.png)

제 경우에는 이미 입력된 값이 있기 때문에 위와 같이 출력됐습니다.

</br></br>

---

# ekscluster config yaml file 만들기

`eksctl`을 이용하여 cluster를 생성하는 경우, 커맨드 + 옵션을 이용해 간단하게 만들 수 있지만, 설정을 보다 편하게 하기 위해 yaml file을 만들어서 사용하겠습니다.

### createcluster.yaml

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: test-cluster
  region: ap-northeast-2
  tags:
    environment: devel

vpc:
  cidr: "10.10.0.0/16" #"192.168.0.0/16"
  nat:
    gateway: Single

availabilityZones:
  - ap-northeast-2a
  - ap-northeast-2b
  - ap-northeast-2c

managedNodeGroups:
  - name: management
    instanceType: t3.medium
    minSize: 1
    desiredCapacity: 3
    maxSize: 4
    volumeType: gp3
    volumeSize: 30
    ssh:
      enableSsm: true
    availabilityZones: ["ap-northeast-2a","ap-northeast-2b","ap-northeast-2c"]

cloudWatch:
  clusterLogging:
    enableTypes:
      - api
      - audit
      - authenticator      
```

위 파일대로 cluster를 생성하면
- 10.10.0.0/16 cidr를 사용하는 vpc에
- nat gateway 하나를 생성하고
- ap-northeast-2 리전의 a,b,c 가용영역을 사용하는
- t3.medium 인스턴스 타입의 노드 3개를 가진 managed node group이 생성되며
- 각 노드는 gp3 타입의 30G짜리 volume을 가집니다.

<div class="warning" style="background-color:#87cefa; border-radius:4px; padding:8px; margin:8px;">
위에서 나열한 항목들은 모두 customize 가능한 것입니다. 제 경우 테스트 해 볼 k8s resource가 많기 때문에 t3.medium 타입의 인스턴스를 사용하지만, 비교적 간단한 테스트를 진행할 예정이라면 t3.micro로도 충분합니다.
각 노드의 volume 용량이나 type, vpc cidr, 가용영역 등 모두 customize 가능하기 때문에, 본인이 테스트 하고자 하는 것에 따라 편하게 변경해서 사용하시면 됩니다.
</div>

<div class="warning" style="background-color:#f08080; border-radius:4px; padding:8px; margin:8px;">
다만, 이 createcluster.yaml에는 k8s cluster에 설치하는 addon과 같은 좀 더 추가적인 내용은 전혀 포함되어 있지 않습니다.
추가 기능을 더 적용해야 하는 경우, 아래의 eksctl config file schema를 참조하시거나 github을 검색해서 적용하시는 것을 추천 드립니다.
</div>

[eksctl config schema](https://eksctl.io/usage/schema/)

</br></br>

---

# EKS Cluster 만들기

위에서 eks cluster를 만들기 위한 설정 파일을 만들었기 때문에 간단한 명령어로 cluster 생성이 가능합니다.

```sh
eksctl create cluster -f createcluster.yaml
```

<div class="warning" style="background-color:#87cefa; border-radius:4px; padding:8px; margin:8px;">
EKS Cluster 생성에는 많은 시간이 소요됩니다. config file에 명시한 내용에 따라 몇십분까지도 소요되므로 인내심을 갖고 기다리셔야 합니다...
</div>

eksctl은 AWS CloudFormation이라는 서비스를 통해 EKS Cluster를 생성합니다.
eksctl 명령어를 날리면
- CloudFormation에 cluster를 생성하라는 스택이 만들어지고
- cluster 생성이 완료되면 node group을 생성하라는 스택이 만들어집니다.

![Cloudformation](https://user-images.githubusercontent.com/78069372/167093586-a83c4914-e18b-43c3-9620-050113d93d83.png)

위와 같이, 스택이 총 두개 생기는데, cluster 생성 스택이 먼저 생기고 그것이 완료되면 node group 생성 스택이 생기는 순서입니다.

![stack info](https://user-images.githubusercontent.com/78069372/167093942-59928210-6f56-4a45-968f-d73f40a48201.png)

생성중에도 스택 이름을 클릭하여 위와 같이 어떤 리소스가 생성되고 있는지 확인이 가능합니다.