# mlops on k8s

## 개요
쿠버네티스 환경에서 작동하는 ml 파이프라인 구축

- kubeflow와 mlflow를 결합한 형태로 사용

## 기능

1. GKE 상에 구축한 kubeflow
2. 메인 대쉬보드를 통해 통합적인 파이프라인 관리 가능
    - 덱스의 이메일 기반 Authetication과 Authroization  (W. 프로필 기반의 RBAC )
3. 커스텀 노트북 이미지를 통한 대화형 셀을 통한 모델 개발 및 데이터 EDA (w. Jupyter) 
4. 분산 데이터 처리 (W. Spark 오퍼레이터)
5. 실험 추적 및 모델 레지스트리 제공 (W. mlflow)
6. 외부에 노출된 모델 서빙. 모델 IP 제공 (W. KServe)

## 데모 영상







## 빌드 방법
### 준비사항
- kubectl 설치

### 빌드 튜토리얼

- [GKE 설치](/0.build_gke/0.build_gke.md)
- [kubeflow 설치](/1.build_kubeflow/build.md)

위의 두 과정을 통해 기본적인 kubeflow 사용이 가능해진다.

### 운영 튜토리얼
- [커스텀 노트북 설치]


##  트러블슈팅
빌드 및 운영과정에서 발생한 다양한 트러블슈팅을 공유합니다.



eks에 kubeflow 설치 성공
pv가 bound 된 상태로 남아서 안됐다
eks라서 안되는 줄알고
1. kubeflow on eks로 바꿧지만 AUTH로 사용하는 기술?(암튼 kubeflow on aws 사용되는 기술 스택 중 일부가) 스택이 이미 운영 안하는 회사의것이라서 안됐다
<- 업데이트 멈춤  (https://github.com/kubeflow/manifests/issues/2852)
근데 kubeflow/manifest도 eks에 잘된다 그래서 다시 eks에서 manifest 하는걸로 변경했다.

2. pvc in eks -> storageclass gp2로 설정
정상 운영이 안되는 파드들이 몇 개 있었는데, describe로 보니 pvc가 바운드가 안되어 있었다.
pvc를 descrive 해보니 볼륨도 설정이 안되어있었다. 여기서 상당히 막혔다. 찾아보니까 비슷한 문제들이 있었다.
(https://github.com/kubeflow/kubeflow/issues/3481) 해결 방법은 pvc에 대응하는 pv를 지정해주면 된다고 한다.
난 eks를 쓰니까 pvc에 stroageClass를 지정했다. 이를 위해 기존의 pvc를 `kubectl get pvc katib-mysql -n kubeflow -o yaml > katib-mysql-backup.yaml` 와 같은 방식으로 yaml 파일로 다시 만들고 spec에 `storageClassName: gp2  # storageClassName 필드를 spec 내부로 이동` 으로 추가했더니 제대로 pvc가 바운드되었다. 이 부분은 다시 처음부터 해보고 정리해볼 생각

이렇게 pvc 바운드가 정상적이게 되니 파드들이 하나둘씩 재실행되면서 정상이 되었다.

3. gp2를 기본 프로비저너로 설정하기
이제 신나서 노트북을 생성해보려니까 파드가 kubeflow-user-example-com 네임스페이스에서 생생되다가 펜딩된다. 똑같은 pvc 문제였다.
이번엔 gp2를 기본 프로비저너?로 설정할 수 있는지 확인해보고 이를 적용시켰다
kubectl patch storageclass gp2 --patch '{\"metadata\": {\"annotations\": {\"storageclass.kubernetes.io/is-default-class\": \"true\"}}}'
그 후 새로운 파드를 생성하니 pvc가 문제없이 생성되고 노트북도 잘 만들어졌다. 다만 cpu가 간당간당해 보인다. 0.5 코어 할당했더니 cpu 부족으로 생성이 안됐다.

오늘은 기술의 적용에 급급했으니 내일 다시 설정해보고 오늘 사용한 기술들을 공부해서 정리할것
세상에 요구조건에 default storageclass를 가진 쿠버네티스여야한다고 써있엇다!
젠장



4. 모든 파드가 정상적인데 웹UI에 연결이 안되는 포트 포워딩의 문제
- 1. 포워딩 된 도커 컨테이너에서 8080:80 으로 서비스 포워딩하고 외부에서 접속하려고 했는데 안됐다.
    2. 로컬 머신 터미널에서 포워딩하고 접근하니까 안됐다 ->
        - localhost:8080으로 접근하니까 안됐다. -> 127.0.0.1:8080으로 접속하니까 됐다. 왜일까?


5. 유저추가하기
새로운 유저를 만들어 로그인해보고 싶었지만 잘 안됨
-> dex 에 유저를 추가해야한다는것을 알아냄
-> dex에 유저 추가하는 방법은 하드하게 추가하기와 OIDC (구글) 등의 방법이 있는 것 같은데
하드하게 추가하는 방법은 잘 안됨 일단 유저 추가하기는 보류

6. numpy.nd array 문제
파이토치로 모델 학습하려는데 nd.array문제 발생. 검색해보니 numpy 버전 문제임. 파드에 exec로 접속해 numpy 삭제 후 버전 맞춰서 재설치
잘됨, 모델 학습 성공

7. custom image 문제
위의 파드 변경점은 파드 삭제시 초기화됨(컨테이너 기반이므로 당연히) kubeflow의 docs에 커스텀 이미지를 만드는 방법이 소개되어있음.
도커 빌드를 make를 이용해 자동화하는데 처음 보는 거라 처음에 읽기 어려웠음. 그래도 보다보니 환경변수를 설정하고 그거에 따라 통합적으로 빌드하는 시스템인것을 확인. 핵심은 kubeflow측에서 도커허브 저장한 이미지를 FROM으로 넣고 내 이미지를 제작하면 간단하게 만들 수 있었음. 그 외에도 아예 커스텀하게 만드는 방법도 있는듯 하지만 이는 s6 overlay에 대해 공부해야할듯해보여서 지금은 넘어갈 계획


8. dex static new 유저
dex에 새 static 유저를 추가하는 과정에서, hashFromEnv라는 항목이 있는데, 이 ENV가 어느 환경 기준인지 확인이 어려웠다. 덱스 파드로 exec 해보려했지만 bash가 없었다. 

해결 -> manifest/common/dex ..를 들어가보니 secret에 사용된 환경변수가 있는걸 확인했다. 여기에 추가할 수 있겠다.



## 추후 추가할 기능
1. 프로메테우스 스택추가로 파드 및 노드 메트릭 모니터링단 추가
2. KFP 도입으로 모델 학습 및 배포의 자동화 
   1. 모델 드리프트 감지 확인을 위한 모델 모니터링
3. GPU 학습
