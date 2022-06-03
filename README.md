# EKS

## USE
1. Kubernetes
* EKS
  
2. CD tool
* ArgoCD

3. Monitoring
* Prometheus
* Grafna

## 구성
```
├── README.md
├── dump.rdb
├── eks초기설정
│   ├── AWSLoadBalancerControllerIAMPolicy
│   ├── cluster-role-trust-policy.json
│   ├── iam_policy.json
│   └── node-role-trust-policy.json
├── k8s
│   ├── deployment-apiserver.yaml
│   ├── deployment-authserver.yaml
│   ├── deployment-redis.yaml
│   ├── grafana.yaml
│   ├── ingress-argo.yaml
│   ├── ingress.yaml
│   ├── service-apiserver.yaml
│   ├── service-authserver.yaml
│   └── service-redis.yaml
├── make-cluster-2.yaml
├── make-cluster.md
└── make-cluster.yaml
```
## Project Explanation

## Step
1. EKS로 클러스터와 노드를 생성한다.
2. deploymnet로는 API Server, Auth Server, Redis 총 3개로 구성한다.
3. Service는 3개로써, API Server와 Auth Server는 nodePort로, Redis는 ClusterIP로 구성한다.
4. Ingress는 alb를 이용하여 구성한다
5. ArgoCD와 Github를 이용하여 클러스터를 관리한다.
6. 모니터링으로는 프로메티우스와 Grafana를 이용해 구성한다.
7. EKS 사용자 추가하기
8. 명령어 정리


## Start
EKS 클러스터를 생성하는 방법은 2가지가 있다.
첫번째 방법은 콘솔을 이용하는 것이고,
두번째 방법은 eksctl를 이용하는 방법이 있다.

### Getting Start

``` eksctl create cluster -f cluster.yaml ```

공식문서 (url 넣기)를 기반으로 eksctl을 설치 후 간단하게 클러스터를 생성할 수 있다.
하지만 클러스터만 생성 될 뿐 노드는 생성하지 못한다.

노드까지 생성할려면 다음 명령어를 실행하면 된다.

``` 
eksctl create cluster \
--name cluster-1 \
--version 1.21 \
--region us-west-2 \
--nodegroup-name c1-nodes \
--node-type t3.medium \
--nodes 2
```
이와같이 eksctl에 여러가지 옵션을 줄 수 있다.

yaml로도 클러스터와 노드를 정의해서 사용할 수 있다.

make-cluster.yaml을 실행시켜 원하는대로 노드와 권한, vpc 등을 정의해서 시작할 수 있다.

실행명령어는 다음과 같다.

``` kubectl apply -f make-cluster.yaml ```

### ekctl 설치 ~ alb설치 과정
1. ekctl 설치 : https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/eksctl.html
1-1. ekctl 다운
   curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

1-2. 압축 해제된 이진파일 옮기기
   sudo mv /tmp/eksctl /usr/local/bin

1-3. 버전확인
    eksctl version

2. eks 클러스터 및 노드 생성 : https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/getting-started-eksctl.html
2-1. 생성(생성되는데 시간이 조금 걸림)
    eksctl create cluster --name staging --region ap-northeast-2
    (eksctl create cluster --name my-cluster --region region-code)

2-2. 리소스 보기(클러스터 노드 확인)
    kubectl get nodes -o wide

2-3. 클러스터에 실행중인 워크로드 확인
    kubectl get pods --all-namespaces -o wide

2-4. 클러스터와 통신
aws eks update-kubeconfig --region region-code --name my-cluster

3. AWS Load Balancer Controller 설치 : https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/aws-load-balancer-controller.html
   
3-1. IAM 정책 생성
    curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.0/docs/install/iam_policy.json

3-2. IAM 정책 만들기
    aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json

3-3. eksctl을 이용해 kubectl을 사용하여 IAM 역할을 생성하고 AWS 로드 밸런서 컨트롤러의 kube-system 네임스페이스에 aws-load-balancer-controller라는 Kubernetes 서비스 계정을 추가


    eksctl create iamserviceaccount \
    --cluster=staging \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --attach-policy-arn=arn:aws:iam::060701521359:policy/AWSLoadBalancerControllerIAMPolicy \
    --approve

    (eksctl create iamserviceaccount \
    --cluster=my-cluster \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --attach-policy-arn=arn:aws:iam::111122223333:policy/AWSLoadBalancerControllerIAMPolicy \
    --override-existing-serviceaccounts \
    --approve)

    //no IAM OIDC provider associated with cluster란 문구가 뜨면 try 뒤 부터 복사한다음 --approve를 붙여서 실행 후 위 명령어 다시 실행
    eksctl utils associate-iam-oidc-provider --region=ap-northeast-2 --cluster=staging --approve

4. helm을 사용하여 AWS Load Balancer Controller 설치 
   
4-1. eks-charts 리포지토리 추가
    helm repo add eks https://aws.github.io/eks-charts

4-2. 최신 차트가 적용되도록 로컬 리포지토리를 업데이트
    helm repo update

4-3. aws 로드밸런서 컨트롤러 설치

    helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
    -n default \
    --set clusterName=staging \
    --set serviceAccount.create=false \
    --set serviceAccount.name=aws-load-balancer-controller 

    ( helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
    -n kube-system \
    --set clusterName=cluster-name \
    --set serviceAccount.create=false \
    --set serviceAccount.name=aws-load-balancer-controller )

4-4. 다음과 같이 떠야함
    > kubectl get deployment -n kube-system aws-load-balancer-controller
        NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
        aws-load-balancer-controller   2/2     2            2           47s

5. 서브넷 -> public, private 태그 추가
   
    키 - kubernetes.io/cluster/pj4-staging
    값 - shared 또는 owned

    (키 - kubernetes.io/cluster/cluster-name)
    (값 - shared 또는 owned)

5-1. 프라이빗 서브넷에 태그 추가

    키 - kubernetes.io/role/internal-elb
    값 - 1

5-2. 퍼블릭 서브넷에 태그 추가

    키 - kubernetes.io/role/elb
    값 - 1

6. 클러스터에 대한 IAM 사용자 및 역할 액세스 사용 설정 : https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/add-user-role.html

    curl -o eks-console-full-access.yaml https://amazon-eks.s3.us-west-2.amazonaws.com/docs/eks-console-full-access.yaml
    kubectl apply -f eks-console-full-access.yaml


<<<<<<< HEAD
=======
## 프로젝트 목표
일반적으로 container 배포 시 명령형 쉘스크립트로 deployment, service, ingress 들을 노출하는 것을 생각할 것입니다.
이번 프로젝트에서는 Argo CD 라는 DevOps tool을 사용하여 CI/CD 배포를 편하게 하여 배포과정에서의 Auto Scailing, 무중단배포의 핵심 tool이라고 생각합니다. 

## 개발환경

## SA
eksctl delete cluster --name pj4-staging --region ap-northeast-2
>>>>>>> be022a46536c6b0f558f3a2e4c8eaf089ec6bca0
