# 📗 08.19 AWS
## *.log 파일 S3에 자동 적재
### 1. 로그 수집 스크립트 : /home/ec2-user/log_collector.sh
```bash
#!/bin/bash
LOG_DIR="/var/log/mylogs"
mkdir -p "$LOG_DIR"

while true
do
  LOG_FILE="$LOG_DIR/$(date +%Y%m%d-%H%M).log"
  NOW=$(date +"%H:%M:%S")

  CPU=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d. -f1)
  MEM=$(free | awk '/Mem/{printf "%.0f", $3/$2*100}')
  DISK=$(df / | tail -1 | awk '{print $5}' | sed 's/%//')

  echo "$NOW CPU:${CPU}% MEM:${MEM}% DISK:${DISK}%" >> "$LOG_FILE"
  sleep 5
done
```
- 권한 부여
```bash
sudo mkdir -p /var/log/mylogs
sudo chown ec2-user:ec2-user /var/log/mylogs
chmod +x /home/ec2-user/log_collector.sh
```
---
### 2. 로그 업로드 스크립트 + Slack 알림 : `/home/ec2-user/log_uploader.sh`
```bash
#!/bin/bash
set -Eeuo pipefail

LOG_DIR="/var/log/mylogs"
BUCKET="s3://sample-psj-s3/logs"
AWS_CLI="/usr/bin/aws"
CURL="/usr/bin/curl"
AWS_REGION="ap-northeast-2"
SLACK_WEBHOOK="Slack URL"

notify_slack() {
  local msg="$1"
  $CURL -s -X POST -H 'Content-type: application/json' \
    --data "{\"text\":\"${msg}\"}" "$SLACK_WEBHOOK" >/dev/null || true
}

PREV_MINUTE="$(date -d '1 minute ago' +%Y%m%d-%H%M)"
file="$LOG_DIR/$PREV_MINUTE.log"
[ -f "$file" ] || exit 0
filename="$(basename "$file")"

n=0
until [ $n -ge 3 ]; do
  if $AWS_CLI s3 cp "$file" "$BUCKET/$filename" --region "$AWS_REGION" --only-show-errors; then
    rm -f "$file"
    notify_slack "✅ 업로드 성공 : $filename → sample-psj-s3"
    exit 0
  fi
  n=$((n+1)); sleep 5
done

echo "$filename 파일 s3 업로드에 실패했습니다."
notify_slack "❌ 업로드 실패 : $filename (3회 재시도 후 실패)"
```
- 권한 부여
```bash
chmod +x /home/ec2-user/log_uploader.sh
```
---
### 3. user systemd 유닛 파일 경로 : `~/.config/systemd/user/`
- 경로 생성
```bash
mkdir -p /home/ec2-user/.config/systemd/user
```
### 3-1. 수집기 서비스 : `~/.config/systemd/user/log_collector.service`
```bash
[Unit]
Description=Log Collector Service
After=network.target

[Service]
ExecStart=/home/ec2-user/log_collector.sh
Restart=always

[Install]
WantedBy=default.target
```
### 3-2. 업로더 서비스 : `~/.config/systemd/user/log_uploader.service`
```bash
[Unit]
Description=Log Uploader Service

[Service]
Type=oneshot
ExecStart=/home/ec2-user/log_uploader.sh
```
### 3-3. 업로더 타이머 : `~/.config/systemd/user/log_uploader.timer`
```bash
[Unit]
Description=Run log uploader every 1 minute

[Timer]
OnCalendar=*:0/1
Persistent=true

[Install]
WantedBy=timers.target
```
---
### 4. systemd 활성화 : user 모드
```bash
systemctl --user daemon-reload
systemctl --user enable --now log_collector.service
systemctl --user enable --now log_uploader.timer
```
- 부팅 후 자동 실행 유지
```bash
sudo loginctl enable-linger ec2-user
```
---
### 5. 확인 방법
```bash
# 수집기 상태
systemctl --user status log_collector.service

# 업로더 타이머 스케줄
systemctl --user list-timers | grep log_uploader

# 업로드 즉시 테스트
systemctl --user start log_uploader.service

# 업로드 서비스 로그
journalctl --user -u log_uploader.service -n 50 --no-pager

# S3 업로드 확인
aws s3 ls s3://sample-psj-s3/ --region ap-northeast-2
```
---
### 6. 멈추기 · 재시작
```bash
# 멈춤
systemctl --user stop log_collector.service
systemctl --user stop log_uploader.timer

# 재시작
systemctl --user restart log_collector.service
systemctl --user restart log_uploader.timer
```
---
### 7. 네트워크 · 권한 점검 팁
- 네트워크 : `curl -I https://s3.ap-northeast-2.amazonaws.com` 가 응답해야 함
- IAM 최소 권한 : `s3:ListBucket` on `arn:aws:s3:::sample-psj-s3`, `s3:PutObject` on `arn:aws:s3:::sample-psj-s3/*`
- systemd 환경 차이 방지 : `/usr/bin/aws`, `/usr/bin/curl` 절대경로와 `-region ap-northeast-2` 사용