# 📗 08.22 AWS
## 📚 3 Tier Architecture Project
### 계획 수립
- 메인 : 구인정, 박세진, 홍승재
- 서브 : 김도영, 이현정
---
1. 3 Tier로 구성 (public, private, DB 영역)
2. ELB로 분산하여 Webserver > DB 연결 (포트 : 8080 사용)
3. DB는 Master와 Slave로 나눈 후 Slave는 Standby, DB 자동 백업 설정
4. 로그를 수집할 쉘 스크립트 사용 > 로그는 S3로 저장 (1시간 간격으로 저장)
5. CPU 사용률 50% 이상 시 경고, 70% 이상 시 치명적 결함이라는 Slack으로 알림과 메일 보내기
6. 50% 이상 CPU 사용률 일시 Webserver 오토 스케일링 (초기 2개, 최대 6개)
7. 구성 완료 시 그라파나를 적용하여 실시간 모니터링 구성
### 진행 상황
- VPC (10.0.0.0/16) 로 구성 완료
- AZ 2개로 구성 완료
- Public subnet1 (10.0.1.0/24), Public subnet2 (10.0.2.0/24) 로 구성 완료
- Private subnet1 (10.0.21.0/24), Private subnet2 (10.0.22.0/24) 로 구성 완료
- NAT, IGW 구성 완료
- Bastion host 구성 완료
- 인스턴스 (Webserver01, Webserver02) 구성 완료
- RDS, S3 구성 완료
### 계획 완료 후 추가 진행 계획
1. CI/CD 분리 및 파이프라인 구축
2. Elastic Searth 클러스터 구성 및 애플리케이션 마운트 (Kafka 이용)
### 로그 파일 S3에 자동 저장 쉘 스크립트
```bash
#!/bin/bash

LOG_DIR="/var/log/mylogs"    # 로그 저장할 폴더 경로 지정
mkdir -p "$LOG_DIR"    # 폴더 없으면 새로 생성
sudo chown ec2-user:ec2-user /var/log/mylogs    # mylogs에 권한 부여

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
    sleep 5                       
  done

  if [ $n -ge 3 ]; then                                               # 재시도 횟수가 3 이상이면 if문 실행
    echo "$filename 파일 s3 업로드에 실패했습니다."                     # 콘솔에 업로드 실패 메시지 출력
    err="${last_err:-에러 메시지 없음}"                                # last_err 가 비었거나 미정의면 기본 문구로 대체해 err 에 저장
    short_err="$(printf '%s' "$err" | head -c 500)"                   # 에러 내용을 최대 500바이트로 잘라 short_err 에 저장
    notify_slack "❌ 업로드 실패 : $filename (3회 재시도 후 실패)\n에러 : $short_err"  # 슬랙으로 실패 알림 메시지 전송
  fi                                                              
done
```