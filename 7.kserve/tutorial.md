kubectl port-forward svc/istio-ingressgateway -n istio-system 8080:80



문제상황 : RevisionFailed: Revision "iris-svm-v2151-predictor-00001" failed with message: Initial scale was never achieved.

인퍼런스 서비스를 만들었지만 안되는 상황



디버깅 코드

```
kubectl get pod -n kubeflow-user-example-com
kubectl describe pod irish-s3-predictor-00001-deployment-649f846f8b-vcqw9  -n kubeflow-user-example-com
kubectl get serviceaccount -n kubeflow-user-example-com
```

빌드 코드
```
kubectl apply -f sa.yaml
```


https://kserve.github.io/website/0.7/modelserving/storage/s3/s3/#attach-secret-to-a-service-account


https://github.com/kserve/kserve/tree/master/docs/samples/istio-dex

처음에 왜 안됐지?




# 참고 블로그
- mlflow 엔드포인트 생성
  https://kserve.github.io/website/0.7/modelserving/storage/s3/s3/#attach-secret-to-a-service-account

- 


- kserve 엔드포인트 외부로 노출
https://kyeongseo.tistory.com/entry/kserve-%EC%82%AC%EC%9A%A9-%EB%B0%8F-%EC%84%A4%EC%A0%95-%EA%B0%80%EC%9D%B4%EB%93%9C



mlops 서빙엔드포인트는 여전히 rbac가 나온는 상황


kubeflow docs for kserve
https://github.com/kubeflow/manifests/tree/master/contrib/kserve


kserve의 모델 메쉬가 뭐야?

ChatGPT의 말:
ChatGPT
KServe의 모델 메쉬(Model Mesh)는 다수의 머신러닝 모델을 Kubernetes 환경에서 효율적으로 배포하고 관리하기 위한 기능입니다. Model Mesh는 특히 대규모 모델 배포 환경에서 효율적인 리소스 사용과 자동 스케일링을 가능하게 해줍니다.

주요 기능
다중 모델 배포: 모델 메쉬는 동일한 인퍼런스 서버에서 여러 개의 모델을 동시 관리할 수 있도록 해줍니다. 모델을 각각 별도의 서버에 할당하지 않고, 요청에 따라 메모리 안팎으로 로드하여 리소스를 절약합니다.

자동 스케일링: KServe 모델 메쉬는 모델을 메모리 내에 필요한 만큼만 로드하고, 트래픽 패턴에 따라 스케일 업/다운하여 리소스 낭비를 줄입니다. 이를 통해 요청이 많을 때는 모델 인스턴스를 늘리고, 요청이 적을 때는 리소스를 최소화하여 비용 절감이 가능합니다.

온디맨드 모델 로드: 모델 메쉬는 모든 모델을 메모리에 항상 로드하는 것이 아니라, 요청이 있을 때만 해당 모델을 로드합니다. 이로 인해 한정된 메모리 내에서도 많은 모델을 효과적으로 관리할 수 있습니다.

멀티 프레임워크 지원: TensorFlow, PyTorch, ONNX 등 다양한 모델을 지원하여 다양한 ML 프레임워크와 함께 사용할 수 있습니다.

통합 관리 및 모니터링: Model Mesh는 KServe와 통합된 환경에서 중앙 집중형 관리 및 모니터링이 가능해, 클러스터에서 실행되는 모든 모델을 쉽게 관리하고 추적할 수 있습니다.


kubectl port-forward svc/istio-ingressgateway -n istio-system 8080:80



문제상황 : RevisionFailed: Revision "iris-svm-v2151-predictor-00001" failed with message: Initial scale was never achieved.

인퍼런스 서비스를 만들었지만 안되는 상황



디버깅 코드

```
kubectl get pod -n kubeflow-user-example-com
kubectl describe pod irish-s3-predictor-00001-deployment-649f846f8b-vcqw9  -n kubeflow-user-example-com
kubectl get serviceaccount -n kubeflow-user-example-com
```

빌드 코드
```
kubectl apply -f sa.yaml
```


https://kserve.github.io/website/0.7/modelserving/storage/s3/s3/#attach-secret-to-a-service-account


https://github.com/kserve/kserve/tree/master/docs/samples/istio-dex

처음에 왜 안됐지?




# 참고 블로그
- mlflow 엔드포인트 생성
  https://kserve.github.io/website/0.7/modelserving/storage/s3/s3/#attach-secret-to-a-service-account

- 


- kserve 엔드포인트 외부로 노출
https://kyeongseo.tistory.com/entry/kserve-%EC%82%AC%EC%9A%A9-%EB%B0%8F-%EC%84%A4%EC%A0%95-%EA%B0%80%EC%9D%B4%EB%93%9C



## mlops 서빙엔드포인트는 여전히 rbac가 나온는 상황


- host : (https://kserve.github.io/website/0.7/modelserving/storage/s3/s3/#deploy-the-model-on-s3-with-inferenceservice)



- kserve rbac 엔드포인트 url만 적고, v1 ~ predict 부분을 안적음


##  `Jwt issuer is not configured ` 라고 뜨면서 external 전송이 안되던 상황

header의 HOST를 고쳐주니까 해결되었다.

이제 보니까 다른 inferenceservice에 통신하니까 된 것이다.
```
kubectl get inferenceservice -A
NAMESPACE                   NAME           URL                                                                    READY   PREV   LATEST   PREVROLLEDOUTREVISION   LATESTREADYREVISION            AGE
kserve-test                 sklearn-iris   http://sklearn-iris.kserve-test.34.47.121.117.sslip.io                 True           100                              sklearn-iris-predictor-00001   4d2h
kubeflow-user-example-com   sklearn-iris   http://sklearn-iris.kubeflow-user-example-com.34.47.121.117.sslip.io   True           100                              sklearn-iris-predictor-00001   85m
```

재발급을한 토큰을 사용해서 kserve-test에 존재하는 IS에는 예측이 잘 간다.
-> 토큰의 문제는 아니다.

근데 URL과 HOST가 일치하지 않는데..? 
-> 만들 때 is 네임을 똑같이 설정해서 생성되는 외부 url도 똑같았다.

HOST 명
sklearn-iris.kubeflow-user-example-com.34.47.121.117.sslip.io -> 에러발생
vs
sklearn-iris.kubeflow-user-example-com.svc.cluster.local -> 정상작동

sklearn-iris.kubeflow-user-example-com.svc.cluster.local 은 내부 클러스터에서만 사용 가능한 주소로 JWT 검사가 무시된다. 실제로 JWT 토큰을 뺴도 정상적으로 예측이 됐다. 
다음 트러블슈팅에서 계속

## 외부 IP(sklearn-iris.kubeflow-user-example-com.34.47.121.117.sslip.io)을 사용하고 에러가 발생하지 않도록 수정해보자

세가지 방법이 있는듯 하다.

O


아무래도 gke 사용해서 다음 깃헙에 나온 문제가 발생한 것 같다.
https://github.com/kubeflow/manifests/issues/2850

아래의 공식문서도 참고해보자
https://github.com/kubeflow/manifests/tree/master/common/oauth2-proxy


1. OIDC 우회정책
다음 블로그를 참고해 (https://kyeongseo.tistory.com/entry/kserve-%EC%82%AC%EC%9A%A9-%EB%B0%8F-%EC%84%A4%EC%A0%95-%EA%B0%80%EC%9D%B4%EB%93%9C)
아래의 우회정책을 사용해봤다.

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
        notPaths: ["/v1/*", "/v2/*"]
```
v1,v2로 향하는 호출에는 oidc 노출을 우회하는건데 여전히 RBAC 에러가 발생

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
        notPaths: ["/v1/*", "/v2/*", "*"]
```
로 고쳐보니 이젠 UI 전체에 RBAC 에러가 발생했다. 왜 그럴까?

위의 정책이 모든 요청에 OIDC 요청을 오히려 처리하게끔 하는 걸까?


not path에 /v3로 해보니 dex 로그인하라는 요청이 나온다.


- 다시 원래 설정을 돌아가서 로그를 확인해보자
  





- 내 이슈어 확인

```
kubectl get --raw /.well-known/openid-configuration
{"issuer":"https://container.googleapis.com/v1/projects/nomadic-asset-441010-q1/locations/asia-northeast3/clusters/cluster-1","jwks_uri":"https://10.178.0.2:443/openid/v1/jwks","response_types_supported":["id_token"],"subject_types_supported":["public"],"id_token_signing_alg_values_supported":["RS256"]}
```


- istio-ingressgateway-require-jwt 를 수정하는 것으로 jwt 인증 우회로 해결

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

위의 설정으로 jwt 우회에 성공 근본적으로 jwt issuer가 혼돈된 상태이지만 현재 진행중인 이슈인것 같으므로 일단 패스
jwt 우회 설정을 사용하기로 결정

- mlflow 엔드포인트 생성 후 다시 시도했지만 method not allowed 에러 발생
예제의 v1 protocal 과  mlflow의 v2 protocal의 형식이 달라서 생긴 문제
URL과 input의 형식을 수정하니 잘 됐다.




