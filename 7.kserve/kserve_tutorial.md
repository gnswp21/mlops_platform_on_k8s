# Kserve로 모델 외부로 배포하기

## 개요
다음의 과정을 통해 모델 배포를 완료한다.

1. inferenceservice가 사용할 secret, serviceaccount 생성
2. kubeflow 대쉬보드 UI를 통해 inferenceservice 생성
3. rbac 에러를 방지하기 위해 istio의 차단 정책을 우회 (by. authorizationpolicy)
4. k8s 내부/외부에서 정상적으로 접근 가능한 모델 배포 완료


## 설치

### 참고
[kserve mlflow](https://kserve.github.io/website/latest/modelserving/v1beta1/mlflow/v2/#deploy-with-inferenceservice)
[kserve s3](https://kserve.github.io/website/latest/modelserving/storage/s3/s3/)

위의 두 페이지를 참고해 제작하였다.


### inferenceservice가 사용할 secret, serviceaccount 생성

- inferenceservice는 minio에 저장된 mlflow 모델을 파드 생성시에 다운 받는다. 
- 이 떄, minio와의 통신을 위해 적합한 증명이 필요하게된다.이를 위해 secret, serviceaccount 생성한다.

- sa.yaml
```
apiVersion: v1
kind: Secret
metadata:
  name: s3creds
  namespace: kubeflow-user-example-com
  annotations:
     serving.kserve.io/s3-endpoint: minio-service.kubeflow:9000 # replace with your s3 endpoint e.g minio-service.kubeflow:9000
     serving.kserve.io/s3-usehttps: "0" # by default 1, if testing with minio you can set to 0
     serving.kserve.io/s3-region: "us-east-1"
     serving.kserve.io/s3-useanoncredential: "false" # omitting this is the same as false, if true will ignore provided credential and use anonymous credentials
type: Opaque
stringData: # use `stringData` for raw credential string or `data` for base64 encoded string
  AWS_ACCESS_KEY_ID: minio
  AWS_SECRET_ACCESS_KEY: minio123

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sa
  namespace: kubeflow-user-example-com
secrets:
- name: s3creds

```

### 2. kubeflow 대쉬보드 UI를 통해 inferenceservice 생성
대쉬보드에 직접 yaml 파일을 붙여넣기 해도 되고 평범하게 apply로도 생성이 가능하다.

앞서 훈련한 irish 모델을 배포하는 서비스가 생성된다.` storageUri` 에 자신이 저장한 모델의 s3 uri를 작성한다. minio의 경우 아래와 같이 적용하면 된다. (자신의 환경에 맞게 조정하자.)

`버켓이름/1/런ID/artifacts/모델이름` 등의 형태이다.

- is_mlflow.yaml
```
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "irish-s3"
spec:
  predictor:
    serviceAccountName: sa
    model:
      modelFormat:
        name: mlflow
      protocolVersion: v2
      storageUri: "s3://mlflow/1/8b60da388c9547c3b6728a07aa794bf0/artifacts/iris_model"

```

정상적으로 배포되었다면 InferenceService ready되었다고 나온다. 이제 UI를 통해 모델의 내부/외부 엔드포인트를 확인해볼 수 있다.




### 3. rbac 에러를 방지하기 위해 istio의 차단 정책을 우회 (by. authorizationpolicy)

kubeflow는 istio 기반으로한 k8s 서비스매쉬를 지향한다. istio는 각 구성요서간의 통신을 더 세밀하게 조정할 수 있게 해준다. 기본으로 적용된 istio 정책은 폐쇠적인 정책을 가지고있다. 위에서 배포된 모델에 접근해보면 내부/외부에서 모두 rbac 에러가 발생한다. 

따라서, 우회 정책을 생성한다.

- ap_path.yaml

```
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allowlist-by-paths
  namespace: istio-system
spec:
  action: ALLOW
  rules:
  - to:
    - operation:
        paths:
        - /metrics
        - /healthz
        - /ready
        - /wait-for-drain
        - /v1/models/*
        - /v2/models/*
```
v1,v2 models로 가는 요청을 허가한다. 다시말해, 모델 요청을 허가해준다.



- ap_jwt.yaml

```
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"security.istio.io/v1beta1","kind":"AuthorizationPolicy","metadata":{"annotations":{},"name":"istio-ingressgateway-require-jwt","namespace":"istio-system"},"spec":{"action":"DENY","rules":[{"from":[{"source":{"notRequestPrincipals":["*"]}}],"to":[{"operation":{"notPaths":["/dex/*","/dex/**","/oauth2/*"]}}]}],"selector":{"matchLabels":{"app":"istio-ingressgateway"}}}}
  creationTimestamp: "2024-11-14T06:55:32Z"
  generation: 2
  name: istio-ingressgateway-require-jwt
  namespace: istio-system
  resourceVersion: "13253374"
  uid: 9b34e6bc-3808-49a1-bc3e-d12c3d16b5d1
spec:
  action: DENY
  rules:
  - from:
    - source:
        notRequestPrincipals:
        - '*'
    to:
    - operation:
        notPaths:
        - /dex/*
        - /dex/**
        - /oauth2/*
        - /v1/*
        - /v2/*
  selector:
    matchLabels:
      app: istio-ingressgateway
```
기본적으로 존재하는 istio-ingressgateway-require-jwt를 수정한다. 이는 v1,v2 models로 가는 요청에 jwt 검사를 제외한다.

기존의 istio-ingressgateway-require-jwt는 모델의 외부 엔드포인트에서 접근시 jwt 검사를 한다. 일반적인 환경이라면 다음 [kserve readme.md](https://github.com/kserve/kserve/blob/master/docs/samples/istio-dex/README.md)에 나온 방식으로 토큰을 발행할 수 있지만, eks, gcp 환경에선 jwt 발행자가 기본 발행자가 아닌 자체 발행자를 사용해 jwt 인증에 문제가 발생한다. (readme.md 트러블슈팅 참고)

따라서, 이 경우에는 jwt 검사를 우회하는 것으로 해결했다. 위의 두 정책이 적용되면 (inferenecservie 재시작할 필요x) 모델이 내부 외부에서 잘 접근이 가능한 것을 알 수 있다.

- 확인
`kubectl get inferenceservice -A`


### 4. k8s 내부/외부에서 정상적으로 접근 가능한 모델 배포 완료

inferenceservice를 생성할 때, 모델의 프로토콜을 v2로 하였기에 이 부분에 주의하여 요청을 날려보자. 

- 모델 요청 코드
```
```

![모델 배포 이미지]()




