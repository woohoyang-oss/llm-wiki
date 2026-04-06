# EC2 스냅샷 생성 방법

## 사전 준비
- AWS CLI 설치 필요 (/opt/homebrew/bin/aws)
- 프로필: tbnws-ec2 (ap-northeast-2)

## 1. 인스턴스/볼륨 조회
```bash
aws ec2 describe-instances --profile tbnws-ec2 \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name,Tags[?Key==`Name`].Value|[0],BlockDeviceMappings[*].[DeviceName,Ebs.VolumeId]]' \
  --output json
```

## 2. 스냅샷 생성
```bash
aws ec2 create-snapshot --profile tbnws-ec2 \
  --volume-id <VOLUME_ID> \
  --description "<NAME> snapshot <DATE>" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=<NAME>-snapshot-<DATE>}]' \
  --output json
```

## 현재 인스턴스 목록 (2026-03-23 기준)
| 이름 | 인스턴스 ID | 볼륨 ID | 상태 |
|------|-----------|---------|------|
| auto-email-poc | i-0745490e05c7e0259 | vol-0dff8e880ea6b9d45 | running |
| vpn-korea | i-02eb1fbbaead06b1c | vol-0272092767cb77cd3 | stopped |
| akaxa-space | i-0296ae507d8c67f42 | vol-01bebbfd990347ebb | running |
| claude | i-03115fe75a57de373 | vol-03fc624f4a1de7193 | running |
