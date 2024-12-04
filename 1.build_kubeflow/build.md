# build kubeflow
공식문서(https://github.com/kubeflow/manifests)를 바탕으로 작성

`ver kubeflow 1.9.1`

요구사항 (문서)
- 16코어 32GB메모리
- 50GB의 저장공간
- kbind |  default stroageclass가 설정된 k8s 클러스터

요구사항 (경험)
- 단지 구성이 목적이라면 8코어에도 아슬아슬하게 설치가능(쿠브플로 파이프라인이 코어를 많이 요구하므로 제외한다면 더 낮은 코어에서도 설치가능)
- kbind외의 클라우드, 온프레미스 등 모든 k8s에 manifest를 통해 설치 가능
- 일반 k8s를 사용한다면 default stroageclass가 설정되어있는것이 중요 (pvc가 bound 안되면 이 부분이 문제 -> eks는 default storageclass 설정이 안되어 있으므로 주의)



## 0. kubeflow 깃허브 클론


```
#git clone https://github.com/kubeflow/manifests.git --branch 1.9.1
cd manifest
```


## 1. docker login 정보 제공
kubeflow 공식문서에 따라, 도커 이미지 다운을 위한 로그인 정보를 k8s 클러스터에 제공한다.

- linux 환경의 경우 docker login을 함으로써 로그인정보가 담긴 `dockerconfigjson` 파일이 생성된다.
- 도커 데스크톱을 사용하는 환경에서는 `dockerconfigjson` 파일에 로그인 정보가 담기지 않는다. (로그인을 앱을 통해 관되되므로) 따라서, 해당 파일을 직접 써서 제공하였다.
  

`temp.json`
```
{
    "auths": {
        "https://index.docker.io/v1/": {
            "auth": "BASE64(ID:PASSWORD)"
        }
    }
}

위의 파일을 예시로 auth의 value 값에 BASE64로 인코딩된 실제로 도커 허브 로그인에 사용되는 ID:PASSWORD를 지정해주었다.


```

### sercret 생성

```
 kubectl create secret generic regcred \
    --from-file=.dockerconfigjson=temp.json \
    --type=kubernetes.io/dockerconfigjson
```



## 2. apply
설치하려는 구성요소에 따라 두가지 선택지 제시됨



```
cd manifests
while ! kustomize build example | kubectl apply --server-side --force-conflicts -f -; do echo "Retrying to apply resources"; sleep 20; done
```
위의 코드로 간단하게 모든 kubeflow 요소를 설치한다.

### specific kubeflow
- [kubeflow component 선택적 설치](/1.build_kubeflow/build_op.md)
  - 이 방식은 kubeflow의 요소를 하나씩 설치하는 방법으로 일부 요소만 선택적으로 설치하고 싶을 때 사용하자.


# 생성 확인
kubectl get pods -n cert-manager
kubectl get pods -n istio-system
kubectl get pods -n auth
kubectl get pods -n knative-eventing
kubectl get pods -n knative-serving
kubectl get pods -n kubeflow
kubectl get pods -n kubeflow-user-example-com

# 클러스터 외부 노출
## 인그레스 포트포워딩
`kubectl port-forward svc/istio-ingressgateway -n istio-system 8080:80`

## 로드밸런서 사용

`kubectl patch svc istio-ingressgateway -n istio-system -p '{"spec": {"type": "LoadBalancer"}}'`

위 두가지 방법 중 하나로 istio를 외부로 노출시키자


## dex UI 로그인 정보
- 기본유저 아이디 : user@example.com 비밀번호 : 12341234

# 삭제
kubectl delete namespace cert-manager
kubectl delete namespace istio-system
kubectl delete namespace auth
kubectl delete namespace knative-eventing
kubectl delete namespace knative-serving
kubectl delete namespace kubeflow
kubectl delete namespace kubeflow-user-example-com

