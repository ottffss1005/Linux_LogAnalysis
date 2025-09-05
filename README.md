# Linux 사용자별 로그 수집 및 분석 프로젝트

## 프로젝트 목표
- Linux 서버에서 여러 사용자를 생성하고, 권한을 다르게 설정하여 명령어 실행 기록과 파일접근 (permisson denied) 기록을 관리
- 사용자별 로그(history, Permission denied)를 수집하고 분석
- 권한이 없는 접근 시도 발생 시 Slack 알림 발송
- cron과 awk를 활용하여 자동화

---

## 사용자 및 권한 설정

| 사용자  | 권한                      | 설명 |
|---------|---------------------------|------|
| ubuntu  | 읽기 + 쓰기               | 공유 파일 접근 및 수정 가능 |
| dev     | 읽기만                   | 공유 파일 읽기만 가능 |
| ops     | 접근 불가                | 공유 파일에 접근 불가 |



## 로그 데이터 구조
```
srv/
    shared_dir/           # 세 사용자의 공유 디렉터리(ops는 접근 권한이 없음)
        poem.txt          # 권한 있는 사용자만 접근

log_analysis/
    ubuntu/
        history.log       # ubuntu 사용자가 실행한 명령어 기록
        denied.log        # ubuntu의 Permission denied 로그
    dev/
        history.log       # dev 사용자가 실행한 명령어 기록
        denied.log        # dev의 Permission denied 로그
    ops/
        history.log       # ops 사용자가 실행한 명령어 기록
        denied.log        # ops의 Permission denied 로그
    summary/
        comparison_report.txt  # 사용자별 denied 횟수, top 명령어 집계
```

## 로그 분석 명령어 예시

### 1. 인증 로그 수집
```
sudo grep "Permission denied" /var/log/auth.log | grep "ubuntu" > ~/log_analysis/ubuntu/denied.log
sudo grep "Permission denied" /var/log/auth.log | grep "dev" > ~/log_analysis/dev/denied.log
sudo grep "Permission denied" /var/log/auth.log | grep "ops" > ~/log_analysis/ops/denied.log
```

### Top 명령어 집계 (awk 활용)
```
awk '{cmd[$1]++} END {for(c in cmd) print cmd[c], c}' ~/log_analysis/ubuntu/history.log > ~/log_analysis/summary/ubuntu_summary.txt
awk '{cmd[$1]++} END {for(c in cmd) print cmd[c], c}' ~/log_analysis/dev/history.log > ~/log_analysis/summary/dev_summary.txt
awk '{cmd[$1]++} END {for(c in cmd) print cmd[c], c}' ~/log_analysis/ops/history.log > ~/log_analysis/summary/ops_summary.txt
```

### cron 예시 (1분마다 자동 수집/분석)
```
* * * * * /bin/bash /home/ubuntu/log_collect.sh
* * * * * /bin/bash /home/ubuntu/log_analyze.sh
```

---

### 2. 파일접근 로그 수집

### auditd 설치 및 활성화

```bash
sudo apt update
sudo apt install auditd audispd-plugins -y
sudo systemctl enable --now auditd
```

* `auditd`: 커널에서 발생하는 시스템 콜 이벤트를 기록하는 데몬
* `audispd-plugins`: audit 로그를 외부로 전달하거나 가공할 수 있는 플러그인 모음

### 규칙 설정 (Permission denied 추적)

```bash
sudo auditctl -a always,exit -F arch=b64 -S open,openat,creat \
              -F exit=-EACCES -F auid=1001 -k denied-all
```

* `-a always,exit` : 시스템 콜 종료 시점에 항상 기록
* `-S open,openat,creat` : 파일 열기/생성 관련 syscall 감시
* `-F exit=-EACCES` : 권한 거부(`Permission denied`) 된 경우만
* `-F auid=1001` : `user01` 사용자만 추적
* `-k denied-all` : 로그 검색 키워드


### 로그 파싱

```bash
sudo ausearch -ua ops -k denied-all -i | awk -v RS="----" '
/type=PROCTITLE/ {
    if (match($0, /proctitle=(.*)/, p)) {
        cmd = p[1]
        gsub(/^ +| +$/, "", cmd)
        print strftime("%Y-%m-%d %H:%M:%S"), "| user=ops | cmd=\"" cmd "\" | result=Permission denied"
    }
}'
```

* `ausearch`: audit 로그 검색 툴
* `-ua ops`: ops의 이벤트만
* `-k denied-all`: 지정한 규칙 키만
* `-i`: 사람이 읽기 좋은 포맷
* `awk`: 각 블록에서 `proctitle` → 실행한 명령어 복원

출력 예시:
<img width="1587" height="189" alt="image" src="https://github.com/user-attachments/assets/7455feb0-d298-4b9b-97aa-87adadf6bce9" />

```
2025-09-05 14:09:16 | user=ops | cmd="cat /etc/shadow"{생략} | result=Permission denied
```

---

## 트러블 슈팅





