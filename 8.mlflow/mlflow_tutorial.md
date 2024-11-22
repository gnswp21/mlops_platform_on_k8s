
### linux mlflow 로그인
ID : user
PASSWORD : kubectl get secret --namespace mlflow mlflow-tracking -o jsonpath="{.data.admin-password }" | base64 -d

### PowerShell

kubectl get secret --namespace mlflow mlflow-tracking -o jsonpath="{.data.admin-password}" | ForEach-Object { [System.Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($_)) }
PwlnCZthnf


# 참고 블로그

https://suwani.tistory.com/169