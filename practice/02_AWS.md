# 📗 08.14 AWS
### 자동으로 인스턴스 t3.small로 자동화
```bash
./change-to-t3-small.sh --instance-id i-09908ba6f81cc8a79 --dry-run
./change-to-t3-small.sh --instance-id i-09908ba6f81cc8a79 --force

#!/usr/bin/env bash
set -Eeuo pipefail

PROFILE=""
REGION=""
INSTANCE_ID=""
NAME_TAG=""
DRYRUN=0
FORCE=0   # 현재 타입이 t3.micro가 아니어도 강제 변경

usage() {
  cat <<'USAGE'
Usage:
  change-to-t3-small.sh (--instance-id i-xxxx | --name <TAG_NAME>) [--profile <AWS_PROFILE>] [--region <REGION>] [--dry-run] [--force]

Options:
  --instance-id ID       대상 인스턴스 ID
  --name NAME            Name 태그로 대상 선택(여러 개면 모두 변경)
  --profile PROFILE      AWS CLI profile
  --region REGION        AWS region (예: ap-northeast-2)
  --dry-run              실제 변경 없이 검증만 수행
  --force                현재 타입이 t3.micro가 아니어도 t3.small로 변경

Example:
  ./change-to-t3-small.sh --name web01 --profile default --region ap-northeast-2
  ./change-to-t3-small.sh --instance-id i-0123456789abcdef0 --dry-run
USAGE
}

AWSOPTS=()
awsc() {
  aws "${AWSOPTS[@]}" "$@"
}

parse_args() {
  while [[ $# -gt 0 ]]; do
    case "$1" in
      --instance-id) INSTANCE_ID="$2"; shift 2;;
      --name) NAME_TAG="$2"; shift 2;;
      --profile) PROFILE="$2"; shift 2;;
      --region) REGION="$2"; shift 2;;
      --dry-run) DRYRUN=1; shift;;
      --force) FORCE=1; shift;;
      -h|--help) usage; exit 0;;
      *) echo "Unknown option: $1"; usage; exit 1;;
    esac
  done

  [[ -n "${PROFILE}" ]] && AWSOPTS+=(--profile "${PROFILE}")
  [[ -n "${REGION}"  ]] && AWSOPTS+=(--region  "${REGION}")

  if [[ -z "${INSTANCE_ID}" && -z "${NAME_TAG}" ]]; then
    echo "에러: --instance-id 또는 --name 중 하나를 지정하세요." >&2
    exit 1
  fi
}

ensure_aws() {
  command -v aws >/dev/null || { echo "aws CLI가 필요합니다."; exit 1; }
}

ids_from_name() {
  local name="$1"
  awsc ec2 describe-instances \
    --filters "Name=tag:Name,Values=${name}" "Name=instance-state-name,Values=pending,running,stopping,stopped" \
    --query 'Reservations[].Instances[].InstanceId' --output text
}

get_state() {
  local id="$1"
  awsc ec2 describe-instances --instance-ids "$id" \
    --query 'Reservations[0].Instances[0].State.Name' --output text
}

get_type() {
  local id="$1"
  awsc ec2 describe-instances --instance-ids "$id" \
    --query 'Reservations[0].Instances[0].InstanceType' --output text
}

wait_state() {
  local id="$1" want="$2"
  case "$want" in
    stopped) awsc ec2 wait instance-stopped --instance-ids "$id" ;;
    running) awsc ec2 wait instance-running --instance-ids "$id" ;;
    *) echo "알 수 없는 상태: $want"; exit 1;;
  esac
}

stop_if_needed() {
  local id="$1" st
  st=$(get_state "$id")
  case "$st" in
    running)
      echo "[i] $id 현재 상태: running → stop 실행"
      if [[ $DRYRUN -eq 1 ]]; then
        awsc ec2 stop-instances --instance-ids "$id" --dry-run || true
      else
        awsc ec2 stop-instances --instance-ids "$id" >/dev/null
        wait_state "$id" stopped
      fi
      ;;
    stopping)
      echo "[i] $id stopping 대기..."
      [[ $DRYRUN -eq 1 ]] || wait_state "$id" stopped
      ;;
    pending)
      echo "[i] $id pending 대기 후 stop"
      [[ $DRYRUN -eq 1 ]] || awsc ec2 wait instance-running --instance-ids "$id"
      [[ $DRYRUN -eq 1 ]] || awsc ec2 stop-instances --instance-ids "$id" >/dev/null
      [[ $DRYRUN -eq 1 ]] || wait_state "$id" stopped
      ;;
    stopped)
      echo "[i] $id 이미 stopped"
      ;;
    shutting-down|terminated)
      echo "[!] $id 상태가 $st 입니다. 건너뜀." ; return 1;;
    *)
      echo "[!] $id 알 수 없는 상태: $st" ; return 1;;
  esac
}

modify_type() {
  local id="$1" current target="t3.small"
  current=$(get_type "$id")
  echo "[i] $id 현재 타입: $current"

  if [[ "$current" == "$target" ]]; then
    echo "[=] $id 이미 $target 입니다. 건너뜀."
    return 0
  fi

  if [[ $FORCE -eq 0 && "$current" != "t3.micro" ]]; then
    echo "[!] $id 현재 타입이 t3.micro가 아닙니다($current). --force 없으면 변경하지 않습니다."
    return 1
  fi

  echo "[+] $id 타입을 $current → $target 로 변경"
  if [[ $DRYRUN -eq 1 ]]; then
    awsc ec2 modify-instance-attribute --instance-id "$id" \
      --instance-type "{\"Value\":\"$target\"}" --dry-run || true
  else
    awsc ec2 modify-instance-attribute --instance-id "$id" \
      --instance-type "{\"Value\":\"$target\"}"
  fi
}

start_if_stopped() {
  local id="$1" st
  st=$(get_state "$id")
  if [[ "$st" == "stopped" ]]; then
    echo "[i] $id start 실행"
    if [[ $DRYRUN -eq 1 ]]; then
      awsc ec2 start-instances --instance-ids "$id" --dry-run || true
    else
      awsc ec2 start-instances --instance-ids "$id" >/dev/null
      wait_state "$id" running
    fi
  fi
}

process_one() {
  local id="$1"
  echo "===================================="
  echo "[*] 대상: $id"
  stop_if_needed "$id" || return 1
  modify_type "$id" || return 1
  start_if_stopped "$id"
  local after
  after=$(get_type "$id")
  echo "[OK] $id 최종 타입: $after"
}

main() {
  ensure_aws
  parse_args "$@"

  local ids=()
  if [[ -n "$INSTANCE_ID" ]]; then
    ids+=("$INSTANCE_ID")
  else
    mapfile -t ids < <(ids_from_name "$NAME_TAG")
    [[ ${#ids[@]} -gt 0 ]] || { echo "Name='${NAME_TAG}' 인스턴스를 찾지 못함"; exit 1; }
  fi

  for id in "${ids[@]}"; do
    process_one "$id" || echo "[WARN] $id 처리 중 일부 단계 실패"
  done
}

main "$@"
```