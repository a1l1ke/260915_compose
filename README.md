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