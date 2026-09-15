```sh
# export <- 한 터미널 안에서만 유지가 된다 (새롭게 gitbash나 다른 창을 띄우면 리셋)
# export STUDENT_ID="student00"
export STUDENT_ID="studentXX"
export AWS_PROFILE="$STUDENT_ID"
export AWS_REGION="ap-northeast-2"
export AWS_PAGER=""
```

```sh
export MY_KEY_NAME="${STUDENT_ID}-key"
export MY_SG_NAME="${STUDENT_ID}-web-sg"
export MY_INSTANCE_NAME="${STUDENT_ID}-compose-ec2"
```

```sh
# 키 페어: 로컬 .pem이 없으면 기존 키를 정리하고 새로 발급
# 키 페어 파일이 없으면
if [ ! -f ./"$MY_KEY_NAME".pem ]; then
  # 해당 키 페어 이름으로 서버에 등록된 것을 지우고
  aws ec2 delete-key-pair --key-name "$MY_KEY_NAME" >/dev/null 2>&1
  # 새롭게 해당 이름으로 해서 이 설정으로 키 페어를 만들어 달라
  aws ec2 create-key-pair --key-name "$MY_KEY_NAME" \
    --tag-specifications "ResourceType=key-pair,Tags=[{Key=Name,Value=$MY_KEY_NAME},{Key=Course,Value=infra-training},{Key=Owner,Value=$STUDENT_ID}]" \
    --query "KeyMaterial" --output text > ./"$MY_KEY_NAME".pem
  # 권한을 400으로 (AWS에서 사용가능하게)
  chmod 400 ./"$MY_KEY_NAME".pem
fi
# 해당 pem의 권한을 표시
ls -l ./"$MY_KEY_NAME".pem
```

---

```sh
aws sts get-caller-identity
```

```sh
aws configure sso --profile "${STUDENT_ID}"
# SSO session name: infra-training
# SSO start URL: https://infra-lab-ai.awsapps.com/start
# SSO region: ap-northeast-2
# SSO registration scopes: sso:account:access

# CLI default client Region: ap-northeast-2
# CLI default output format: json
# CLI profile name: studentXX
```

```sh
aws sso login --profile "${STUDENT_ID}"
```

---

```sh
rm -rf ./"$MY_KEY_NAME".pem
aws ec2 delete-key-pair --key-name "$MY_KEY_NAME"
aws ec2 create-key-pair --key-name "$MY_KEY_NAME" \
    --tag-specifications "ResourceType=key-pair,Tags=[{Key=Name,Value=$MY_KEY_NAME},{Key=Course,Value=infra-training},{Key=Owner,Value=$STUDENT_ID}]" \
    --query "KeyMaterial" --output text > ./"$MY_KEY_NAME".pem
chmod 400 ./"$MY_KEY_NAME".pem
```

---

```sh
# echo $MY_SG_NAME
export MY_SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=$MY_SG_NAME" \
  --query "SecurityGroups[0].GroupId" --output text)

if [ "$MY_SG_ID" = "None" ]; then
  export VPC_ID=$(aws ec2 describe-vpcs --filters "Name=is-default,Values=true" \
    --query "Vpcs[0].VpcId" --output text)
  export MY_SG_ID=$(aws ec2 create-security-group \
    --group-name "$MY_SG_NAME" --vpc-id "$VPC_ID" \
    --description "Security Group for Docker Compose Practice" \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=$MY_SG_NAME},{Key=Course,Value=infra-training},{Key=Owner,Value=$STUDENT_ID}]" \
    --query "GroupId" --output text)
fi
echo "보안 그룹 ID:$MY_SG_ID"
```

```sh
# 현재 접속 중인 컴퓨터가 쓰고 있는 ip 주소
# curl -fsS https://checkip.amazonaws.com
export MY_IP=$(curl -fsS https://checkip.amazonaws.com)
# 22 -> 내가 접속한 곳에서만 허용하게 
aws ec2 authorize-security-group-ingress --group-id "$MY_SG_ID" --protocol tcp --port 22 --cidr "$MY_IP/32"
# 80, 8080 <- 외부에서도 접속해서 (서버 역할)
aws ec2 authorize-security-group-ingress --group-id "$MY_SG_ID" --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id "$MY_SG_ID" --protocol tcp --port 8080 --cidr 0.0.0.0/0

aws ec2 describe-security-groups --group-ids "$MY_SG_ID" \
  --query "SecurityGroups[0].IpPermissions[].{Port:FromPort,Cidr:IpRanges[0].CidrIp}" --output table
```

---

```sh
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=$MY_INSTANCE_NAME" "Name=tag:Owner,Values=$STUDENT_ID" \
            "Name=instance-state-name,Values=pending,running,stopping,stopped" \
  --query "Reservations[].Instances[].[InstanceId,InstanceType,State.Name]" --output table
```

```sh
export AMI_ID=$(aws ssm get-parameter \
  --name /aws/service/canonical/ubuntu/server/26.04/stable/current/arm64/hvm/ebs-gp3/ami-id \
  --query "Parameter.Value" --output text)
```

```sh
export INSTANCE_ID=$(aws ec2 run-instances \
  --image-id "$AMI_ID" --instance-type t4g.small \
  --key-name "$MY_KEY_NAME" --security-group-ids "$MY_SG_ID" \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=$MY_INSTANCE_NAME},{Key=Course,Value=infra-training},{Key=Owner,Value=$STUDENT_ID}]" \
  --query "Instances[0].InstanceId" --output text)
echo "인스턴스 ID:$INSTANCE_ID"

#aws ec2 wait instance-running --instance-ids "$INSTANCE_ID"
#aws ec2 wait instance-status-ok --instance-ids "$INSTANCE_ID"
```

- https://948806325749-ticmxh4e.ap-northeast-2.console.aws.amazon.com/ec2/home?region=ap-northeast-2#Instances:v=3;instanceState=running

---

```sh
# 공인 IP 조회
export PUBLIC_IP=$(aws ec2 describe-instances --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].PublicIpAddress" --output text)
echo "공인 IP:$PUBLIC_IP"
```

```sh
ssh -i ./"$MY_KEY_NAME".pem -o StrictHostKeyChecking=accept-new ubuntu@"$PUBLIC_IP"
```

```sh
uname -m
free -h | head -2
```

```sh
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

sudo systemctl is-active docker
sudo docker --version
sudo docker compose version

# sudo docker ps
```

---

```sh
vi nginx.conf

# i -> INSERT
events {
    worker_connections 1024;
}

http {
    server {
        listen 80;

        location / {
            proxy_pass http://app:8080;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}

# esc -> :wq -> enter
cat nginx.conf
# 기본 10줄
# head nginx.conf
# tail nginx.conf
# tail -n 20 nginx.conf
```
