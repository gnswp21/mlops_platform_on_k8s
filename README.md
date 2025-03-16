# mlops on k8s

## 개요
쿠버네티스 환경에서 작동하는 ml 파이프라인 구축
구축 과정에서 필요한 튜토리얼, 디버깅 과정을 공유

- 쿠브플로 아키텍쳐
![쿠브플로 이미지](/9.trobleshooting-image/kubeflow.png)
- 버전
  - kubeflow 1.9.1
  - mlflow 2.10.1



## 플랫폼이 제공하는 기능

1. GKE 상에 구축한 kubeflow
2. 메인 대쉬보드를 통해 통합적인 파이프라인 관리 가능
    - 덱스의 이메일 기반 Authetication과 Authroization  (W. 프로필 기반의 RBAC )
3. 커스텀 노트북 이미지를 통한 대화형 셀을 통한 모델 개발 및 데이터 EDA (w. Jupyter) 

5. 실험 추적 및 모델 레지스트리 제공 (W. mlflow)
6. 외부에 노출된 모델 서빙. 모델 IP 제공 (W. KServe)

## 데모 영상

<video width="640" height="360" controls>
    <source src="./9.trobleshooting-image/kubeflow_demo.mp4" type="video/mp4">
    Your browser does not support the video tag.
</video>




https://github.com/user-attachments/assets/e4c6fadd-81e0-4cf5-838d-b58c3c2a1d41






## 빌드
### 준비사항
- kubectl 설치
- helm 설치
- kustomize
- 로드밸런서 혹은 인그레서 포트포워딩을 통해 외부로 노출된 k8s 클러스터
- gcp 라면 gcloud 설치

### 빌드 튜토리얼

- [GKE 설치](/0.build_gke/0.build_gke.md)
- [kubeflow 설치](/1.build_kubeflow/build.md)

위의 두 과정을 통해 기본적인 kubeflow 사용이 가능해진다.

### 운영 튜토리얼
- [커스텀 노트북 설치](/2.notebook_custom_image/notebook_tutorial.md)
- [덱스 새로운 스태틱 유저 추가](/3.dex_add_new_user/3.dex_add_new_user_tutorial.md)
- [새 프로필 생성](/4.add_new_profile/4.add_new_profile.md)
- [mlflow 결합하여 실험추적 및 모델 레지스티리](/8.mlflow/mlflow_tutorial.md)
- [KServe로 모델 외부 포인트에 노출](/7.kserve/kserve_tutorial.md)




##  트러블슈팅
빌드 및 운영과정에서 발생한 다양한 트러블슈팅을 공유합니다.



### 1. kubeflow 빌드 | 파드 생성 | 해결

- 요약 : default storageclass를 설정하지 않아 발생한 문제(문서 요구사항에 있음에도 불구하고).

- 해결

1. pvc in eks -> storageclass gp2로 설정
   
정상 운영이 안되는 파드들이 몇 개 있었는데, describe로 보니 pvc가 바운드가 안되어 있었다.
pvc를 descrive 해보니 볼륨도 설정이 안되어있었다. 여기서 상당히 막혔다. 찾아보니까 비슷한 문제들이 있었다.
(https://github.com/kubeflow/kubeflow/issues/3481) 해결 방법은 pvc에 대응하는 pv를 지정해주면 된다고 한다.
난 eks를 쓰니까 pvc에 stroageClass를 지정했다. 이를 위해 기존의 pvc를 `kubectl get pvc katib-mysql -n kubeflow -o yaml > katib-mysql-backup.yaml` 와 같은 방식으로 yaml 파일로 다시 만들고 spec에 `storageClassName: gp2` 으로 추가했더니 제대로 pvc가 바운드되었다. 

이렇게 pvc 바운드가 정상적이게 되니 파드들이 하나둘씩 재실행되면서 정상이 되었다.

2. gp2를 기본 프로비저너로 설정하기
   
이제 노트북을 생성해보려니까 파드가 kubeflow-user-example-com 네임스페이스에서 생생되다가 펜딩된다. 똑같은 pvc 문제였다.
이번엔 gp2를 기본 프로비저너로 설정할 수 있는지 확인해보고 이를 적용시켰다
`kubectl patch storageclass gp2 --patch '{\"metadata\": {\"annotations\": {\"storageclass.kubernetes.io/is-default-class\": \"true\"}}}'`

이제 정상적으로 파드가 생성되었다. 앞서 요약했듯이 문서 요구사항에 적힌대로 default storageclass가 설정되어 있어야 했다.


---
### 2. kubeflow 빌드 | 파드 생성 2 | 해결
그 후 새로운 파드를 생성하니 pvc가 문제없이 생성되고 노트북도 잘 만들어졌다. 다만 cpu가 간당간당해 보인다. 0.5 코어 할당했더니 cpu 부족으로 생성이 안됐다.

---
### 3. kubeflow 빌드 | web UI | 미해결?

1. 모든 파드가 정상적인데 웹UI에 연결이 안되는 포트 포워딩의 문제
   1. 포워딩 된 도커 컨테이너에서 8080:80 으로 서비스 포워딩하고 외부에서 접속하려고 했는데 안됐다.
   2. 로컬 머신 터미널에서 포워딩하고 접근하니까 안됐다 ->
        - localhost:8080으로 접근하니까 안됐다. -> 127.0.0.1:8080으로 접속하니까 됐다. 왜일까? 이유는 모르지만 재부팅후에는 정상적으로 된듯하니 내 컴퓨터의 네트워크 설정에 문제가 있는듯하다. 다시 같은 문제가 발생하지 않아 일단 넘어갔다.

---
### 4. dex 운영 | 새 유저 추가 | 해결

새로운 유저를 만들어 로그인해보고 싶었지만 잘 안되었다.
-> dex 에 유저를 추가해야한다는것을 알아내었다.
-> dex에 유저 추가하는 방법은 하드하게 추가하기와 OIDC (구글) 등의 방법이 있는 것 같은데
하드하게 추가하는 방법은 잘 안됨 일단 유저 추가하기는 보류

이후에 두가지 과정을 통해 새로운 스태틱 유저를 추가하는 방법을 알아내었다.
1. [덱스 새로운 스태틱 유저 추가](/3.dex_add_new_user/3.dex_add_new_user_tutorial.md)
2. [새 프로필 생성](/4.add_new_profile/4.add_new_profile.md)


---
### 5. 노트북 운영 | 노트북의 저장된 패키지 의존성 불일치 | 해결

요약 
- 원인 : 패키지 의존성 불일치
- 해결 : 새 패키지 버전을 설치한 커스텀 노트북 이미지 생성
  - [커스텀 노트북 설치](/2.notebook_custom_image/notebook_tutorial.md) 

문제

1. numpy.nd array 문제
파이토치로 모델 학습하려는데 nd.array문제 발생. 검색해보니 numpy 버전 문제임. 파드에 exec로 접속해 numpy 삭제 후 버전 맞춰서 재설치
잘됨, 모델 학습 성공

1. custom image 문제
위의 파드 변경점은 파드 삭제시 초기화됨(컨테이너 기반이므로 당연히) kubeflow의 docs에 커스텀 이미지를 만드는 방법이 소개되어있음.
도커 빌드를 make를 이용해 자동화하는데 처음 보는 거라 처음에 읽기 어려웠다. 그래도 보다보니 환경변수를 설정하고 그거에 따라 통합적으로 빌드하는 시스템인것을 확인. 핵심은 kubeflow측에서 도커허브 저장한 이미지를 FROM으로 넣고 내 이미지를 제작하면 간단하게 만들 수 있었음. 그 외에도 아예 커스텀하게 만드는 방법도 있는듯 하지만 이는 s6 overlay에 대해 공부해야할듯해보여서 지금은 넘어갈 계획

---
### 6. mlflow 운영 | UI 로그인 불가능 | 해결
*상황*
- k8s상에서 mlflow를 사용하기 위해 bitnami가 관린하는 mlflow 사용
- mlflow UI 접근 시 로그인/패스워드를 제출해야하는데, bitnami 리드미에 적힌 user/"" 가 적용되지 않는 상황.

![이미지](/9.trobleshooting-image/bitnami_mlflow_admin.png)

*해결*
- 아래 블로그를 참고해 k8s 관리자는 트래킹 서버 UI의 비밀번호를 추출할 수 있음을 알게되어서 적용
```
ID : user
PASSWORD : kubectl get secret --namespace mlflow mlflow-tracking -o jsonpath="{.data.admin-password }" | base64 -d

(참고: https://suwani.tistory.com/169)

```
---
### 7. kserve 운영 | inferenceservice 생성 중 펜딩 | 해결
*상황*
- inferenceservice 생성 중 펜딩
- 에러 메세지:` RevisionFailed: Revision "iris-svm-v2151-predictor-00001" failed with message: Initial scale was never achieved.`

*해결*
- inferenceservice가 minio에서 모델을 받지 못 하는 상황
- inferenceservice가에 작성된 모델의 주소가 틀렸거나, 적합한 보안증명이 없기 떄문에 발생
- secret이 적용된 serviceaccount를 생성하고 inferenceservice에 추가해주었다.

*과정*

디버깅 코드

```
kubectl get pod -n kubeflow-user-example-com
kubectl describe pod irish-s3-predictor-00001-deployment-649f846f8b-vcqw9  -n kubeflow-user-example-com
```
파드가 멈춘 것을 확인 이후 파드의 로그를 확인해보니 보안증명이 없기에 발생한 문제임을 확인했다.

*참고*
[minio 보안증명](https://kserve.github.io/website/0.7/modelserving/storage/s3/s3/#attach-secret-to-a-service-account)


---
### 8. kserve 운영 | inferenceservice의 rbac 문제 | 해결

*상황*
- 엔드포인트에 접근해 요청을 날렸지만, 권한 없다는 응답만 나오는 상황

*해결*
- 엔드포인트 url만 적고, v1 ~ predict 부분을 안적음
- `http://${INGRESS_HOST}:${INGRESS_PORT}/v2/models/mlflow-v2-wine-classifier/infer` 이 예시와 같이 /v1 부분을 추가로 작성해야 한다.
 
*참고*
- [s3](https://kserve.github.io/website/0.7/modelserving/storage/s3/s3/#attach-secret-to-a-service-account)
- [mlflow 엔드포인트 요청 형식](https://kserve.github.io/website/latest/modelserving/v1beta1/mlflow/v2/#testing-deployed-model)


---
### 9. kserve 운영 | inferenceservice의 Jwt issuer is not configured | 해결

*상황*
- 외부 엔드포인트에 접근해 요청을 날렸지만, 여전히 rbac 문제
- 내부 엔드포인트는 정상적인 접근이 가능함

*해결*
- istio가 jwt를 검사하는 과정에서 발생한 문제
- 일반적으로 요청에 jwt 인증 토큰을 담는 것으로 해결할 수 있다.
- 단, kserve_tutorial.md에 언급했듯이 이 프로젝트의 gke기반이고 jwt의 발행자(issuer)가 kubeflow가 상정한 기본값과 다르다. 여기서 발행자 불일치가 발생해 요청에 토큰을 담아도 여전히 해당 이슈가 발생한다.
- 따라서, istio jwt 검사 정책을 수정해 우회 경로를 만든다. (ap_jwt.yaml)
- 이를 통해 정상적인 접근이 가능했다. 단, 외부의 접근에 jwt를 검사하지 않아 요청에 대한 보안은 낮아졌다.

*팁*
- 내 이슈어 확인

`kubectl get --raw /.well-known/openid-configuration`

```
{"issuer":"https://container.googleapis.com/v1/projects/
nomadic-asset-441010-q1/locations/asia-northeast3/clusters/cluster-1",
"jwks_uri":"https://10.178.0.2:443/openid/v1/jwks",
"response_types_supported":["id_token"],"subject_types_supported":["public"],
"id_token_signing_alg_values_supported":["RS256"]}
```
  
 
*참고*
- [jwt 인증 kserve 깃허브 공식 문서](https://github.com/kserve/kserve/tree/master/docs/samples/istio-dex)

- [Jwt issuer is not configured 논의 깃허브](https://github.com/kubeflow/manifests/issues/2850)

- [kserve 엔드포인트 rbac 문제 해결 개인블로그](https://kyeongseo.tistory.com/entry/kserve-%EC%82%AC%EC%9A%A9-%EB%B0%8F-%EC%84%A4%EC%A0%95-%EA%B0%80%EC%9D%B4%EB%93%9C) 


---
### 10. kserve 운영 | inferenceservice의 method not allowed | 해결

*상황*
- mlflow 엔드포인트 생성 후 요청 시에 method not allowed 에러 발생
  

*해결*
- 예제의 v1 protocal 과  mlflow의 v2 protocal의 형식이 달라서 생긴 문제
- URL과 input의 형식을 수정하니 잘 됐다.

*참고*

- [mlflow 엔드포인트 요청 형식](https://kserve.github.io/website/latest/modelserving/v1beta1/mlflow/v2/#testing-deployed-model)

---
### 11. kserve 운영 | inferenceservice의 403, 404 | 해결

*상황*
- 재설치한 kserve에 외부에서 외부엔드포인트로 요청시 403 에러가 발생
- 재설치한 kserve에 내부에서 외부엔드포인트로 요청시 404 에러가 발생
  

*해결*
- istio의 oidc 프록시 우회 경로가 설정되어 있지 않기 떄문에 발생했다.
- `ap.yaml` AuthorizationPolicy를 추가해 해결할 수 있었다.

*참고*

- [kserve 엔드포인트 rbac 문제 해결 개인블로그](https://kyeongseo.tistory.com/entry/kserve-%EC%82%AC%EC%9A%A9-%EB%B0%8F-%EC%84%A4%EC%A0%95-%EA%B0%80%EC%9D%B4%EB%93%9C) 


---
## 추후 추가할 기능
1. 프로메테우스 스택추가로 파드 및 노드 메트릭 모니터링단 추가
2. KFP 도입으로 모델 학습 및 배포의 자동화 
   1. 모델 드리프트 감지 확인을 위한 모델 모니터링
3. GPU 학습 (W. 트레이닝 오퍼레이터)
4. 분산 데이터 처리 (W. Spark 오퍼레이터)



