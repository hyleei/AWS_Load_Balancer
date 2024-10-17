# AWS Load Balancer 실습 🚀

https://catalog.us-east-1.prod.workshops.aws/workshops/600420b7-5c4c-498f-9b80-bc7798963ba3/ko-KR/ec2/50-elb
AWS TechCamp의 실습자료를 AWS CLI를 사용하여 VPC, 서브넷, 인터넷 게이트웨이, 라우팅 테이블, 보안 그룹, EC2 인스턴스, 그리고 Application Load Balancer를 생성해보았습니다.


## Ubuntu에 AWS CLI 설치 및 설정 🛠️

1. AWS CLI 설치: `sudo apt update && sudo apt install awscli -y`
2. AWS CLI 버전 확인: `aws --version`
3. AWS 자격 증명 설정: `aws configure`
4. 자격 증명 확인: `aws sts get-caller-identity`

## 사전 준비 📋

- AWS CLI가 설치되어 있어야 합니다.
- AWS 계정과 적절한 권한이 있어야 합니다.
- `<KEY_PAIR_NAME>.pem` 키 페어가 생성되어 있어야 합니다.

## 실습 단계 🛠️

### 1. VPC 생성 🌐
```bash
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=<VPC_NAME>}]' --query 'Vpc.VpcId' --output text)
```

### 2. 서브넷 생성 🕸️
```bash
SUBNET_A_ID=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.10.0/24 --availability-zone <AZ_A> --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=<SUBNET_A_NAME>}]' --query 'Subnet.SubnetId' --output text)
SUBNET_C_ID=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.20.0/24 --availability-zone <AZ_C> --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=<SUBNET_C_NAME>}]' --query 'Subnet.SubnetId' --output text)
```

### 3. 인터넷 게이트웨이 생성 및 연결 🌉
```bash
IGW_ID=$(aws ec2 create-internet-gateway --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=<IGW_NAME>}]' --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID
```

### 라우팅 테이블 생성 및 설정 🗺️
```bash
RTB_ID=$(aws ec2 create-route-table --vpc-id $VPC_ID --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=<RTB_NAME>}]' --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id $RTB_ID --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
aws ec2 associate-route-table --subnet-id $SUBNET_A_ID --route-table-id $RTB_ID
aws ec2 associate-route-table --subnet-id $SUBNET_C_ID --route-table-id $RTB_ID
```

### 보안 그룹 생성 🔒
```bash
SG_ID=$(aws ec2 create-security-group --group-name <SG_NAME> --description "security group for web servers" --vpc-id $VPC_ID --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=<SG_NAME>}]' --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 22 --cidr $(curl -s ifconfig.me)/32
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 80 --cidr 0.0.0.0/0
```

### EC2 인스턴스 생성 💻
```bash
AMI_ID=$(aws ec2 describe-images --owners amazon --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" "Name=state,Values=available" --query "reverse(sort_by(Images, &CreationDate))[0].ImageId" --output text)
cat << EOF > user_data.sh
#!/bin/sh
        
# Install a LAMP stack
sudo amazon-linux-extras install -y lamp-mariadb10.2-php7.2 php7.2
sudo yum -y install httpd php-mbstring
sudo yum -y install git
        
# Start the web server
sudo chkconfig httpd on
sudo systemctl start httpd
        
# Install the web pages for our lab
if [ ! -f /var/www/html/aws-boarding-pass-webapp.tar.gz ]; then
    cd /var/www/html
    wget -O 'techcamp-webapp-2024.zip' 'https://ws-assets-prod-iad-r-icn-ced060f0d38bc0b0.s3.ap-northeast-2.amazonaws.com/600420b7-5c4c-498f-9b80-bc7798963ba3/techcamp-webapp-2024.zip'
    unzip techcamp-webapp-2024.zip
    sudo mv techcamp-webapp-2024/* .
    sudo rm -rf techcamp-webapp-2024
    sudo rm -rf techcamp-webapp-2024.zip
fi
        
# Install the AWS SDK for PHP
if [ ! -f /var/www/html/aws.zip ]; then
    cd /var/www/html
    sudo mkdir vendor
    cd vendor
    sudo wget https://docs.aws.amazon.com/aws-sdk-php/v3/download/aws.zip
    sudo unzip aws.zip
fi
        
# Update existing packages
sudo yum -y update > /var/www/html/index.html
EOF
INSTANCE_ID_1=$(aws ec2 run-instances --image-id $AMI_ID --count 1 --instance-type t2.micro --key-name <KEY_PAIR_NAME> --security-group-ids $SG_ID --subnet-id $SUBNET_A_ID --associate-public-ip-address --user-data file://user_data.sh --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=<INSTANCE_1_NAME>}]' --query 'Instances[0].InstanceId' --output text)
INSTANCE_ID_2=$(aws ec2 run-instances --image-id $AMI_ID --count 1 --instance-type t2.micro --key-name <KEY_PAIR_NAME> --security-group-ids $SG_ID --subnet-id $SUBNET_C_ID --associate-public-ip-address --user-data file://user_data.sh --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=<INSTANCE_2_NAME>}]' --query 'Instances[0].InstanceId' --output text)
```

### 7. 로드 밸런서 생성 ⚖️
```bash
ALB_ARN=$(aws elbv2 create-load-balancer --name <ALB_NAME> --subnets $SUBNET_A_ID $SUBNET_C_ID --security-groups $SG_ID --scheme internet-facing --type application --query 'LoadBalancers[0].LoadBalancerArn' --output text)
TG_ARN=$(aws elbv2 create-target-group --name <TG_NAME> --protocol HTTP --port 80 --vpc-id $VPC_ID --health-check-path / --query 'TargetGroups[0].TargetGroupArn' --output text)
aws elbv2 register-targets --target-group-arn $TG_ARN --targets Id=$INSTANCE_ID_1 Id=$INSTANCE_ID_2
aws elbv2 create-listener --load-balancer-arn $ALB_ARN --protocol HTTP --port 80 --default-actions Type=forward,TargetGroupArn=$TG_ARN
```

## 문제 해결 🔧
인스턴스 상태가 unhealthy인 경우:

1. 대상 그룹 상태 확인: aws elbv2 describe-target-health --target-group-arn $TG_ARN
2. 인스턴스에 SSH 접속:
```bash
UNHEALTHY_IP=$(aws ec2 describe-instances --instance-ids $UNHEALTHY_INSTANCE --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
ssh -i ~/<KEY_PAIR_NAME>.pem ec2-user@$UNHEALTHY_IP
```
3. 웹 서버 상태 확인 및 시작:
```bash
sudo systemctl status httpd
sudo systemctl start httpd
sudo systemctl enable httpd
```

4. 로그 확인:
```bash
sudo tail -f /var/log/httpd/error_log
sudo cat /var/log/cloud-init-output.log
```
   
## 자원 삭제 🧹

생성한 자원들을 삭제하려면 다음 명령어들을 순서대로 실행하세요. 각 명령어 실행 후 자원이 완전히 삭제될 때까지 기다린 후 다음 명령어를 실행하는 것이 좋습니다.

1. 로드 밸런서 삭제
```bash
aws elbv2 delete-listener --listener-arn $LISTENER_ARN
aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN
aws elbv2 delete-target-group --target-group-arn $TG_ARN
```

2. EC2 인스턴스 종료
```bash
aws ec2 terminate-instances --instance-ids $INSTANCE_ID_1 $INSTANCE_ID_2
```

3. 보안 그룹 삭제
```bash
aws ec2 delete-security-group --group-id $SG_ID
```

4. 라우팅 테이블 삭제
```bash
aws ec2 disassociate-route-table --association-id $ROUTE_TABLE_ASSOC_ID_A
aws ec2 disassociate-route-table --association-id $ROUTE_TABLE_ASSOC_ID_C
aws ec2 delete-route-table --route-table-id $RTB_ID
```

5. 인터넷 게이트웨이 분리 및 삭제
```bash
aws ec2 detach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID
```

6. 서브넷 삭제
```bash
aws ec2 delete-subnet --subnet-id $SUBNET_A_ID
aws ec2 delete-subnet --subnet-id $SUBNET_C_ID
```

7. VPC 삭제
```bash
aws ec2 delete-vpc --vpc-id $VPC_ID
```   
