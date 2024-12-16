# Kserve로 모델 외부로 배포하기

## 개요
다음의 과정을 통해 모델 배포를 완료한다.

1. inferenceservice가 사용할 secret, serviceaccount 생성
2. kubeflow 대쉬보드 UI를 통해 inferenceservice 생성
3. rbac 에러를 방지하기 위해 istio의 차단 정책을 우회 (by. authorizationpolicy)
4. k8s 내부/외부에서 정상적으로 접근 가능한 모델 배포 완료


## 설치

### 참고
- [kserve mlflow](https://kserve.github.io/website/latest/modelserving/v1beta1/mlflow/v2/#deploy-with-inferenceservice)
- [kserve s3](https://kserve.github.io/website/latest/modelserving/storage/s3/s3/)
- [kserve 엔드포인트 rbac 문제 해결 개인블로그](https://kyeongseo.tistory.com/entry/kserve-%EC%82%AC%EC%9A%A9-%EB%B0%8F-%EC%84%A4%EC%A0%95-%EA%B0%80%EC%9D%B4%EB%93%9C) 

위의 페이지를 참고해 제작하였다.

### 0. 매직 DNS 생성

매직 dns 서비스를 적용하면 개발 및 테스트 환경에서 복잡한 DNS 구성 없이도 서비스에 쉽게 접근할 수 있다.
 

```
kubectl patch svc istio-ingressgateway -n istio-system -p '{"spec": {"type": "LoadBalancer"}}'

kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.15.2/serving-default-domain.yaml
 ```
위 명령어를 사용해 istio-ingressgateway service의 type을 LoadBalancer로 변경한다.

Knative Serving 환경에서 sslip.io를 기본 도메인을 설정하는 Job과 관련 Service를 적용한다.
 


 


### 1. inferenceservice가 사용할 secret, serviceaccount 생성

- inferenceservice는 minio에 저장된 mlflow 모델을 파드 생성시에 다운 받는다. 
- 이 떄, minio와의 통신을 위해 적합한 증명이 필요하게된다.이를 위해 secret, serviceaccount 생성한다.

- sa.yaml
`kubectl apply -f sa.yaml`

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
      storageUri: "s3://mlflow/1/d9a3dd7164bd423b9482ab8a77316c33/artifacts/iris_model"

```


정상적으로 배포되었다면 InferenceService ready되었다고 나온다. 이제 UI를 통해 모델의 내부/외부 엔드포인트를 확인해볼 수 있다.
- 확인
`kubectl get inferenceservice -A`

![사진](/9.trobleshooting-image/ks.png)



### 3. rbac 에러를 방지하기 위해 istio의 차단 정책을 우회 (by. authorizationpolicy)

kubeflow는 istio 기반으로한 k8s 서비스매쉬를 지향한다. istio는 각 구성요간의 통신을 더 세밀하게 조정할 수 있게 해준다. 기본으로 적용된 istio 정책은 폐쇠적인 정책을 가지고있다. 위에서 배포된 모델에 접근해보면 내부/외부에서 모두 rbac 에러가 발생한다. 

따라서, 우회 정책을 생성한다.

`차단정책`
1. istio 경로 차단 -> `ap_path.yaml` 으로 우회
2. istio의 oidc 프록시 인증 요구 -> `ap.yaml` 으로 우회
3. istio jwt 요구 -> `ap_jwt.yaml`으로 우회

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
        ## 이부분부터 추가한다.
        - /v1/*
        - /v2/*
        ## 여기까지 추가한다.
  selector:
    matchLabels:
      app: istio-ingressgateway
```
기본적으로 존재하는 istio-ingressgateway-require-jwt를 수정한다. 이는 v1,v2 models로 가는 요청에 jwt 검사를 제외한다.

기존의 istio-ingressgateway-require-jwt는 모델의 외부 엔드포인트에서 접근시 jwt 검사를 한다. 일반적인 환경이라면 다음 [kserve readme.md](https://github.com/kserve/kserve/blob/master/docs/samples/istio-dex/README.md)에 나온 방식으로 토큰을 발행할 수 있지만, eks, gcp 환경에선 jwt 발행자가 기본 발행자가 아닌 자체 발행자를 사용해 jwt 인증에 문제가 발생한다. (readme.md 트러블슈팅 참고)

따라서, 이 경우에는 jwt 검사를 우회하는 것으로 해결했다. 위의 두 정책이 적용되면 (inferenecservie 재시작할 필요x) 모델이 내부 외부에서 잘 접근이 가능한 것을 알 수 있다.

- ap.yaml
```
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: istio-ingressgateway-oauth2-proxy
  namespace: istio-system
spec:
  action: CUSTOM
  provider:
    name: oauth2-proxy
  selector:
    matchLabels:
      app: istio-ingressgateway
  rules:
  - to:
    - operation:
        notPaths: ["/v1*", "/v2*"]
```
모델에 요청을 보냈는데, 외부에서 요청시 dex 로그인을 요구하거나, 내부에서 요청시 404에러가 뜬다면 위의 AuthorizationPolicy가 필요하다.

oauth2-proxy에 우회경로를 설정해주어 모델 요청시 dex 로그인이 필요없게된다.



```
kubectl apply -f ap_path.yaml
kubectl apply -f ap.yaml
kubectl edit Authorizationpolicy istio-ingressgateway-require-jwt -n istio-system

```



### 4. k8s 내부/외부에서 정상적으로 접근 가능한 모델 배포 완료

inferenceservice를 생성할 때, 모델의 프로토콜을 v2로 하였기에 이 부분에 주의하여 요청을 날려보자. 

- 모델 요청 코드

[모델 요청](/7.kserve/send_request.ipynb)




