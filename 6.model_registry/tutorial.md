cd manifest/model-registy/upstream

```
kubectl apply -k overlays/db
kubectl apply -k options/istio
```
```
kubectl wait --for=condition=available -n kubeflow deployment/model-registry-deployment --timeout=2m
kubectl logs -n kubeflow deployment/model-registry-deployment
```

### 연결 체크
노트북에서 터미널 연 후
```
curl model-registry-service.kubeflow.svc.cluster.local:8080/api/model_registry/v1alpha3/registered_models
```
에러 없으면 잘 된 것!