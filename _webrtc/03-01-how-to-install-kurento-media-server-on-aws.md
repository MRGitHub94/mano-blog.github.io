---
title: "How to Install Kurento Media Server on AWS"
permalink: /webrtc/kurento/installing.html
excerpt: "on AWS"
header:
  overlay_image: /assets/images/webrtc/03-01-how-to-install-kureto-media-server-on-aws/03-02-how-to-install-kurento-media-server.jpg
  teaser: /assets/images/webrtc/03-01-how-to-install-kureto-media-server-on-aws/03-02-how-to-install-kurento-media-server.jpg
  overlay_filter: 0.5
last_modified_at: 2018-09-25T15:58:49-04:00
redirect_from:
  - /theme-setup/
toc: true
author: Jay
---
This is a slightly modified version of original Kurento Cloud Formation template which allows you to
point a domain to the server.

```bash
AWSTemplateFormatVersion: 2010-09-09
Description: Kurento Media Server CloudFormation template.

Parameters:
  KeyName:
    Description: Name of an existing EC2 key pair for SSH access to the EC2 instance.
    Type: AWS::EC2::KeyPair::KeyName
  SSHLocation:
    Description: The IP address range that can be used to SSH to the EC2 instances
    Type: String
    MinLength: '9'
    MaxLength: '18'
    Default: 0.0.0.0/0
    AllowedPattern: "(\\d{1,3})\\.(\\d{1,3})\\.(\\d{1,3})\\.(\\d{1,3})/(\\d{1,2})"
    ConstraintDescription: must be a valid IP CIDR range of the form x.x.x.x/x.
  InstanceType: 
    Description: Kurento Media Server EC2 instance type
    Type: String
    Default: t2.small
    AllowedValues: [t1.micro, t2.nano, t2.micro, t2.small, t2.medium, t2.large, m1.small,
      m1.medium, m1.large, m1.xlarge, m2.xlarge, m2.2xlarge, m2.4xlarge, m3.medium,
      m3.large, m3.xlarge, m3.2xlarge, m4.large, m4.xlarge, m4.2xlarge, m4.4xlarge,
      m4.10xlarge, c1.medium, c1.xlarge, c3.large, c3.xlarge, c3.2xlarge, c3.4xlarge,
      c3.8xlarge, c4.large, c4.xlarge, c4.2xlarge, c4.4xlarge, c4.8xlarge, g2.2xlarge,
      g2.8xlarge, r3.large, r3.xlarge, r3.2xlarge, r3.4xlarge, r3.8xlarge, i2.xlarge,
      i2.2xlarge, i2.4xlarge, i2.8xlarge, d2.xlarge, d2.2xlarge, d2.4xlarge, d2.8xlarge,
      hi1.4xlarge, hs1.8xlarge, cr1.8xlarge, cc2.8xlarge, cg1.4xlarge]
    ConstraintDescription: must be a valid EC2 instance type
  TurnUser:
    Description: User for turn server
    Type: String
    Default: kurento
  TurnPassword:
    Description: Password for turn user
    Type: String
    Default: kurento
    NoEcho: True
  HostedZoneName:
    Description: The route53 HostedZoneName. For example, "mydomain.com."  Don't forget the period at the end.
    Type: String
    Default: forvideostream.com.
  Subdomain:
    Description: The subdomain of the dns entry. For example, hello -> hello.mydomain.com, hello is the subdomain.
    Type: String
    Default: test
Mappings:
  RegionMap:
    us-east-1:
      AMI: ami-da05a4a0
    us-west-1:
      AMI: ami-1c1d217c
    ap-northeast-2:
      AMI: ami-7b1cb915
    ap-northeast-1:
      AMI: ami-15872773
    sa-east-1:
      AMI: ami-466b132a
    ap-southeast-1:
      AMI: ami-67a6e604
    ca-central-1:
      AMI: ami-8a71c9ee
    ap-southeast-2:
      AMI: ami-41c12e23
    us-west-2:
      AMI: ami-0a00ce72
    us-east-2:
      AMI: ami-336b4456
    ap-south-1:
      AMI: ami-bc0d40d3
    eu-central-1:
      AMI: ami-97e953f8
    eu-west-1:
      AMI: ami-add175d4
    eu-west-2:
      AMI: ami-ecbea388

Resources:
  KurentoMediaServer:
    Type: AWS::EC2::Instance
    Metadata:
      Comment: Install and configure KMS
      AWS::CloudFormation::Init:
        sources:
          - "/home/ubuntu/aws-cli": "https://github.com/aws/aws-cli/tarball/master"
        files:
          "/etc/cfn/cfn-hup.conf":
            content: !Sub |
              [main]
              stack=${AWS::StackId}
              region=${AWS::Region}
            mode: "000400"
            owner: "root"
            group: "root"
          "/etc/cfn/hooks.d/cfn-auto-reloader.conf":
            content: !Sub |
              [cfn-auto-reloader-hook]
              triggers=post.update
              path=Resources.KurentoMediaServer.Metadata.AWS::CloudFormation::Init
              action=/opt/aws/bin/cfn-init -v --stack ${AWS::StackName} --resource KurentoMediaServer --region ${AWS::Region}
            mode: "000400"
            owner: "root"
            group: "root"
    Properties:
      ImageId: !FindInMap [RegionMap, !Ref "AWS::Region", AMI]
      InstanceType: !Ref InstanceType
      KeyName: !Ref KeyName
      SecurityGroups:
        - !Ref KurentoMediaServerSecurityGroup
      UserData:
        "Fn::Base64": 
          !Sub |
            #!/bin/bash -xe
            sudo apt-get update
            # install coturn
            apt-get install -y coturn
            # install kms
            sudo apt-get update
            sudo apt-get install -y wget
            echo "deb http://ubuntu.kurento.org xenial kms6" | sudo tee /etc/apt/sources.list.d/kurento.list
            wget -O - http://ubuntu.kurento.org/kurento.gpg.key | sudo apt-key add -
            sudo apt-get update 
            sudo apt-get install -y kurento-media-server-6.0
            systemctl enable kurento-media-server-6.0
            # enable coturn
            sudo echo TURNSERVER_ENABLED=1 > /etc/default/coturn
            # turn config file
            sudo cat >/etc/turnserver.conf<<-EOF
            external-ip=PUBLIC_IP
            fingerprint
            user=${TurnUser}:${TurnPassword}
            lt-cred-mech
            realm=kurento.org
            log-file=/var/log/turnserver/turnserver.log
            simple-log
            EOF
            # install aws tools
            sudo apt-get update
            sudo apt-get -y install python-setuptools
            sudo easy_install https://s3.amazonaws.com/cloudformation-examples/aws-cfn-bootstrap-latest.tar.gz
            # creating launching script 
            sudo cat >/usr/local/bin/launch-kms.sh<<-EOF
            #!/bin/bash

            PUBLIC_IP=\$(curl http://169.254.169.254/latest/meta-data/public-ipv4)
            TURN_USER=${TurnUser}
            TURN_PASSWORD=${TurnPassword}

            systemctl stop kurento-media-server-6.0
            systemctl stop coturn
            sleep 5
            cat /etc/turnserver.conf | grep -v external-ip >/tmp/turnserver.conf
            echo "external-ip=\$PUBLIC_IP" >> /tmp/turnserver.conf
            mv /tmp/turnserver.conf /etc/turnserver.conf
            echo "turnURL=\$TURN_USER:\$TURN_PASSWORD@\$PUBLIC_IP:3478" > /etc/kurento/modules/kurento/WebRtcEndpoint.conf.ini
            sleep 5
            systemctl start coturn
            systemctl start kurento-media-server-6.0
            EOF
            sudo chmod 755 /usr/local/bin/launch-kms.sh
            # Preparing after reboot script
            sudo cat >/etc/rc.local<<-EOF
            #!/bin/sh -e
            /usr/local/bin/launch-kms.sh
            exit 0
            EOF
            echo "Launching Media Server..."
            sudo /usr/local/bin/launch-kms.sh
            # Start cfn-init
            sudo /usr/local/bin/cfn-init -s ${AWS::StackId} -r KurentoMediaServer --region ${AWS::Region}
            sudo /usr/local/bin/cfn-signal -e $? --stack ${AWS::StackId} --resource WaitCondition --region ${AWS::Region}

  KurentoMediaServerSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: !Sub 'KurentoSecurityGroup ${AWS::StackName}'
      GroupDescription: "Full Open + SSH access from specific location"
      SecurityGroupIngress:
      - CidrIp: 0.0.0.0/0
        FromPort: '0'
        ToPort: '65535'
        IpProtocol: tcp
      - CidrIp: 0.0.0.0/0
        FromPort: '0'
        ToPort: '65535'
        IpProtocol: udp
      - CidrIp: !Ref SSHLocation
        FromPort: '22'
        IpProtocol: tcp
        ToPort: '22'
  
  WaitCondition:
    Type: AWS::CloudFormation::WaitCondition
    CreationPolicy:
      ResourceSignal:
        Timeout: PT5M
        Count: 1

  DnsRecord:
    Type: AWS::Route53::RecordSet
    Properties:
      HostedZoneName: !Ref 'HostedZoneName'
      Comment: DNS name for my instance.
      Name: !Join ['', [!Ref 'Subdomain', ., !Ref 'HostedZoneName']]
      Type: A
      TTL: '900'
      ResourceRecords:
      - !GetAtt KurentoMediaServer.PublicIp

Outputs:
  InstanceId:
    Description: The instance ID of the Media Server
    Value:
      Ref: KurentoMediaServer
  KurentoURL:
    Value:
      !Sub 'ws://${KurentoMediaServer.PublicIp}:8888/kurento'
    Description: URL for newly created Kurento Media Server stack
  PublicDnsName:
    Description: Instance public DNS name
    Value: !GetAtt KurentoMediaServer.PublicDnsName
  PublicIP:
    Description: Public IP address of the Media Server
    Value:
      !GetAtt KurentoMediaServer.PublicIp
  TurnUserOutput:
    Description: Turn user 
    Value: !Ref TurnUser
  TurnPasswordOutput:
    Description: Turn Password
    Value: !Ref TurnPassword
```