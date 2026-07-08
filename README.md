# ReferralDB-CloudInfras

AWS Cloud Infrastructure to host a new secure database in YAML and imported using CloudFormation.

- Aurora SQL Database Cluster (MariaDB)
-  New VPC, subnets, and CIDR block, Route Table, and Security Groups
-  Key Management Services
-  Auto Scaling Groups
-  S3 Buckets and Endpoint
-  Linux EC2 Instances (PHP and Apache)
-  Environmental Variables
-  Application Load Balancer
-  Cloudfront Distribution
-  Route 53 A, and CNAME Records
-  AWs Certificate Manager
-  SNS Topics and SQS
-  SES Messaging for notification
-  Anti Virtus (Clam AV)


## Features
Quick Deploy of Infrastructure needed to host new referral web application for social workers and legal attorney across the u.s
- High Avialble Database
- Secure Network Infrastructure
- Scalaing capabilties
- Email Notifications
- Frontend Access with content delivery across the u.s


## Architectural and Data Flow Diagram

<img width="1536" height="1024" alt="ReferralDB" src="https://github.com/user-attachments/assets/86d8a2d5-50a5-44c8-b38c-8516898e8125" />


## iAC CloudFormation 

AWSTemplateFormatVersion: "2010-09-09"
Description: >
  PHP web app infra in us-east-2: VPC (2 public + 2 private subnets),
  public EC2 Auto Scaling (SSH/HTTP/HTTPS), Aurora Serverless v2 in private subnets,
  S3 uploads bucket, S3 Gateway VPC endpoint, internet-facing ALB, and EFS via Access Point.

Parameters:
  ProjectName:
    Type: String
    Default: "AcaciaReferralApp"
    Description: Project/stack name prefix used for tagging.
  VpcCidr:
    Type: String
    Default: 10.0.0.0/16
  AZ1:
    Type: AWS::EC2::AvailabilityZone::Name
    Default: us-east-2a
  AZ2:
    Type: AWS::EC2::AvailabilityZone::Name
    Default: us-east-2b
  PublicSubnet1Cidr:
    Type: String
    Default: 10.0.0.0/24
  PublicSubnet2Cidr:
    Type: String
    Default: 10.0.1.0/24
  PrivateSubnet1Cidr:
    Type: String
    Default: 10.0.10.0/24
  PrivateSubnet2Cidr:
    Type: String
    Default: 10.0.11.0/24
  AllowedSSHLocation:
    Type: String
    Default: 0.0.0.0/0
    Description: CIDR allowed to SSH into web servers (change from 0.0.0.0/0 for security).
  AllowedHTTPCIDR:
    Type: String
    Default: 0.0.0.0/0
    Description: CIDR allowed to access HTTP/HTTPS.
  KeyName:
    Type: AWS::EC2::KeyPair::KeyName
    Description: Existing EC2 KeyPair name for SSH.
  InstanceType:
    Type: String
    Default: t3.small
    AllowedValues:
      [t3.micro, t3.small, t3.medium, t3.large, t4g.small, t4g.medium]
    Description: EC2 instance type for web servers.
  DesiredCapacity:
    Type: Number
    Default: 2
  MinSize:
    Type: Number
    Default: 2
  MaxSize:
    Type: Number
    Default: 2
  DBName:
    Type: String
    Default: referrals
  DBUsername:
    Type: String
    Default: admin
  DBPassword:
    Type: String
    NoEcho: true
    MinLength: 8
    Description: Aurora DB user password.
  DBEngine:
    Type: String
    Default: aurora-mysql
    AllowedValues: [aurora-mysql, aurora-postgresql]
    Description: Aurora engine. Template uses Serverless v2.
  DBMinACU:
    Type: Number
    Default: 0.5
    Description: Serverless v2 min ACUs
  DBMaxACU:
    Type: Number
    Default: 4
    Description: Serverless v2 max ACUs
  DBKmsKeyArn:
    Type: String
    Default: ""
    Description: "Optional: ARN of an existing KMS key for Aurora at-rest encryption. If blank, a new CMK will be created."
  AL2023AmiId:
    Type: "AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>"
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64
  ALBCertificateArn:
    Type: String
    Default: ""
    Description: ACM cert ARN in us-east-2 for ALB. If blank only HTTP listener is created.
  NewUserName:
    Type: String
    Default: "what ever you want"
  NewUserPassword:
    Type: String
    NoEcho: true
    Default: "whatever you want"
    MinLength: 8
  EnablePasswordSSH:
    Type: String
    AllowedValues: ["true", "false"]
    Default: "true"
  EfsFileSystemId:
    Type: String
    Description: "EFS file system ID (e.g., fs-0abc1234def567890)."
  EfsAccessPointId:
    Type: String
    Description: "EFS access point ID (e.g., fsap-0123456789abcdef0)."
  EfsSecurityGroupId:
    Type: String
    Default: ""
    Description: "Security Group on EFS mount targets (TCP/2049). Optional."
  AppDomain:
    Type: String
    Default: your MS Tenant/Domain 
  UploadsBucketParam:
    Type: String
    Default: location of your s3 bucket
  S3UploadsPrefix:
    Type: String
    Default: referrals
  SesToNewReferral:
    Type: String
    Default: email used for ses messaging
  SesFrom:
    Type: String
    Default: email used for ses messaging
  AdminNotify:
    Type: String
    Default: email used for ses messaging
  SesTo:
    Type: String
    Default: email used for ses messaging
  SesCc:
    Type: String
    Default: email used for ses messaging
  SesConfigSet:
    Type: String
    Default: ""
  DashboardUrl:
    Type: String
    Default: "you subdomain name for your app"
  SesSmtpUser:
    Type: String
    NoEcho: true
    Default: ""
  SesSmtpPass:
    Type: String
    NoEcho: true
    Default: ""
  DbHost:
    Type: String
    Default: ""
  DbUser:
    Type: String
    Default: acacia
  DbPass:
    Type: String
    NoEcho: true
    Default: ""
  RecaptchaSecret:
    Type: String
    NoEcho: true
    Default: ""
    Description: "Google reCAPTCHA secret (server-side)."
  RecaptchaSiteKey:
    Type: String
    Default: ""
    Description: "Google reCAPTCHA site key (client-side)."

Conditions:
  IsMySQL: !Equals [!Ref DBEngine, aurora-mysql]
  HasALBCert: !Not [!Equals [!Ref ALBCertificateArn, ""]]
  NoALBCert: !Equals [!Ref ALBCertificateArn, ""]
  HasEfsSG: !Not [!Equals [!Ref EfsSecurityGroupId, ""]]
  UseProvidedKms: !Not [!Equals [!Ref DBKmsKeyArn, ""]]
  CreateNewKms: !Equals [!Ref DBKmsKeyArn, ""]

#---future add new subnet for VPN access
Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCidr
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags: [{ Key: Name, Value: !Sub "${ProjectName}-vpc" }]

  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags: [{ Key: Name, Value: !Sub "${ProjectName}-igw" }]

  VPCGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      InternetGatewayId: !Ref InternetGateway
      VpcId: !Ref VPC

  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Ref AZ1
      CidrBlock: !Ref PublicSubnet1Cidr
      MapPublicIpOnLaunch: true
      Tags: [{ Key: Name, Value: !Sub "${ProjectName}-pub-a" }]

  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Ref AZ2
      CidrBlock: !Ref PublicSubnet2Cidr
      MapPublicIpOnLaunch: true
      Tags: [{ Key: Name, Value: !Sub "${ProjectName}-pub-b" }]

  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Ref AZ1
      CidrBlock: !Ref PrivateSubnet1Cidr
      MapPublicIpOnLaunch: false
      Tags: [{ Key: Name, Value: !Sub "${ProjectName}-priv-a" }]

  PrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Ref AZ2
      CidrBlock: !Ref PrivateSubnet2Cidr
      MapPublicIpOnLaunch: false
      Tags: [{ Key: Name, Value: !Sub "${ProjectName}-priv-b" }]

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags: [{ Key: Name, Value: !Sub "${ProjectName}-public-rt" }]

  PublicDefaultRoute:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  PublicRTAssoc1:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet1
      RouteTableId: !Ref PublicRouteTable

  PublicRTAssoc2:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet2
      RouteTableId: !Ref PublicRouteTable

  PrivateRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags: [{ Key: Name, Value: !Sub "${ProjectName}-private-rt" }]

  PrivateRTAssoc1:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnet1
      RouteTableId: !Ref PrivateRouteTable

  PrivateRTAssoc2:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnet2
      RouteTableId: !Ref PrivateRouteTable

  S3GatewayEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      ServiceName: !Sub "com.amazonaws.${AWS::Region}.s3"
      VpcId: !Ref VPC
      RouteTableIds: [!Ref PrivateRouteTable]
      VpcEndpointType: Gateway

  ALBSG: #temporary will scope usuage and limits
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: ALB SG - allow 80/443 from internet
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: !Ref AllowedHTTPCIDR
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: !Ref AllowedHTTPCIDR
      SecurityGroupEgress:
        - IpProtocol: -1
          CidrIp: 0.0.0.0/0
      Tags: [{ Key: Name, Value: !Sub "${ProjectName}-alb-sg" }]

  WebSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Web SG - allow HTTP/HTTPS from ALB and optional SSH
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: !Ref AllowedSSHLocation
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          SourceSecurityGroupId: !Ref ALBSG
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          SourceSecurityGroupId: !Ref ALBSG
      SecurityGroupEgress:
        - IpProtocol: -1
          CidrIp: 0.0.0.0/0
      Tags: [{ Key: Name, Value: !Sub "${ProjectName}-web-sg" }]

  WebFromEfsIngress:
    Condition: HasEfsSG
    Type: AWS::EC2::SecurityGroupIngress
    Properties:
      GroupId: !Ref WebSG
      IpProtocol: tcp
      FromPort: 2049
      ToPort: 2049
      SourceSecurityGroupId: !Ref EfsSecurityGroupId

  DBSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: DB SG - allow 3306 from Web SG only
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 3306
          ToPort: 3306
          SourceSecurityGroupId: !Ref WebSG
      SecurityGroupEgress:
        - IpProtocol: -1
          CidrIp: 0.0.0.0/0
      Tags: [{ Key: Name, Value: !Sub "${ProjectName}-db-sg" }]

  EfsFromWebIngress:
    Condition: HasEfsSG
    Type: AWS::EC2::SecurityGroupIngress
    Properties:
      GroupId: !Ref EfsSecurityGroupId
      IpProtocol: tcp
      FromPort: 2049
      ToPort: 2049
      SourceSecurityGroupId: !Ref WebSG
  #for phase 2 archiving needs - scope usuage
  UploadsBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain
    UpdateReplacePolicy: Retain
    Properties:
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault: { SSEAlgorithm: AES256 }
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
      VersioningConfiguration: { Status: Enabled }
      Tags: [{ Key: Name, Value: !Sub "${ProjectName}-lasrf-uploads-2025" }]

  WebInstanceRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal: { Service: ec2.amazonaws.com }
            Action: sts:AssumeRole
      ManagedPolicyArns: []
      Policies:
        - PolicyName: S3UploadsAccess
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Sid: ListBucket
                Effect: Allow
                Action: s3:ListBucket
                Resource: !Sub "arn:aws:s3:::${UploadsBucket}"
              - Sid: ObjectRW
                Effect: Allow
                Action:
                  [
                    s3:GetObject,
                    s3:PutObject,
                    s3:DeleteObject,
                    s3:AbortMultipartUpload,
                  ]
                Resource: !Sub "arn:aws:s3:::${UploadsBucket}/*"
        - PolicyName: EfsAccessPointMount
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: Allow
                Action:
                  [elasticfilesystem:ClientMount, elasticfilesystem:ClientWrite]
                Resource: !Sub arn:aws:elasticfilesystem:${AWS::Region}:${AWS::AccountId}:access-point/${EfsAccessPointId}
              - Effect: Allow
                Action:
                  - elasticfilesystem:DescribeFileSystems
                  - elasticfilesystem:DescribeMountTargets
                  - elasticfilesystem:DescribeAccessPoints
                Resource: "*"
        - PolicyName: SesSendEmail
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: Allow
                Action:
                  - ses:SendEmail
                  - ses:SendRawEmail
                Resource: "*"
      Tags: [{ Key: Name, Value: !Sub "${ProjectName}-web-role" }]

  WebInstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Roles: [!Ref WebInstanceRole]

  WebLaunchTemplate:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateName: !Sub "${AWS::StackName}-web-lt"
      LaunchTemplateData:
        ImageId: !Ref AL2023AmiId
        InstanceType: !Ref InstanceType
        IamInstanceProfile: { Arn: !GetAtt WebInstanceProfile.Arn }
        KeyName: !Ref KeyName
        DisableApiTermination: false
        NetworkInterfaces:
          - AssociatePublicIpAddress: true
            DeviceIndex: 0
            Groups: [!Ref WebSG]
        TagSpecifications:
          - ResourceType: instance
            Tags: [{ Key: Name, Value: !Sub "${ProjectName}-web" }]
        UserData:
          Fn::Base64:
            Fn::Sub:
              - |
                #!/bin/bash
                set -euxo pipefail
                REGION='${AWS::Region}'
                ALB_DNS='${AlbDnsName}'
                APP_DOMAIN='${AppDomain}'
                UPLOADS_BUCKET='${UploadsBucketParam}'
                S3_PREFIX='${S3UploadsPrefix}'
                dnf -y update
                dnf -y install \
                httpd mod_ssl openssl amazon-efs-utils ec2-instance-connect \
                php php-cli php-common php-mysqlnd php-json php-mbstring php-xml php-gd php-intl php-zip php-curl \
                unzip fontconfig dejavu-sans-fonts dejavu-serif-fonts dejavu-sans-mono-fonts || true
                NEW_USER='${NewUserName}'
                NEW_PASS='${NewUserPassword}'
                if [ -n "$NEW_USER" ]; then
                  id "$NEW_USER" >/dev/null 2>&1 || useradd -m -s /bin/bash "$NEW_USER"
                  if [ -n "$NEW_PASS" ]; then
                    echo "$NEW_USER:$NEW_PASS" | chpasswd
                    passwd -u "$NEW_USER" || true
                    chage -I -1 -m 0 -M 3650 -E -1 "$NEW_USER" || true
                  fi
                  usermod -aG wheel "$NEW_USER" || true
                fi
                systemctl stop amazon-ssm-agent || true
                systemctl disable amazon-ssm-agent || true
                rpm -q amazon-ssm-agent && dnf -y remove amazon-ssm-agent || true
                id ssm-user && userdel -r ssm-user || true
                rm -f /etc/sudoers.d/ssm-agent-users || true
                rm -f /etc/ssh/sshd_config.d/10-disable-ec2-instance-connect.conf || true
                mkdir -p /etc/ssh/sshd_config.d
                cat >/etc/ssh/sshd_config.d/50-ec2-instance-connect.conf <<'EOF'
                AuthorizedKeysCommand /usr/bin/eic_run_authorized_keys
                AuthorizedKeysCommandUser ec2-instance-connect
                EOF
                cat >/etc/ssh/sshd_config.d/60-password-auth.conf <<'EOF'
                PasswordAuthentication yes
                KbdInteractiveAuthentication yes
                UsePAM yes
                PubkeyAuthentication yes
                ChallengeResponseAuthentication no
                PermitRootLogin prohibit-password
                ClientAliveInterval 20
                ClientAliveCountMax 3
                EOF
                sysctl -w net.ipv4.tcp_keepalive_time=60
                sysctl -w net.ipv4.tcp_keepalive_intvl=20
                sysctl -w net.ipv4.tcp_keepalive_probes=3
                grep -q tcp_keepalive_time /etc/sysctl.conf || cat >> /etc/sysctl.conf <<'EOF'
                net.ipv4.tcp_keepalive_time=60
                net.ipv4.tcp_keepalive_intvl=20
                net.ipv4.tcp_keepalive_probes=3
                EOF
                systemctl reload sshd || true
                mkdir -p /var/www/html
                systemctl enable --now httpd
                CERT_DIR="/etc/pki/tls/certs"
                KEY_DIR="/etc/pki/tls/private"
                install -d -m 0755 "$CERT_DIR" "$KEY_DIR"
                if [ ! -s "$CERT_DIR/server.crt" ] || [ ! -s "$KEY_DIR/server.key" ]; then
                  openssl req -x509 -nodes -newkey rsa:2048 \
                    -keyout "$KEY_DIR/server.key" \
                    -out "$CERT_DIR/server.crt" \
                    -days 365 \
                    -subj "/C=US/ST=NA/L=NA/O=App/CN=$ALB_DNS"
                  chmod 600 "$KEY_DIR/server.key"
                fi
                cat >/etc/httpd/conf.d/00-redirect.conf <<EOF
                <VirtualHost *:80>
                  ServerName ${AlbDnsName}
                  ServerAlias ${AlbDnsName}
                  RewriteEngine On
                  RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]
                </VirtualHost>
                EOF
                cat >/etc/httpd/conf.d/10-app-ssl.conf <<'EOF'
                <VirtualHost *:443>
                  ServerName ${AlbDnsName}
                  ServerAlias ${AppDomain} localhost 127.0.0.1 ${AlbDnsName}
                  RewriteEngine On
                  RewriteRule ^/?$ /webform [R=301,L]
                  SetEnv AWS_REGION ${AWS::Region}
                  SetEnv UPLOADS_BUCKET ${UploadsBucketParam}
                  SetEnv S3_UPLOADS_BUCKET ${UploadsBucketParam}
                  SetEnv S3_UPLOADS_PREFIX ${S3UploadsPrefix}
                  SetEnv S3_SSE aes256
                  SetEnv SES_ENABLED 1
                  SetEnv SES_TO_NEW_REFERRAL "${SesToNewReferral}"
                  SetEnv SES_FROM ${SesFrom}
                  SetEnv ADMIN_NOTIFY ${AdminNotify}
                  SetEnv SES_TO "${SesTo}"
                  SetEnv SES_CC "${SesCc}"
                  SetEnv SES_CONFIGSET "${SesConfigSet}"
                  SetEnv DASHBOARD_URL "${DashboardUrl}"
                  SetEnv SES_SMTP_USER ${SesSmtpUser}
                  SetEnv SES_SMTP_PASS ${SesSmtpPass}
                  SetEnv DB_HOST ${DbHost}
                  SetEnv DB_NAME ${DBName}
                  SetEnv DB_USER ${DbUser}
                  SetEnv DB_PASS ${DbPass}
                  SetEnv RECAPTCHA_SECRET ${RecaptchaSecret}
                  SetEnv RECAPTCHA_SITE_KEY ${RecaptchaSiteKey}
                  php_admin_flag log_errors On
                  php_admin_value error_log "/var/log/httpd/php_errors.log"
                  php_admin_value memory_limit "512M"
                  SetEnv APP_ENV dev
                  PassEnv AWS_REGION UPLOADS_BUCKET S3_UPLOADS_BUCKET S3_UPLOADS_PREFIX S3_SSE
                  PassEnv SES_ENABLED SES_TO_NEW_REFERRAL SES_FROM ADMIN_NOTIFY SES_TO SES_CC SES_CONFIGSET DASHBOARD_URL
                  PassEnv DB_NAME DB_USER DB_PASS APP_ENV SES_SMTP_USER SES_SMTP_PASS
                  PassEnv RECAPTCHA_SECRET
                  PassEnv RECAPTCHA_SITE_KEY
                  SSLEngine on
                  SSLCertificateFile "/etc/pki/tls/certs/server.crt"
                  SSLCertificateKeyFile "/etc/pki/tls/private/server.key"
                  SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1
                  SSLCipherSuite HIGH:!aNULL:!MD5
                  SSLHonorCipherOrder on
                  Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
                  DocumentRoot "/var/www/html"
                  DirectoryIndex index.php index.html
                  <Directory "/var/www/html">
                    Options -Indexes +FollowSymLinks -MultiViews
                    AllowOverride All
                    Require all granted
                    Header always set X-Content-Type-Options "nosniff"
                    Header always set X-Frame-Options "SAMEORIGIN"
                    Header always set Referrer-Policy "strict-origin-when-cross-origin"
                    Header always set Permissions-Policy "geolocation=(), camera=(), microphone=()"
                  </Directory>
                  <Location "/uploads/presign.php">
                    <LimitExcept GET POST OPTIONS>
                      Require all denied
                    </LimitExcept>
                    Header always set Cache-Control "no-store, no-cache, must-revalidate, max-age=0"
                    Header always set Pragma "no-cache"
                    Header always set Expires "0"
                    Header always set Access-Control-Expose-Headers "ETag, x-amz-request-id, x-amz-id-2"
                  </Location>
                  LimitRequestBody 10485760
                  ErrorLog  "/var/log/httpd/portal-error.log"
                  CustomLog "/var/log/httpd/portal-access.log" combined
                </VirtualHost>
                EOF
                FS_ID='${EfsFileSystemId}'
                AP_ID='${EfsAccessPointId}'
                MNT='/var/www/html'
                mkdir -p "$MNT"
                FSTAB_LINE="$FS_ID:/  $MNT  efs  _netdev,tls,iam,accesspoint=$AP_ID,noresvport  0  0"
                grep -q "accesspoint=$AP_ID " /etc/fstab || echo "$FSTAB_LINE" >> /etc/fstab
                attempt_mount() {
                  if command -v mount.efs >/dev/null 2>&1; then
                    mount -t efs -o tls,iam,accesspoint="$AP_ID",noresvport "$FS_ID":/ "$MNT"
                  else
                    dnf -y install nfs-utils || true
                    mount -t nfs4 -o nfsvers=4.1 "$FS_ID.efs.$REGION.amazonaws.com:/" "$MNT"
                  fi
                }
                for i in {1..10}; do
                  mountpoint -q "$MNT" && break
                  attempt_mount || true
                  mountpoint -q "$MNT" && break
                  sleep 15
                done
                if id apache >/dev/null 2>&1; then
                  chown apache:apache "$MNT"; WEB_UID_GID="48:48"; WEB_USER="apache"
                elif id www-data >/dev/null 2>&1; then
                  chown www-data:www-data "$MNT"; WEB_UID_GID="33:33"; WEB_USER="www-data"
                else
                  WEB_UID_GID="0:0"; WEB_USER="root"
                fi
                chmod 0755 "$MNT"
                if [ ! -f "$MNT/index.php" ]; then
                  echo '<?php phpinfo(); ?>' > "$MNT/index.php"
                fi
                if ! command -v composer >/dev/null 2>&1; then
                  php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
                  php composer-setup.php --install-dir=/usr/local/bin --filename=composer
                  php -r "unlink('composer-setup.php');"
                fi
                chmod 0755 /usr/local/bin/composer
                cd "$MNT"
                export COMPOSER_HOME="$MNT/.composer"
                if ! grep -q phpoffice/phpspreadsheet "$MNT/composer.lock" 2>/dev/null || \
                   ! grep -q dompdf/dompdf "$MNT/composer.lock" 2>/dev/null; then
                  su -s /bin/bash "$WEB_USER" -c "cd '$MNT' && COMPOSER_HOME='$MNT/.composer' /usr/local/bin/composer require phpoffice/phpspreadsheet:^1.29 dompdf/dompdf:^2.0 --no-dev -o --no-interaction --ansi=never" || true
                else
                  su -s /bin/bash "$WEB_USER" -c "cd '$MNT' && COMPOSER_HOME='$MNT/.composer' /usr/local/bin/composer install --no-dev -o --no-interaction --ansi=never" || true
                fi
                chown -R $WEB_UID_GID "$MNT/vendor" "$MNT/.composer" || true
                mkdir -p /var/log/clamav
                UID_VAL="$(echo "$WEB_UID_GID" | cut -d: -f1)"
                GID_VAL="$(echo "$WEB_UID_GID" | cut -d: -f2)"
                chown -R "$UID_VAL:$GID_VAL" /var/log/clamav || true
                systemctl enable --now clamav-freshclam || systemctl enable --now freshclam || true
                cat >/usr/local/sbin/scan-webroot.sh <<'EOSCAN'
                #!/bin/bash
                su -s /bin/bash "$WEB_USER" -c "cd '$MNT' && COMPOSER_HOME='$MNT/.composer' /usr/local/bin/composer require aws/aws-sdk-php:^3 phpmailer/phpmailer:^6.9 --no-dev -o --no-interaction --ansi=never" || true
                chown -R $WEB_UID_GID "$MNT/vendor" "$MNT/.composer" || true
                #!/bin/bash
                set -euo pipefail
                TARGET="/var/www/html"
                if [ $# -gt 0 ] && [ -n "$1" ]; then TARGET="$1"; fi
                LOGDIR="/var/log/clamav"
                mkdir -p "$LOGDIR"
                LOG="$LOGDIR/webroot-scan-$(date +%F).log"
                echo "[$(date -Is)] Starting ClamAV scan of $TARGET" | tee -a "$LOG"
                clamscan -ri --exclude-dir='^/proc' --exclude-dir='^/sys' "$TARGET" | tee -a "$LOG" || true
                echo "[$(date -Is)] Scan complete" | tee -a "$LOG"
                EOSCAN
                chmod +x /usr/local/sbin/scan-webroot.sh
                cat >/etc/systemd/system/clamav-webroot-scan.service <<'EOSVC'
                [Unit]
                Description=Weekly ClamAV scan of web root (/var/www/html)
                After=network-online.target
                ConditionPathIsDirectory=/var/www/html
                [Service]
                Type=oneshot
                ExecStart=/usr/local/sbin/scan-webroot.sh /var/www/html
                EOSVC
                cat >/etc/systemd/system/clamav-webroot-scan.timer <<'EOTMR'
                [Unit]
                Description=Run ClamAV webroot scan weekly
                [Timer]
                OnCalendar=Sun *-*-* 03:00:00
                Persistent=true
                [Install]
                WantedBy=timers.target
                EOTMR
                systemctl daemon-reload
                systemctl enable --now clamav-webroot-scan.timer
                touch /var/log/httpd/php_errors.log
                chown apache:apache /var/log/httpd/php_errors.log || true
                apachectl configtest
                systemctl restart httpd
              - {
                  AlbDnsName: !GetAtt ALB.DNSName,
                  AppDomain: !Ref AppDomain,
                  UploadsBucketParam: !Ref UploadsBucket,
                  S3UploadsPrefix: !Ref S3UploadsPrefix,
                  SesToNewReferral: !Ref SesToNewReferral,
                  SesFrom: !Ref SesFrom,
                  AdminNotify: !Ref AdminNotify,
                  SesTo: !Ref SesTo,
                  SesCc: !Ref SesCc,
                  SesConfigSet: !Ref SesConfigSet,
                  DashboardUrl: !Ref DashboardUrl,
                  SesSmtpUser: !Ref SesSmtpUser,
                  SesSmtpPass: !Ref SesSmtpPass,
                  DbHost: !Ref DbHost,
                  DbUser: !Ref DbUser,
                  DbPass: !Ref DbPass,
                  NewUserName: !Ref NewUserName,
                  NewUserPassword: !Ref NewUserPassword,
                  RecaptchaSecret: !Ref RecaptchaSecret,
                }

  WebAutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    DeletionPolicy: Retain
    UpdateReplacePolicy: Retain
    Properties:
      VPCZoneIdentifier: [!Ref PublicSubnet1, !Ref PublicSubnet2]
      MinSize: !Ref MinSize
      MaxSize: !Ref MaxSize
      DesiredCapacity: !Ref DesiredCapacity
      HealthCheckType: ELB
      HealthCheckGracePeriod: 180
      TargetGroupARNs: [!Ref WebTargetGroup]
      NewInstancesProtectedFromScaleIn: true
      LaunchTemplate:
        LaunchTemplateId: !Ref WebLaunchTemplate
        Version: !GetAtt WebLaunchTemplate.LatestVersionNumber
      Tags:
        - Key: Name
          Value: !Sub "${ProjectName}-web-asg"
          PropagateAtLaunch: true
    UpdatePolicy:
      AutoScalingRollingUpdate:
        MinInstancesInService: 1
        MaxBatchSize: 1
        PauseTime: PT5M
        WaitOnResourceSignals: false

  WebTargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      VpcId: !Ref VPC
      TargetType: instance
      Protocol: HTTPS
      Port: 443
      HealthCheckProtocol: HTTPS
      HealthCheckPath: /index.php
      Matcher: { HttpCode: "200-399" }
      Tags: [{ Key: Name, Value: !Sub "${ProjectName}-tg" }]

  ALB:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: !Sub "${ProjectName}-alb"
      Scheme: internet-facing
      Type: application
      Subnets: [!Ref PublicSubnet1, !Ref PublicSubnet2]
      SecurityGroups: [!Ref ALBSG]
      Tags: [{ Key: Name, Value: !Sub "${ProjectName}-alb" }]

  HTTPListenerRedirect:
    Condition: HasALBCert
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref ALB
      Port: 80
      Protocol: HTTP
      DefaultActions:
        - Type: redirect
          RedirectConfig:
            Protocol: HTTPS
            Port: "443"
            StatusCode: HTTP_301

  HTTPListenerForward:
    Condition: NoALBCert
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref ALB
      Port: 80
      Protocol: HTTP
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref WebTargetGroup

  HTTPSListener:
    Condition: HasALBCert
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref ALB
      Port: 443
      Protocol: HTTPS
      Certificates:
        - CertificateArn: !Ref ALBCertificateArn
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref WebTargetGroup

  # ===== NEW: ALB listener rules to redirect "/" -> "/webform" =====
  RootToWebformHTTPSRule:
    Condition: HasALBCert
    Type: AWS::ElasticLoadBalancingV2::ListenerRule
    Properties:
      ListenerArn: !Ref HTTPSListener
      Priority: 1
      Actions:
        - Type: redirect
          RedirectConfig:
            Protocol: "#{protocol}"
            Port: "#{port}"
            Host: "#{host}"
            Path: "/webform"
            Query: "#{query}"
            StatusCode: HTTP_301
      Conditions:
        - Field: path-pattern
          PathPatternConfig:
            Values: ["/"]

  RootToWebformHTTPRule:
    Condition: NoALBCert
    Type: AWS::ElasticLoadBalancingV2::ListenerRule
    Properties:
      ListenerArn: !Ref HTTPListenerForward
      Priority: 1
      Actions:
        - Type: redirect
          RedirectConfig:
            Protocol: "#{protocol}"
            Port: "#{port}"
            Host: "#{host}"
            Path: "/webform/index.php"
            Query: "#{query}"
            StatusCode: HTTP_301
      Conditions:
        - Field: path-pattern
          PathPatternConfig:
            Values: ["/"]

  DBSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: Private subnets for Aurora
      SubnetIds: [!Ref PrivateSubnet1, !Ref PrivateSubnet2]
      DBSubnetGroupName: !Sub "${ProjectName}-dbsubnets"

  DBKmsKey:
    Condition: CreateNewKms
    Type: AWS::KMS::Key
    Properties:
      Description: !Sub "${ProjectName} Aurora at-rest encryption key"
      EnableKeyRotation: true
      KeyPolicy:
        Version: "2012-10-17"
        Statement:
          - Sid: AllowRootAccount
            Effect: Allow
            Principal:
              AWS: !Sub arn:aws:iam::${AWS::AccountId}:root
            Action: "kms:*"
            Resource: "*"

  DBKmsKeyAlias:
    Condition: CreateNewKms
    Type: AWS::KMS::Alias
    Properties:
      AliasName: !Sub "alias/${ProjectName}-aurora"
      TargetKeyId: !Ref DBKmsKey

  DBCluster:
    Type: AWS::RDS::DBCluster
    DeletionPolicy: Snapshot
    UpdateReplacePolicy: Snapshot
    Properties:
      Engine: !Ref DBEngine
      DBSubnetGroupName: !Ref DBSubnetGroup
      VpcSecurityGroupIds: [!Ref DBSG]
      DatabaseName: !Ref DBName
      MasterUsername: !Ref DBUsername
      MasterUserPassword: !Ref DBPassword
      StorageEncrypted: true
      KmsKeyId: !If [UseProvidedKms, !Ref DBKmsKeyArn, !GetAtt DBKmsKey.Arn]
      BackupRetentionPeriod: 7
      CopyTagsToSnapshot: true
      ServerlessV2ScalingConfiguration:
        MinCapacity: !Ref DBMinACU
        MaxCapacity: !Ref DBMaxACU
      Tags: [{ Key: Name, Value: !Sub "${ProjectName}-aurora-cluster" }]

  DBInstance:
    Type: AWS::RDS::DBInstance
    Properties:
      DBClusterIdentifier: !Ref DBCluster
      DBInstanceClass: db.serverless
      Engine: !Ref DBEngine
      PubliclyAccessible: false
      Tags: [{ Key: Name, Value: !Sub "${ProjectName}-aurora-instance" }]

Outputs:
  VpcId:
    Value: !Ref VPC
    Export: { Name: !Sub "${AWS::StackName}-VpcId" }
  PublicSubnets:
    Value: !Join [",", [!Ref PublicSubnet1, !Ref PublicSubnet2]]
  PrivateSubnets:
    Value: !Join [",", [!Ref PrivateSubnet1, !Ref PrivateSubnet2]]
  WebSecurityGroupId:
    Value: !Ref WebSG
  DBSecurityGroupId:
    Value: !Ref DBSG
  UploadsBucketName:
    Value: !Ref UploadsBucket
  S3VpcEndpointId:
    Value: !Ref S3GatewayEndpoint
  AutoScalingGroupName:
    Value: !Ref WebAutoScalingGroup
  ALBArn:
    Value: !Ref ALB
  ALBDNSName:
    Value: !GetAtt ALB.DNSName
  TargetGroupArn:
    Value: !Ref WebTargetGroup
  DBClusterEndpoint:
    Value: !GetAtt DBCluster.Endpoint.Address
  DBReaderEndpoint:
    Value: !GetAtt DBCluster.ReadEndpoint.Address
  DBEncryptionKeyArn:
    Condition: CreateNewKms
    Value: !GetAtt DBKmsKey.Arn
