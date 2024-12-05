# kubeflow 운영을 위해 선택적 설치
## kubeflow 기반에 해당하여 일반적인 경우 필수 설치
### cert manager

```
kustomize build common/cert-manager/cert-manager/base | kubectl apply -f -
kustomize build common/cert-manager/kubeflow-issuer/base | kubectl apply -f -
echo "Waiting for cert-manager to be ready ..."
kubectl wait --for=condition=ready pod -l 'app in (cert-manager,webhook)' --timeout=180s -n cert-manager
kubectl wait --for=jsonpath='{.subsets[0].addresses[0].targetRef.kind}'=Pod endpoints -l 'app in (cert-manager,webhook)' --timeout=180s -n cert-manager

```

### istio
```
echo "Installing Istio configured with external authorization..."
kustomize build common/istio-1-23/istio-crds/base | kubectl apply -f -
kustomize build common/istio-1-23/istio-namespace/base | kubectl apply -f -
kustomize build common/istio-1-23/istio-install/overlays/oauth2-proxy | kubectl apply -f -

echo "Waiting for all Istio Pods to become ready..."
kubectl wait --for=condition=Ready pods --all -n istio-system --timeout 300s
```


### Oauth2-proxy
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

### dex
```
echo "Installing Dex..."
kustomize build common/dex/overlays/oauth2-proxy | kubectl apply -f -
kubectl wait --for=condition=ready pods --all --timeout=180s -n auth
```


### namespace
```
kustomize build common/kubeflow-namespace/base | kubectl apply -f -
```

### network policy
```
kustomize build common/networkpolicies/base | kubectl apply -f -
```

### kubeflow roles
```
kustomize build common/kubeflow-roles/base | kubectl apply -f -
```

### Kubeflow Istio Resources
```
kustomize build common/istio-1-23/kubeflow-istio-resources/base | kubectl apply -f -
```

### Katib

```
kustomize build apps/katib/upstream/installs/katib-with-kubeflow | kubectl apply -f -
```
Katib는 kubeflow의 AutoML 구성요소이지만, 노트북 실행시 katib가 없다면 에러가 발생 (ex. katib webhook ...)

### Central Dashboard

```
kustomize build apps/centraldashboard/overlays/oauth2-proxy | kubectl apply -f -
```


###  Admission Webhook
```
kustomize build apps/admission-webhook/upstream/overlays/cert-manager | kubectl apply -f -
```


### Notebooks 1.0
```
kustomize build apps/jupyter/notebook-controller/upstream/overlays/kubeflow | kubectl apply -f -
```

### Jupyter Web App for notebook
```
kustomize build apps/jupyter/jupyter-web-app/upstream/overlays/istio | kubectl apply -f -
```

### PVC Viewer Controller
```
kustomize build apps/pvcviewer-controller/upstream/default | kubectl apply -f -
```

### Profiles + KFAM
```
kustomize build apps/profiles/upstream/overlays/kubeflow | kubectl apply -f -
```

### Volums Web Application

```
kustomize build apps/volumes-web-app/upstream/overlays/istio | kubectl apply -f -

```




  
## optional
###  Kserve

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

### KFP

- kubeflow pielines (include argo)

```
kustomize build apps/pipeline/upstream/env/cert-manager/platform-agnostic-multi-user | kubectl apply -f -
```

### Tensorboard

- Tensorboard
```
kustomize build apps/tensorboard/tensorboards-web-app/upstream/overlays/istio | kubectl apply -f -
```

- Tensorboard Controller
```
kustomize build apps/tensorboard/tensorboard-controller/upstream/overlays/kubeflow | kubectl apply -f -
```

### Training Operator
- Training Operator
```
kustomize build apps/training-operator/upstream/overlays/kubeflow | kubectl apply -f -
```


### User Namespaces
- User Namespaces
```
kustomize build common/user-namespace/base | kubectl apply -f -
```


