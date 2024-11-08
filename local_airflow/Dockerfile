FROM apache/airflow:2.10.0-python3.8
USER root
RUN mkdir -p /home/root
WORKDIR /home/root
## apt-get update
RUN apt-get update && apt-get install unzip && apt-get install -y procps && apt-get install -y lsof

# install aws cli
# https://docs.aws.amazon.com/ko_kr/cli/latest/userguide/getting-started-install.html
RUN curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" && \
  unzip awscliv2.zip && \
  ./aws/install

# install kubectl \
# https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/install-kubectl.html
RUN curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.30.2/2024-07-12/bin/linux/amd64/kubectl && \
  chmod +x ./kubectl  && \
  mv ./kubectl /usr/local/bin/kubectl


## eksctl
# https://eksctl.io/installation/#for-unix
# for ARM systems, set ARCH to: `arm64`, `armv6` or `armv7`
RUN ARCH=amd64 && \
  PLATFORM=$(uname -s)_$ARCH && \
  curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz" && \
  tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz  && \
  mv /tmp/eksctl /usr/local/bin


## helm k8s package manager
RUN curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 && \
  chmod 700 get_helm.sh && \
  ./get_helm.sh

## prometheus-community repo
RUN helm repo add prometheus-community https://prometheus-community.github.io/helm-charts && helm repo update


# Give airflow user access to binaries
RUN chown -R airflow /usr/local/bin /usr/local/aws-cli
# config dir가 없다.
RUN mkdir -p /opt/airflow/config 
RUN chown -R airflow /opt/airflow/config

# Switch back to airflow user
USER airflow

# insatll request python package
RUN pip install requests boto3 requests pandas


# Ensure airflow user PATH includes /usr/local/bin
ENV PATH="/usr/local/bin:$PATH"
WORKDIR /opt/airflow


