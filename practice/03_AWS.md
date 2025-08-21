# 📗 08.19 AWS
## 📚 로그 파일 S3에 자동 적재
### 1. UTC -> KST 타임존 변경
```bash
sudo timedatectl set-timezone Asia/Seoul
timedatectl
```
### 2. 로그 수집 스크립트 : /home/ec2-user/log_collector.sh
```bash
#!/bin/bash

LOG_DIR="/var/log/mylogs"    # 로그 저장할 폴더 경로 지정
mkdir -p "$LOG_DIR"    # 폴더 없으면 새로 생성

while true    # 무한 반복 시작
do
  # 날짜+시간(년월일-시분)으로 로그 파일 이름 생성
	LOG_FILE="$LOG_DIR/$(date +%Y%m%d-%H%M).log"
	# 현재 시각 시:분:초 저장
	NOW=$(date +"%H:%M:%S")    
	# CPU 사용률 숫자만 뽑아서 저장
	CPU=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d. -f1)
	# 메모리 사용률(%) 계산해서 저장
	MEM=$(free | awk '/Mem/{printf "%.0f", $3/$2*100}')
	# 디스크 사용률(%) 가져와서 % 빼고 숫자만 저장
	DISK=$(df / | tail -1 | awk '{print $5}' | sed 's/%//')
	# 현재 시각 + CPU/메모리/디스크 사용률 로그에 기록
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
### 3. 로그 업로드 스크립트 + Slack 알림 : `/home/ec2-user/log_uploader.sh`
```bash
#!/bin/bash

# 에러 발생 시 즉시 종료, 파이프라인 에러 잡기, 정의 안 된 변수 사용 시 에러 처리
set -Eeuo pipefail               

# 설정
LOG_DIR="/var/log/mylogs"        # 로그 파일이 저장될 디렉터리 경로
BUCKET="s3://sample-psj-s3/logs" # 업로드할 대상 S3 버킷 경로
AWS_REGION="ap-northeast-2"      # 사용할 AWS 리전 설정
AWS_CLI="/usr/bin/aws"           # AWS CLI 실행 파일 위치 지정
CURL="/usr/bin/curl"             # curl 실행 파일 위치 지정
SLACK_WEBHOOK="Slack URL"        # Slack Webhook URL

# 슬랙 알림을 보내는 함수 정의
notify_slack() {
  local msg="$1"                 	# 첫 번째 인자를 msg 변수에 저장
  $CURL -s -X POST -H 'Content-type: application/json'   # Slack API로 POST 요청 보냄
    --data "{\"text\":\"${msg}\"}" \                     # 메시지를 JSON 형태로 전달
    "$SLACK_WEBHOOK" >/dev/null || true                  # 전송 실패해도 스크립트 중단되지 않도록 처리
}

# 현재 시간에서 "년월일-시분"을 저장 (작성 중인 로그 파일 이름과 비교하기 위해)
CUR_MINUTE="$(TZ='Asia/Seoul' date +%Y%m%d-%H%M)"

# 로그 디렉터리 내의 .log 파일 목록 수집
shopt -s nullglob               # 매칭되는 파일 없을 때 빈 배열 반환 (오류 방지)
files=("$LOG_DIR"/*.log)        # .log 확장자를 가진 모든 파일을 배열에 저장

# 업로드할 파일이 하나도 없는 경우
if [ ${#files[@]} -eq 0 ]; then
  notify_slack "ℹ️ 업로드할 로그 파일이 없습니다. (경로: $LOG_DIR)" # 슬랙에 안내 메시지 전송
  exit 0                       # 스크립트 정상 종료
fi

# 수집된 파일들을 하나씩 처리
for file in "${files[@]}"; do
  [ -f "$file" ] || continue     # 파일이 아니면 건너뜀
  filename="$(basename "$file")" # 파일명만 추출

  # 현재 시각 분 단위 파일은 업로드 대상에서 제외 (작성 중일 수 있음)
  [[ "$filename" == "$CUR_MINUTE.log" ]] && continue

  # 업로드를 최대 3회 재시도
  n=0                            # 시도 횟수 초기화
  last_err=""                    # 마지막 에러 메시지 저장 변수 초기화
  until [ $n -ge 3 ]; do          # 3회 시도 반복
    if $AWS_CLI s3 cp "$file" "$BUCKET/$filename" --region "$AWS_REGION" --only-show-errors 2> >(last_err=$(cat); typeset -p last_err >/dev/null); then
      rm -f "$file" || true       # 업로드 성공 시 해당 파일 삭제
      notify_slack "✅ 업로드 성공 : $filename → sample-psj-s3/logs" # 성공 알림 전송
      break                       # 성공했으니 반복 종료
    fi
    n=$((n+1))                    # 실패하면 횟수 증가
    sleep 5                       # 5초 대기 후 재시도
  done

  # 3회 모두 실패한 경우
  if [ $n -ge 3 ]; then
    echo "$filename 파일 s3 업로드에 실패했습니다." # 콘솔에 실패 메시지 출력
    # 에러 메시지가 있다면 슬랙에도 전달
    if [ -n "${last_err:-}" ]; then
      short_err="$(echo "$last_err" | head -c 500)"   # 너무 길면 앞부분 500자만 추출
      notify_slack "❌ 업로드 실패 : $filename (3회 재시도 후 실패)\n에러: $short_err"
    else
      notify_slack "❌ 업로드 실패 : $filename (3회 재시도 후 실패)"
    fi
  fi
done
```
- 권한 부여
```bash
chmod +x /home/ec2-user/log_uploader.sh
```
### 4. user systemd 유닛 파일 경로 : `~/.config/systemd/user/`
- 경로 생성
```bash
mkdir -p /home/ec2-user/.config/systemd/user
```
### 4-1. 수집기 서비스 : `~/.config/systemd/user/log_collector.service`
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
### 4-2. 업로더 서비스 : `~/.config/systemd/user/log_uploader.service`
```bash
[Unit]
Description=Log Uploader Service

[Service]
Type=oneshot
ExecStart=/home/ec2-user/log_uploader.sh
```
### 4-3. 업로더 타이머 : `~/.config/systemd/user/log_uploader.timer`
```bash
[Unit]
Description=Run log uploader every 1 minute

[Timer]
OnCalendar=*:0/1  # 5분마다 업로드 설정 시 *:0/5
Persistent=true

[Install]
WantedBy=timers.target
```
### 5. systemd 활성화 : user 모드
```bash
systemctl --user daemon-reload
systemctl --user enable --now log_collector.service
systemctl --user enable --now log_uploader.timer
```
- 부팅 후 자동 실행 유지
```bash
sudo loginctl enable-linger ec2-user
```
### 6. 확인 방법
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
### 7. 멈추기 · 재시작
```bash
# 멈춤
systemctl --user stop log_collector.service
systemctl --user stop log_uploader.timer

# 재시작
systemctl --user restart log_collector.service
systemctl --user restart log_uploader.timer
```

### 8. 네트워크 · 권한 점검 팁
- 네트워크 : `curl -I https://s3.ap-northeast-2.amazonaws.com` 가 응답해야 함
- IAM 최소 권한 : `s3:ListBucket` on `arn:aws:s3:::sample-psj-s3`, `s3:PutObject` on `arn:aws:s3:::sample-psj-s3/*`
- systemd 환경 차이 방지 : `/usr/bin/aws`, `/usr/bin/curl` 절대경로와 `-region ap-northeast-2` 사용

## 💎 개선 사항 정리
### UTC로 로그 파일이 생성되어 KST로 타임존 수정
```bash
sudo timedatectl set-timezone Asia/Seoul
timedatectl
```
### log_uploader.sh / 추가 : 로그 파일이 없으면 로그 없음 출력
```bash
# 대상이 없으면 안내만 하고 종료
if [ ${#files[@]} -eq 0 ]; then
  notify_slack "ℹ️ 업로드할 로그 파일이 없습니다. (경로 : $LOG_DIR)"
  exit 0
fi
```
### log_uploader.sh / 기존 : 직전 1분 .log 파일 업로드 코드
```bash
PREV_MINUTE="$(date -d '1 minute ago' +%Y%m%d-%H%M)"
file="$LOG_DIR/$PREV_MINUTE.log"
[ -f "$file" ] || exit 0
filename="$(basename "$file")"
...
aws s3 cp "$file" "$BUCKET/$filename"
...
```
### log_uploader.sh / 개선 : 누락된 모든 .log 파일 업로드 코드
```bash
# 현재 분은 아직 쓰고 있으니 제외
CUR_MINUTE="$(date +%Y%m%d-%H%M)"

shopt -s nullglob
for file in "$LOG_DIR"/*.log; do
  filename="$(basename "$file")"

  # 현재 분 파일은 건너뜀
  [[ "$filename" == "$CUR_MINUTE.log" ]] && continue

  # 업로드 시도
  if $AWS_CLI s3 cp "$file" "$BUCKET/$filename" --region "$AWS_REGION"; then
    rm -f "$file"
    notify_slack "✅ 업로드 성공 : $filename → sample-psj-s3/logs"
  ...
```
## 📊 결과 확인
- 박세진 노션 : [PSJ REPOSITORY](https://psjrepository.notion.site/DAY-25-2543d86ddbdc80568b83fef67c889168)