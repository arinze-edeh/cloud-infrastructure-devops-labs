

<img width="1032" height="837" alt="image" src="https://github.com/user-attachments/assets/391530a9-3e3d-4247-96d0-0038cbc48673" />
<img width="1028" height="866" alt="image" src="https://github.com/user-attachments/assets/8d064d34-1dd3-407b-8919-f8c779b4406c" />
<img width="1034" height="845" alt="image" src="https://github.com/user-attachments/assets/d697d887-a697-42c2-b20a-3a951578f99b" />
<img width="1031" height="841" alt="image" src="https://github.com/user-attachments/assets/fd60add0-c99c-4d51-91d9-eefe5a917e52" />
<img width="1034" height="865" alt="image" src="https://github.com/user-attachments/assets/d21d4f15-e399-4b28-bf2f-530141faef9d" />
<img width="1030" height="853" alt="image" src="https://github.com/user-attachments/assets/08a8cd81-0f55-4f52-88da-0cb6113bf1f7" />
<img width="1028" height="867" alt="image" src="https://github.com/user-attachments/assets/ea8e1305-e144-4e9c-8cc2-24626e759ca2" />
<img width="1024" height="862" alt="image" src="https://github.com/user-attachments/assets/e3ce40aa-58e1-460e-b7c1-fe17febc16f8" />
<img width="1028" height="844" alt="image" src="https://github.com/user-attachments/assets/ab08b5b2-a0a5-4bd6-a988-fcd8e93738fb" />
<img width="1032" height="860" alt="image" src="https://github.com/user-attachments/assets/1bc4f0b6-0558-4b19-bedd-d8f9a6c4d28c" />
<img width="1040" height="871" alt="image" src="https://github.com/user-attachments/assets/3517b524-fde7-432b-a6b9-3834f614cee6" />
<img width="1080" height="861" alt="image" src="https://github.com/user-attachments/assets/9dd6b873-178c-4852-8bf6-48ce371c9fc7" />
<img width="1082" height="871" alt="image" src="https://github.com/user-attachments/assets/283a6b21-a826-4953-ad22-aac8aa760ed0" />
<img width="1081" height="848" alt="image" src="https://github.com/user-attachments/assets/19976361-e237-4ea2-ac39-e513eb673cd6" />
<img width="1079" height="863" alt="image" src="https://github.com/user-attachments/assets/a7e60128-588a-4a9d-9638-59fe16aff1aa" />
<img width="1078" height="862" alt="image" src="https://github.com/user-attachments/assets/f4c8b422-25f7-4a49-82ee-f6c8f5f348cc" />
<img width="1134" height="855" alt="image" src="https://github.com/user-attachments/assets/5fca679d-b771-451d-b568-60fc2f9cdd57" />
<img width="1130" height="862" alt="image" src="https://github.com/user-attachments/assets/8f75199b-87e5-403a-9111-2a6d0bacbaac" />
<img width="1049" height="857" alt="image" src="https://github.com/user-attachments/assets/7c8adb28-5c53-49eb-b170-f2e3aad26b2a" />









<img width="654" height="882" alt="image" src="https://github.com/user-attachments/assets/8779ceee-db81-4b33-a573-00e7c44a4e67" />


~ on ☁️  (us-east-1) ➜  PUB_VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.1.0.0/16 \
  --region us-east-1 \
  --query 'Vpc.VpcId' \
  --output text) && \
aws ec2 create-tags \
  --resources $PUB_VPC_ID \
  --tags Key=Name,Value=devops-pub-vpc \
  --region us-east-1 && \
PUB_SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id $PUB_VPC_ID \
  --cidr-block 10.1.1.0/24 \
  --region us-east-1 \
  --query 'Subnet.SubnetId' \
  --output text) && \
aws ec2 create-tags \
  --resources $PUB_SUBNET_ID \
  --tags Key=Name,Value=devops-pub-subnet \
  --region us-east-1 && \
PUB_RT_ID=$(aws ec2 create-route-table \
  --vpc-id $PUB_VPC_ID \
  --region us-east-1 \
  --query 'RouteTable.RouteTableId' \
  --output text) && \
aws ec2 create-tags \
  --resources $PUB_RT_ID \
  --tags Key=Name,Value=devops-pub-rt \
  --region us-east-1 && \
aws ec2 associate-route-table \
  --route-table-id $PUB_RT_ID \
  --subnet-id $PUB_SUBNET_ID \
  --region us-east-1 && \
echo "PUB_VPC_ID=$PUB_VPC_ID  PUB_SUBNET_ID=$PUB_SUBNET_ID  PUB_RT_ID=$PUB_RT_ID"
{
    "AssociationId": "rtbassoc-0523e7ee3745af09b",
    "AssociationState": {
        "State": "associated"
    }
}
PUB_VPC_ID=vpc-00c2cdb99aaab238e  PUB_SUBNET_ID=subnet-0d8b57cf9b154a022  PUB_RT_ID=rtb-0bb48479c56abb39d

~ on ☁️  (us-east-1) ➜  IGW_ID=$(aws ec2 create-internet-gateway \
  --region us-east-1 \
  --query 'InternetGateway.InternetGatewayId' \
  --output text) && \
aws ec2 attach-internet-gateway \
  --internet-gateway-id $IGW_ID \
  --vpc-id $PUB_VPC_ID \
  --region us-east-1 && \
aws ec2 create-route \
  --route-table-id $PUB_RT_ID \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $IGW_ID \
  --region us-east-1 && \
aws ec2 modify-subnet-attribute \
  --subnet-id $PUB_SUBNET_ID \
  --map-public-ip-on-launch \
  --region us-east-1 && \
echo "IGW_ID=$IGW_ID"
{
    "Return": true
}
IGW_ID=igw-0ec6e010f95525cfd

~ on ☁️  (us-east-1) ➜  KEY_NAME=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=devops-priv-ec2" \
  --query 'Reservations[0].Instances[0].KeyName' \
  --output text \
  --region us-east-1) && \
UBUNTU_AMI=$(aws ec2 describe-images \
  --owners 099720109477 \
  --filters \
    "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
    "Name=state,Values=available" \
  --query 'sort_by(Images,&CreationDate)[-1].ImageId' \
  --output text \
  --region us-east-1) && \
PUB_SG_ID=$(aws ec2 create-security-group \
  --group-name devops-pub-sg \
  --description "SG for devops-pub-ec2" \
  --vpc-id $PUB_VPC_ID \
  --region us-east-1 \
  --query 'GroupId' \
  --output text) && \
aws ec2 authorize-security-group-ingress \
  --group-id $PUB_SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0 \
  --region us-east-1 && \
PUB_EC2_ID=$(aws ec2 run-instances \
  --image-id $UBUNTU_AMI \
  --instance-type t2.micro \
  --key-name "$KEY_NAME" \
  --subnet-id $PUB_SUBNET_ID \
  --security-group-ids $PUB_SG_ID \
  --region us-east-1 \
  --query 'Instances[0].InstanceId' \
  --output text) && \
aws ec2 create-tags \
  --resources $PUB_EC2_ID \
  --tags Key=Name,Value=devops-pub-ec2 \
  --region us-east-1 && \
aws ec2 wait instance-running \
  --instance-ids $PUB_EC2_ID \
  --region us-east-1 && \
echo "KEY_NAME=$KEY_NAME  UBUNTU_AMI=$UBUNTU_AMI  PUB_SG_ID=$PUB_SG_ID  PUB_EC2_ID=$PUB_EC2_ID"
{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-0eb36676e79043eb5",
            "GroupId": "sg-0e7dff2b2dbff34fd",
            "GroupOwnerId": "284304506227",
            "IsEgress": false,
            "IpProtocol": "tcp",
            "FromPort": 22,
            "ToPort": 22,
            "CidrIpv4": "0.0.0.0/0",
            "SecurityGroupRuleArn": "arn:aws:ec2:us-east-1:284304506227:security-group-rule/sgr-0eb36676e79043eb5"
        }
    ]
}
KEY_NAME=devops-key  UBUNTU_AMI=ami-00de3875b03809ec5  PUB_SG_ID=sg-0e7dff2b2dbff34fd  PUB_EC2_ID=i-05a7403454c019495

~ on ☁️  (us-east-1) ➜  aws iam create-role \
  --role-name devops-s3-role \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}' && \
aws iam attach-role-policy \
  --role-name devops-s3-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess && \
aws iam create-instance-profile \
  --instance-profile-name devops-s3-role && \
aws iam add-role-to-instance-profile \
  --instance-profile-name devops-s3-role \
  --role-name devops-s3-role && \
sleep 15 && \
aws ec2 associate-iam-instance-profile \
  --instance-id $PUB_EC2_ID \
  --iam-instance-profile Name=devops-s3-role \
  --region us-east-1 && \
echo "IAM role attached"
{
    "Role": {
        "Path": "/",
        "RoleName": "devops-s3-role",
        "RoleId": "AROAUEMO6PVZ56J3PEPSG",
        "Arn": "arn:aws:iam::284304506227:role/devops-s3-role",
        "CreateDate": "2026-03-25T11:42:19Z",
        "AssumeRolePolicyDocument": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Principal": {
                        "Service": "ec2.amazonaws.com"
                    },
                    "Action": "sts:AssumeRole"
                }
            ]
        }
    }
}
{
    "InstanceProfile": {
        "Path": "/",
        "InstanceProfileName": "devops-s3-role",
        "InstanceProfileId": "AIPAUEMO6PVZ2GMD66KZC",
        "Arn": "arn:aws:iam::284304506227:instance-profile/devops-s3-role",
        "CreateDate": "2026-03-25T11:42:21Z",
        "Roles": []
    }
}
{
    "IamInstanceProfileAssociation": {
        "AssociationId": "iip-assoc-06957e99034e284e9",
        "InstanceId": "i-05a7403454c019495",
        "IamInstanceProfile": {
            "Arn": "arn:aws:iam::284304506227:instance-profile/devops-s3-role",
            "Id": "AIPAUEMO6PVZ2GMD66KZC"
        },
        "State": "associating"
    }
}
IAM role attached

~ on ☁️  (us-east-1) ➜  aws s3api create-bucket \
  --bucket devops-s3-logs-25406 \
  --region us-east-1 && \
aws s3api put-public-access-block \
  --bucket devops-s3-logs-25406 \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true && \
echo "S3 bucket ready: devops-s3-logs-25406"
{
    "Location": "/devops-s3-logs-25406"
}
S3 bucket ready: devops-s3-logs-25406

~ on ☁️  (us-east-1) ➜  PRIV_VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=devops-priv-vpc" \
  --query 'Vpcs[0].VpcId' \
  --output text \
  --region us-east-1) && \
PRIV_VPC_CIDR=$(aws ec2 describe-vpcs \
  --vpc-ids $PRIV_VPC_ID \
  --query 'Vpcs[0].CidrBlock' \
  --output text \
  --region us-east-1) && \
PEERING_ID=$(aws ec2 create-vpc-peering-connection \
  --vpc-id $PUB_VPC_ID \
  --peer-vpc-id $PRIV_VPC_ID \
  --region us-east-1 \
  --query 'VpcPeeringConnection.VpcPeeringConnectionId' \
  --output text) && \
aws ec2 create-tags \
  --resources $PEERING_ID \
  --tags Key=Name,Value=devops-vpc-peering \
  --region us-east-1 && \
aws ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id $PEERING_ID \
  --region us-east-1 && \
PRIV_RT_ID=$(aws ec2 describe-route-tables \
  --filters "Name=tag:Name,Values=devops-priv-rt" \
  --query 'RouteTables[0].RouteTableId' \
  --output text \
  --region us-east-1) && \
aws ec2 create-route \
  --route-table-id $PRIV_RT_ID \
  --destination-cidr-block 10.1.0.0/16 \
  --vpc-peering-connection-id $PEERING_ID \
  --region us-east-1 && \
aws ec2 create-route \
  --route-table-id $PUB_RT_ID \
  --destination-cidr-block $PRIV_VPC_CIDR \
  --vpc-peering-connection-id $PEERING_ID \
  --region us-east-1 && \
echo "PRIV_VPC_ID=$PRIV_VPC_ID  PRIV_VPC_CIDR=$PRIV_VPC_CIDR  PEERING_ID=$PEERING_ID  PRIV_RT_ID=$PRIV_RT_ID"
{
    "VpcPeeringConnection": {
        "AccepterVpcInfo": {
            "CidrBlock": "10.10.0.0/16",
            "CidrBlockSet": [
                {
                    "CidrBlock": "10.10.0.0/16"
                }
            ],
            "OwnerId": "284304506227",
            "PeeringOptions": {
                "AllowDnsResolutionFromRemoteVpc": false,
                "AllowEgressFromLocalClassicLinkToRemoteVpc": false,
                "AllowEgressFromLocalVpcToRemoteClassicLink": false
            },
            "VpcId": "vpc-09f0f241ccc7851e0",
            "Region": "us-east-1"
        },
        "RequesterVpcInfo": {
            "CidrBlock": "10.1.0.0/16",
            "CidrBlockSet": [
                {
                    "CidrBlock": "10.1.0.0/16"
                }
            ],
            "OwnerId": "284304506227",
            "PeeringOptions": {
                "AllowDnsResolutionFromRemoteVpc": false,
                "AllowEgressFromLocalClassicLinkToRemoteVpc": false,
                "AllowEgressFromLocalVpcToRemoteClassicLink": false
            },
            "VpcId": "vpc-00c2cdb99aaab238e",
            "Region": "us-east-1"
        },
        "Status": {
            "Code": "provisioning",
            "Message": "Provisioning"
        },
        "Tags": [],
        "VpcPeeringConnectionId": "pcx-07ff722f053d64b7b"
    }
}
{
    "Return": true
}
{
    "Return": true
}
PRIV_VPC_ID=vpc-09f0f241ccc7851e0  PRIV_VPC_CIDR=10.10.0.0/16  PEERING_ID=pcx-07ff722f053d64b7b  PRIV_RT_ID=rtb-0b2873c0c9a0cb96c

~ on ☁️  (us-east-1) ➜  PRIV_SG_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=devops-priv-ec2" \
  --query 'Reservations[0].Instances[0].SecurityGroups[0].GroupId' \
  --output text \
  --region us-east-1) && \
aws ec2 authorize-security-group-ingress \
  --group-id $PRIV_SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr 10.1.0.0/16 \
  --region us-east-1 && \
PUB_EC2_PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids $PUB_EC2_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text \
  --region us-east-1) && \
PUB_EC2_PRIVATE_IP=$(aws ec2 describe-instances \
  --instance-ids $PUB_EC2_ID \
  --query 'Reservations[0].Instances[0].PrivateIpAddress' \
  --output text \
  --region us-east-1) && \
PRIV_EC2_PRIVATE_IP=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=devops-priv-ec2" \
  --query 'Reservations[0].Instances[0].PrivateIpAddress' \
  --output text \
  --region us-east-1) && \
echo "PUB_PUBLIC=$PUB_EC2_PUBLIC_IP  PUB_PRIVATE=$PUB_EC2_PRIVATE_IP  PRIV_PRIVATE=$PRIV_EC2_PRIVATE_IP"
{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-0c77b4fb2d99146f5",
            "GroupId": "sg-034057aef886496b9",
            "GroupOwnerId": "284304506227",
            "IsEgress": false,
            "IpProtocol": "tcp",
            "FromPort": 22,
            "ToPort": 22,
            "CidrIpv4": "10.1.0.0/16",
            "SecurityGroupRuleArn": "arn:aws:ec2:us-east-1:284304506227:security-group-rule/sgr-0c77b4fb2d99146f5"
        }
    ]
}
PUB_PUBLIC=3.235.170.246  PUB_PRIVATE=10.1.1.85  PRIV_PRIVATE=10.10.1.235

~ on ☁️  (us-east-1) ➜  scp -i /root/.ssh/devops-key.pem \
  -o StrictHostKeyChecking=no \
  /root/.ssh/devops-key.pem \
  ubuntu@$PUB_EC2_PUBLIC_IP:/home/ubuntu/.ssh/devops-key.pem && \
ssh -i /root/.ssh/devops-key.pem \
  -o StrictHostKeyChecking=no \
  ubuntu@$PUB_EC2_PUBLIC_IP \
  "chmod 600 /home/ubuntu/.ssh/devops-key.pem" && \
eval $(ssh-agent) && \
ssh-add /root/.ssh/devops-key.pem && \
ssh -i /root/.ssh/devops-key.pem \
  -o StrictHostKeyChecking=no \
  -A ubuntu@$PUB_EC2_PUBLIC_IP \
  "scp -o StrictHostKeyChecking=no /home/ubuntu/.ssh/devops-key.pem ubuntu@$PRIV_EC2_PRIVATE_IP:/home/ubuntu/.ssh/devops-key.pem && \ 
  ssh -o StrictHostKeyChecking=no ubuntu@$PRIV_EC2_PRIVATE_IP 'chmod 600 /home/ubuntu/.ssh/devops-key.pem'" && \
ssh -i /root/.ssh/devops-key.pem \
  -o StrictHostKeyChecking=no \
  ubuntu@$PUB_EC2_PUBLIC_IP \
  "cat /home/ubuntu/.ssh/devops-key.pem | ssh-keygen -y -f /dev/stdin >> /home/ubuntu/.ssh/authorized_keys && echo 'Public key added'" && \
echo "Keys distributed"
Warning: Permanently added '3.235.170.246' (ECDSA) to the list of known hosts.
devops-key.pem                                                                                     100% 1675    16.6KB/s   00:00    
Agent pid 1534
Identity added: /root/.ssh/devops-key.pem (/root/.ssh/devops-key.pem)
Warning: Permanently added '10.10.1.235' (ED25519) to the list of known hosts.
Public key added
Keys distributed

~ on ☁️  (us-east-1) ➜  ssh -i /root/.ssh/devops-key.pem \
  -o StrictHostKeyChecking=no \
  ubuntu@$PUB_EC2_PUBLIC_IP \
  "sudo apt-get update -y -qq && \
  sudo apt-get install -y unzip curl -qq && \
  curl -s https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip -o awscliv2.zip && \
  unzip -q awscliv2.zip && \
  sudo ./aws/install && \
  /usr/local/bin/aws --version && \
  echo 'AWS CLI installed'"
debconf: unable to initialize frontend: Dialog
debconf: (Dialog frontend will not work on a dumb terminal, an emacs shell buffer, or without a controlling terminal.)
debconf: falling back to frontend: Readline
debconf: unable to initialize frontend: Readline
debconf: (This frontend requires a controlling tty.)
debconf: falling back to frontend: Teletype
dpkg-preconfigure: unable to re-open stdin: 
Selecting previously unselected package unzip.
(Reading database ... 66073 files and directories currently installed.)
Preparing to unpack .../unzip_6.0-26ubuntu3.2_amd64.deb ...
Unpacking unzip (6.0-26ubuntu3.2) ...
Setting up unzip (6.0-26ubuntu3.2) ...
Processing triggers for man-db (2.10.2-1) ...

Running kernel seems to be up-to-date.

No services need to be restarted.

No containers need to be restarted.

No user sessions are running outdated binaries.

No VM guests are running outdated hypervisor (qemu) binaries on this host.
You can now run: /usr/local/bin/aws --version
aws-cli/2.34.16 Python/3.14.3 Linux/6.8.0-1050-aws exe/x86_64.ubuntu.22
AWS CLI installed

~ on ☁️  (us-east-1) ➜  ssh -i /root/.ssh/devops-key.pem \
  -o StrictHostKeyChecking=no \
  -A ubuntu@$PUB_EC2_PUBLIC_IP \
  "ssh -o StrictHostKeyChecking=no ubuntu@$PRIV_EC2_PRIVATE_IP \
  'ls -lh /var/log/boot* /var/log/syslog /var/log/kern.log /var/log/auth.log 2>&1 && \
  echo \"=== SYSLOG HEAD ===\" && sudo head -3 /var/log/syslog 2>&1 && \
  echo \"=== KERN HEAD ===\" && sudo head -3 /var/log/kern.log 2>&1 && \
  echo \"=== BOOTS.LOG check ===\" && ls -la /var/log/boots.log 2>&1'"
-rw-r----- 1 syslog adm  5.2K Mar 25 11:49 /var/log/auth.log
-rw-r--r-- 1 root   root   23 Mar 25 11:36 /var/log/boots.log
-rw-r----- 1 syslog adm   54K Mar 25 11:35 /var/log/kern.log
-rw-r----- 1 syslog adm  135K Mar 25 11:49 /var/log/syslog
=== SYSLOG HEAD ===
Mar 25 11:35:41 ip-10-10-1-235 systemd[1]: Mounted Huge Pages File System.
Mar 25 11:35:41 ip-10-10-1-235 systemd[1]: Mounted POSIX Message Queue File System.
Mar 25 11:35:41 ip-10-10-1-235 systemd[1]: Mounted Kernel Debug File System.
=== KERN HEAD ===
Mar 25 11:35:41 ip-10-10-1-235 kernel: [    0.000000] Linux version 5.15.0-1084-aws (buildd@lcy02-amd64-055) (gcc (Ubuntu 9.4.0-1ubuntu1~20.04.2) 9.4.0, GNU ld (GNU Binutils for Ubuntu) 2.34) #91~20.04.1-Ubuntu SMP Fri May 2 06:59:36 UTC 2025 (Ubuntu 5.15.0-1084.91~20.04.1-aws 5.15.179)
Mar 25 11:35:41 ip-10-10-1-235 kernel: [    0.000000] Command line: BOOT_IMAGE=/boot/vmlinuz-5.15.0-1084-aws root=PARTUUID=cfbcdd53-02ad-4c2e-8163-3cd5c66e640a ro console=tty1 console=ttyS0 nvme_core.io_timeout=4294967295 panic=-1
Mar 25 11:35:41 ip-10-10-1-235 kernel: [    0.000000] KERNEL supported cpus:
=== BOOTS.LOG check ===
-rw-r--r-- 1 root root 23 Mar 25 11:36 /var/log/boots.log

~ on ☁️  (us-east-1) ➜  ssh -i /root/.ssh/devops-key.pem -o StrictHostKeyChecking=no -A ubuntu@$PUB_EC2_PUBLIC_IP \
  "ssh -o StrictHostKeyChecking=no ubuntu@$PRIV_EC2_PRIVATE_IP \
  'echo \"* * * * * scp -i /home/ubuntu/.ssh/devops-key.pem -o StrictHostKeyChecking=no /var/log/boots.log ubuntu@$PUB_EC2_PRIVATE_IP:/home/ubuntu/boots.log\" | crontab -'"

~ on ☁️  (us-east-1) ➜  ssh -i /root/.ssh/devops-key.pem -o StrictHostKeyChecking=no ubuntu@$PUB_EC2_PUBLIC_IP \
  "echo \"* * * * * /usr/local/bin/aws s3 cp /home/ubuntu/boots.log s3://devops-s3-logs-25406/devops-priv-vpc/boot/boots.log\" | crontab -"

~ on ☁️  (us-east-1) ➜  aws ec2 describe-vpc-peering-connections \
--vpc-peering-connection-ids $PEERING_ID \
--query 'VpcPeeringConnections[0].Status.Code'
"active"

~ on ☁️  (us-east-1) ➜  aws s3 ls s3://devops-s3-logs-25406/devops-priv-vpc/boot/
2026-03-25 12:02:03         23 boots.log

~ on ☁️  (us-east-1) ➜  
