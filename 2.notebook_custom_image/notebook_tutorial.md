# 노트북 튜토리얼

[쿠브플로 노트북 공식문서](https://www.kubeflow.org/docs/components/notebooks/)

## 커스텀 노트북이 필요한 이유
- 노트북은 기본적으로 파드형태로 생성되므로 노트북에서 설치한 패키지나 환경변수등은 파드가 종류됨(노트북이 삭제됨)에 따라 초기화된다.
- 따라서, 영구적인 변화가 필요하다면 커스텀 노트북을 사용하자

## 커스텀 노트북에 필요한 도커 파일 생성


위의 공식문서에 따라 가장 쉽게 커스텀 노트북이미지를 만드는 것은 기존의 노트북 이미지를 활용하는 것이다.

[쥬피터-파이토치-풀-도커파일](https://github.com/kubeflow/kubeflow/blob/master/components/example-notebook-servers/jupyter-pytorch-full/Dockerfile)

해당 링크의 도커파일은 kubeflow에서 기본적으로 제공하는 노트북이미지 빌드에 사용되는 도커파일이다.

프로젝트에 사용된 노트북 이미지는 넘파이 버전 불일치 문제를 해결하기 위한 이미지이다. 

- [커스텀 도커파일](/2.notebook_custom_image/Dockerfile)


위의 공식문서에서는 사용된 도커파일을 통해 apt 패키지 설치방법, 파이썬 패키지 설치방법에 대해 설명하므로 이를 참고하자.



## 이미지 적용
도커파일을 만들었다면 다음의 과정을 통해 커스텀 노트북을 사용할 수 있다.

![이미지](/9.trobleshooting-image/커스텀%20노트북.png)

준비
- 도커 로그인 정보가 k8s에 시크릿으로 제공되는 환경 (kubeflow 설치과정을 따랐다면 적용되어 있다.)

과정
- 도커 파일 빌드로 이미지 생성
- 이미지 태그 후 개인/회사 레포지토리로 업로드
- kubeflow 노트북 생성 대시보드에서 위에서 새롭게 태그하여 업로드한 커스텀 노트북 이미지를 지정해서 사용 (ex. gnswp21/custom-notebook)










