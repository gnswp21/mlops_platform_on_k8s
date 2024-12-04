# mlflow 튜토리얼

## mlflow 설치

### 공식문서의 설명
공식 문서에 따르면 k8s상의 mlflow는 공식적인 helm 차트는 없으므로 bitnami가 관리하는 mlflow를 사용한다.

- Helm chart mlflow chatbot의 대답
```
To install MLflow on Kubernetes, you can use Helm charts for a streamlined deployment. While 
there isn't an official MLflow Helm chart, community and third-party charts are available. For 
example, Bitnami provides a Helm chart for MLflow, which can be used to deploy MLflow on 
Kubernetes. You can find the chart and instructions on the Bitnami GitHub repository.
```

### mlflow의 두가지 저장소와 minio

- mlflow 아티팩트 저장소로 minio 사용
  mlflow는 두가지 저장소가 필요하다. 메타데이터 저장소인 DB와 아티팩트 저장소인 Storage이다. 여기서 아티팩트 저장소를 kubeflow에서 기본적으로 사용하는 저장소인 minio를 지정하여 저장소를 통일 시킬 수 있다.


포트포워딩 혹은 외우 ip를 통해 minio를 관리하는 UI에 접속할 수 있다.
`kubectl port-forward -n kubeflow svc/minio-service 9000:9000`
- Access Key: minio
- Secret Key: minio123

해당 ui에서 mlflow라는 버킷을 만들 수 있다.

### mlflow 설치 및 minio와 연결

```
kubectl create ns mlflow
 
kubectl create -n mlflow secret generic mlflow-secret \
    --from-literal=AWS_ACCESS_KEY_ID=minio \
    --from-literal=AWS_SECRET_ACCESS_KEY='minio123'
 
helm repo add bitnami https://charts.bitnami.com/bitnami
helm upgrade --install mlflow bitnami/mlflow -n mlflow --version 0.10.3 \
--set minio.enabled=false \
--set externalS3.host=minio-service.kubeflow \
--set externalS3.port=9000 \
--set externalS3.existingSecret=mlflow-secret \
--set externalS3.existingSecretAccessKeyIDKey="AWS_ACCESS_KEY_ID" \
--set externalS3.existingSecretKeySecretKey="AWS_SECRET_ACCESS_KEY" \
--set externalS3.bucket="mlflow" \
--set externalS3.protocol=http
```



이제 mlflow가 정상적으로 설치되었다.

## mlflow UI
저장된 모델과 메타데이터를 시각적으로 확인할 수 있는 UI이다.

### mlflow 로그인
다음이 admin 계정의 기본값으로 설정되어 있다. UI 접속에 필요하다.

ID : user
PASSWORD : kubectl get secret --namespace mlflow mlflow-tracking -o jsonpath="{.data.admin-password }" | base64 -d


![이미지]()






## 실험 추적 및 모델 레지스트리

준비사항
- 노트북 커스텀 이미지 혹은 노트북 내부 쉘을 통해 mlflow API 파이썬 패키지 설치  `pip install mlflow`

참고
- [참고한 mlflow 공식문서](https://mlflow.org/docs/latest/getting-started/intro-quickstart/index.html)

mlflow가 설치된 쥬피터 노트북에서 다음 두가지 셀을 통해 모델 실험을 추적하고 저장할 수 있다.


[pipeline](/8.mlflow/model_train.ipynb)

위 과정들을 통해 실험과 실험에 사용된 모델이 mlflow를 통해 minio에 저장된다.
minio UI, mlflow 트랙킹 서버 UI 양측에서 확인 가능하다

![minio UI 이미지]()
![mlflow UI 이미지]()



### 참고: https://suwani.tistory.com/169



