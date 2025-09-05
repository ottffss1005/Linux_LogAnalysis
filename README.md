# Linux 사용자별 로그 수집 및 분석 프로젝트

## 프로젝트 목표
- Linux 서버에서 여러 사용자를 생성하고, 권한을 다르게 설정하여 파일 접근과 명령어 실행 기록을 관리
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

### Permission denied 로그 수집
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

## 요약
- log_analysis/ 디렉토리 아래 사용자별로 history와 denied 로그를 저장
- awk로 로그를 분석하여 명령어 빈도와 denied 횟수를 집계
- cron으로 주기적인 자동 수집 및 분석 수행

