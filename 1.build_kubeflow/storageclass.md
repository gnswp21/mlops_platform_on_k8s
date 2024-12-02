## 2. eks make default storageclass (if gcp then pass to 3)

```
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update

helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver `
  --namespace kube-system `
  --set controller.serviceAccount.create=true


eksctl create iamserviceaccount `
  --name ebs-csi-controller-sa `
  --namespace kube-system `
  --cluster my-kube-1 `
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonEBSCSIDriverPolicy `
  --approve `
  --override-existing-serviceaccounts

```
# linux
aws eks update-kubeconfig --region ap-northeast-2 --name temp-kubeflow-1
## 1.  eks 설치
export CLUSTER_NAME=temp-kubeflow-1
export NODES=4
export INSTANCE_TYPES=m5.xlarge
eksctl create cluster --name $CLUSTER_NAME --with-oidc --instance-types=$INSTANCE_TYPES --managed --nodes=$NODES

## 2. aws-ebs-csi-driver를 프로비저너로 사용하는 default storageclass 생성,   (kubeflow pvc 설정에 의거해)
kubectl apply -f default-storage-class.yaml


<!-- ## 3. 노드그룹 IAM 권한설정
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster $CLUSTER_NAME \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonEBSCSIDriverPolicy \
  --approve \
  --override-existing-serviceaccounts -->

## 노드 그룹 역할에 EBSCSIDriverPolicy 추가

## aws ebs csi 드라이버 설치
  
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update
helm install aws-ebs-csi aws-ebs-csi-driver/aws-ebs-csi-driver --namespace kube-system



## 노드그룹에  ebs csi 드라이버 추가 IAM 권한



<!-- helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update

helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  --namespace kube-system \
  --set controller.serviceAccount.create=true -->

cd manifests
while ! kustomize build example | kubectl apply -f -; do echo "Retrying to apply resources"; sleep 20; done
