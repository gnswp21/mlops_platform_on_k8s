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
git clone https://github.com/kubeflow/manifests.git --branch 1.9.1
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
(powershell)
```
 kubectl create secret generic regcred `
    --from-file=.dockerconfigjson=$HOME\Projects\mlops\mlops_platform_on_k8s\1.build_kubeflow\temp.json `
    --type=kubernetes.io/dockerconfigjson
```



## 2 apply
설치하려는 구성요소에 따라 두가지 선택지 제시됨
## all kubeflow


### linux
```
while ! kustomize build example | kubectl apply --server-side --force-conflicts -f -; do echo "Retrying to apply resources"; sleep 20; done
```

## specific kubeflow
### must install
- cert manager

```
kustomize build common/cert-manager/cert-manager/base | kubectl apply -f -
kustomize build common/cert-manager/kubeflow-issuer/base | kubectl apply -f -
echo "Waiting for cert-manager to be ready ..."
kubectl wait --for=condition=ready pod -l 'app in (cert-manager,webhook)' --timeout=180s -n cert-manager
kubectl wait --for=jsonpath='{.subsets[0].addresses[0].targetRef.kind}'=Pod endpoints -l 'app in (cert-manager,webhook)' --timeout=180s -n cert-manager

```

- istio
```
echo "Installing Istio configured with external authorization..."
kustomize build common/istio-1-23/istio-crds/base | kubectl apply -f -
kustomize build common/istio-1-23/istio-namespace/base | kubectl apply -f -
kustomize build common/istio-1-23/istio-install/overlays/oauth2-proxy | kubectl apply -f -

echo "Waiting for all Istio Pods to become ready..."
kubectl wait --for=condition=Ready pods --all -n istio-system --timeout 300s
```


- Oauth2-proxy
```
echo "Installing oauth2-proxy..."

# Only uncomment ONE of the following overlays, they are mutually exclusive,
# see `common/oauth2-proxy/overlays/` for more options.

# OPTION 1: works on most clusters, does NOT allow K8s service account 
#           tokens to be used from outside the cluster via the Istio ingress-gateway.
#
kustomize build common/oauth2-proxy/overlays/m2m-dex-only/ | kubectl apply -f -
kubectl wait --for=condition=ready pod -l 'app.kubernetes.io/name=oauth2-proxy' --timeout=180s -n oauth2-proxy

# Option 2: works on Kind/K3D and other clusters with the proper configuration, and allows K8s service account tokens to be used
#           from outside the cluster via the Istio ingress-gateway. For example for automation with github actions.
# 
#kustomize build common/oauth2-proxy/overlays/m2m-dex-and-kind/ | kubectl apply -f -
#kubectl wait --for=condition=ready pod -l 'app.kubernetes.io/name=oauth2-proxy' --timeout=180s -n oauth2-proxy
#kubectl wait --for=condition=ready pod -l 'app.kubernetes.io/name=cluster-jwks-proxy' --timeout=180s -n istio-system
```

- dex
```
echo "Installing Dex..."
kustomize build common/dex/overlays/oauth2-proxy | kubectl apply -f -
kubectl wait --for=condition=ready pods --all --timeout=180s -n auth
```


- namespace
```
kustomize build common/kubeflow-namespace/base | kubectl apply -f -
```

- network policy
```
kustomize build common/networkpolicies/base | kubectl apply -f -
```

- kubeflow roles
```
kustomize build common/kubeflow-roles/base | kubectl apply -f -
```

- Kubeflow Istio Resources
```
kustomize build common/istio-1-23/kubeflow-istio-resources/base | kubectl apply -f -
```

- Katib

```
kustomize build apps/katib/upstream/installs/katib-with-kubeflow | kubectl apply -f -
```
Katib는 kubeflow의 AutoML 구성요소이지만, 노트북 실행시 katib가 없다면 에러가 발생 (ex. katib webhook ...)

- Central Dashboard

```
kustomize build apps/centraldashboard/overlays/oauth2-proxy | kubectl apply -f -
```


-  Admission Webhook
```
kustomize build apps/admission-webhook/upstream/overlays/cert-manager | kubectl apply -f -
```


- Notebooks 1.0
```
kustomize build apps/jupyter/notebook-controller/upstream/overlays/kubeflow | kubectl apply -f -
```

- Jupyter Web App for notebook
```
kustomize build apps/jupyter/jupyter-web-app/upstream/overlays/istio | kubectl apply -f -
```

- PVC Viewer Controller
```
kustomize build apps/pvcviewer-controller/upstream/default | kubectl apply -f -
```

- Profiles + KFAM
```
kustomize build apps/profiles/upstream/overlays/kubeflow | kubectl apply -f -
```

- Volums Web Application

```
kustomize build apps/volumes-web-app/upstream/overlays/istio | kubectl apply -f -

```




  

### optional

- Knative for KServe

```
kustomize build common/knative/knative-serving/overlays/gateways | kubectl apply -f -
kustomize build common/istio-1-23/cluster-local-gateway/base | kubectl apply -f -

```
- KServe

```
kustomize build contrib/kserve/kserve | kubectl apply --server-side --force-conflicts -f -
```

- model web application for KServe
```
kustomize build contrib/kserve/models-web-app/overlays/kubeflow | kubectl apply -f -
```



- kubeflow pielines (include argo)

```
kustomize build apps/pipeline/upstream/env/cert-manager/platform-agnostic-multi-user | kubectl apply -f -
```


- Tensorboard
```
kustomize build apps/tensorboard/tensorboards-web-app/upstream/overlays/istio | kubectl apply -f -
```

- Tensorboard Controller
```
kustomize build apps/tensorboard/tensorboard-controller/upstream/overlays/kubeflow | kubectl apply -f -
```
- Training Operator
```
kustomize build apps/training-operator/upstream/overlays/kubeflow | kubectl apply -f -
```

- User Namespaces
```
kustomize build common/user-namespace/base | kubectl apply -f -
```

kubectl apply -f C:\Users\family\Projects\mlops\mlops_platform_on_k8s\aws-ebs-sc.yaml




# 생성 확인
kubectl get pods -n cert-manager
kubectl get pods -n istio-system
kubectl get pods -n auth
kubectl get pods -n knative-eventing
kubectl get pods -n knative-serving
kubectl get pods -n kubeflow
kubectl get pods -n kubeflow-user-example-com

# 대시보드 포트포워딩 
kubectl port-forward svc/istio-ingressgateway -n istio-system 8080:80

기본유저 아이디 : user@example.com 비밀번호 : 12341234

# 삭제
kubectl delete namespace cert-manager
kubectl delete namespace istio-system
kubectl delete namespace auth
kubectl delete namespace knative-eventing
kubectl delete namespace knative-serving
kubectl delete namespace kubeflow
kubectl delete namespace kubeflow-user-example-com


# 트러블슈팅

## 파드생성불가능-1
파드 생성이 pending 될 경우, cpu 코어나 메모미, 저장공간이 충분하지 확인해보자.
`kubectl  describe 파드명 -n 파드의 네임스페이스` 를 톻해 상태를 확인해보자


## 파드생성불가능-2
위에서 언급한대로 `default storageclass`가 설정되어 있지 않으면 pvc가 bound 되지 않아 mysql 등의 파드들이 생성되지 않는다.

`kubectl get pvc -A` 를 통해 바운드 되지 않은 pvc가 있는지 확인한다.


