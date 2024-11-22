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