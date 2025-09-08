# 운영 시스템(Linux) 사용자별 로그 수집 및 분석 프로젝트

## 👥 Team Members
<table>
  <tr>
    <th>최소영</th>
    <th>홍혜원</th>
  </tr>
  <tr>
    <td align="center">
      <img src="https://github.com/ottffss1005.png" width="120" /><br/>
      <a href="https://github.com/ottffss1005">@ottffss1005</a>
    </td>
    <td align="center">
      <img src="https://github.com/hyewon8245.png" width="120" /><br/>
      <a href="https://github.com/hyewon8245">@hyewon8245</a>
    </td>
  </tr>
</table>

---

## ✅ **프로젝트 개요**
- Linux 서버에서 여러 사용자를 생성하고, 권한을 다르게 설정하여 명령어 실행 기록과 파일접근 (permisson denied) 기록을 관리
- 사용자별 로그(history, Permission denied)를 수집하고 분석

### **주요 목표**
✔️ 사용자별 명령어 실행 기록(`history`)과 Permission denied 이벤트 수집  
✔️ 로그 자동 분석  
✔️ Slack 알림을 통해 보안 이벤트 실시간 통보  
✔️ cron을 활용한 자동화 및 스케줄링  
✔️ auditd로 권한 거부 이벤트 추적 (커널 레벨 보안 로그)  

---

## 📂 **프로젝트 디렉토리 구조**
```
srv/
    shared_dir/                # 공유 디렉터리 (ops는 접근 불가)
        poem.txt               # 테스트용 파일

log_analysis/
    ubuntu/
        history.log            # ubuntu 사용자의 명령어 로그
        denied.log             # ubuntu의 Permission denied 로그
    dev/
        history.log
        denied.log
    ops/
        history.log
        denied.log
    summary/
        comparison_report.txt  # 사용자별 denied 횟수, top 명령어 집계
```

---

## 👤 **사용자 및 권한 정책**
| 사용자  | 권한            | 설명                                |
|---------|-----------------|-----------------------------------|
| **ubuntu** | 읽기 쓰기       | 공유 파일 접근 및 수정 가능           |
| **dev**    | 읽기           | 공유 파일 읽기만 가능                |
| **ops,user01**    | 접근 불가        | 공유 파일 접근 시 Permission denied 발생 |

---

## 🔍 **로그 수집 및 분석**

### **1. Permission Denied 로그 수집**
`/var/log/auth.log` 기반으로 사용자별 denied 로그를 추출:
```bash
sudo grep "Permission denied" /var/log/auth.log | grep "ubuntu" > ~/log_analysis/ubuntu/denied.log
sudo grep "Permission denied" /var/log/auth.log | grep "dev" > ~/log_analysis/dev/denied.log
sudo grep "Permission denied" /var/log/auth.log | grep "ops" > ~/log_analysis/ops/denied.log
```

---

### **2. Top 명령어 집계 (awk 활용)**
```bash
awk '{cmd[$1]++} END {for(c in cmd) print cmd[c], c}' ~/log_analysis/ubuntu/history.log > ~/log_analysis/summary/ubuntu_summary.txt
awk '{cmd[$1]++} END {for(c in cmd) print cmd[c], c}' ~/log_analysis/dev/history.log > ~/log_analysis/summary/dev_summary.txt
awk '{cmd[$1]++} END {for(c in cmd) print cmd[c], c}' ~/log_analysis/ops/history.log > ~/log_analysis/summary/ops_summary.txt
```

---

### **3. cron을 활용한 자동화**
1분마다 로그 수집 및 분석 실행:
```bash
* * * * * /bin/bash /home/ubuntu/log_collect.sh
* * * * * /bin/bash /home/ubuntu/log_analyze.sh
```


## 📑 분석 보고서 예시
- ~/log_analysis/summary/comparison_report.txt

```
===== 2025-09-05 14:20:00 Summary =====
User: ubuntu
  Permission denied attempts: 0
  Top commands:
    12 ls
    7 cat
    3 echo

User: dev
  Permission denied attempts: 1
  Top commands:
    5 cat
    2 less
    1 ls

```
---

## 🛡️ **고급 권한 거부 추적 (auditd 활용)**
- 특정 사용자가 5번이상 권한 밖의 행위를 한 경우에 모니터링 알람 구축

### **auditd 설치 및 활성화**
```bash
sudo apt update
sudo apt install auditd audispd-plugins -y
sudo systemctl enable --now auditd
```

#### **규칙 추가 (Permission denied 이벤트 추적)**

```bash
sudo auditctl -a always,exit -F arch=b64 -S open,openat,creat -F exit=-EACCES -F auid=1001 -k denied-all
```
- `-a always,exit`: 시스템 콜 종료 시점에 기록  
- `-F exit=-EACCES`: 권한 거부(`Permission denied`)만 기록  
- `-k denied-all`: 검색 키워드 지정  

---

### **로그 파싱 (ausearch + awk)**
```bash
sudo ausearch -ua user01 -k denied-all -i | awk -v RS="----" '
/type=PROCTITLE/ {
    if (match($0, /proctitle=(.*)/, p)) {
        cmd = p[1]
        gsub(/^ +| +$/, "", cmd)
        print strftime("%Y-%m-%d %H:%M:%S"), "| user=ops | cmd=\"" cmd "\" | result=Permission denied"
    }
}'
```

✅ 출력 예시:

<img width="1587" height="189" alt="image" src="https://github.com/user-attachments/assets/88e27f0f-a5fb-4a01-a220-36ea92dc6f6d" />


```
2025-09-05 14:09:16 | user=user01 | cmd="cat /etc/shadow" | result=Permission denied
```

---

## 📢 **Slack 알림 연동**
### 9/5 기록 ###
- `auditd` 로그 결과에서 **Permission denied 이벤트** 확인 시 Slack Webhook 호출
`/usr/local/bin/denied_to_slack.sh` 실행

```bash
#!/bin/bash

WEBHOOK_URL="https://hooks.slack.com/services/XXXXX/XXXXX/XXXXXXXX"

LOGS=$(sudo /usr/sbin/ausearch -ua user01 -k denied-all -i | \
/usr/bin/awk -v RS="----" '
/type=PROCTITLE/ {
    if (match($0, /proctitle=(.*)/, p)) {
        cmd = p[1]
        gsub(/^ +| +$/, "", cmd)
        print strftime("%Y-%m-%d %H:%M:%S"), "| user=user01 | cmd=\"" cmd "\" | result=Permission denied"
    }
}')

if [ -n "$LOGS" ]; then
  PAYLOAD=$(echo "$LOGS" | /usr/bin/jq -Rs .)
  /usr/bin/curl -s -X POST -H 'Content-type: application/json' \
       --data "{\"text\": $PAYLOAD}" \
       "$WEBHOOK_URL"
fi
```
<img width="1292" height="280" alt="image" src="https://github.com/user-attachments/assets/41fa66b9-60c1-48b8-9bfc-f4cee309cf46" />

---

### 9/8 기록 ###
실제 요구사항은 **실시간 알림**이었으므로 `tail` 기반 방식으로 전환
### 🔹 ausearch 방식 (초기 시도)

```bash
ausearch -ua user01 -k denied-all -i

```

- audit 로그를 **질의**하여 이벤트 수집
- cron을 활용해서 정기적인 보고서를 작성하고자 하였으나, cron으로 .sh 스크립트를 실행한 경우에 로그 파일을 읽는데에 어려움을 겪는 문제가 발생하여 자동화가 어려움.
- `awk`, `jq` 등을 활용해 Slack Webhook으로 전송 가능
- 장점: 구조화된 출력, 보고서/배치 처리에 적합
- 단점: **실시간 알림 불가** (스크립트를 주기적으로 실행해야 함)
<img width="1292" height="280" alt="image" src="https://github.com/user-attachments/assets/41fa66b9-60c1-48b8-9bfc-f4cee309cf46" />


### 🔹 tail 방식 (최종 적용)

```bash
tail -Fn0 /var/log/audit/audit.log | while read line; do ... done

```

- `/var/log/audit/audit.log` 파일을 **실시간 스트리밍**으로 감시
- 발생 즉시 Slack Webhook으로 전송 가능
- 장점: **실시간 알림 가능**
- 단점: JSON 파싱이나 데이터 구조화가 어려움 (문자열 처리 위주)
 <img width="533" height="101" alt="image" src="https://github.com/user-attachments/assets/39f1c91b-7c0f-4aa5-913c-d1b665fe8bb8" />



## 최종 구현 (tail 방식)

### 규칙 설정

```bash
sudo auditctl -D
sudo auditctl -a always,exit -F arch=b64 -S open,openat,creat \
     -F exit=-EACCES -F auid=1001 -k user01-denied

```

### Slack Webhook 스크립트

`/home/ubuntu/audit/auditd-slack.sh`

```bash
#!/bin/bash
SLACK_WEBHOOK="https://hooks.slack.com/services/AAA/BBB/CCC"
COUNT_FILE="/tmp/user01_denied_count"
[ -f $COUNT_FILE ] || echo 0 > $COUNT_FILE

tail -Fn0 /var/log/audit/audit.log | while read line; do
    # Permission denied (exit=-13) + user01 (auid=1001) 체크
    if echo "$line" | grep -q 'exit=-13' && echo "$line" | grep -q 'auid=1001'; then
        COUNT=$(cat $COUNT_FILE)
        COUNT=$((COUNT+1))
        echo $COUNT > $COUNT_FILE
        echo "DEBUG: count=$COUNT"

        if [ $COUNT -ge 5 ]; then
            echo "DEBUG: Sending Slack alert..."
            PAYLOAD='{"text":":rotating_light: user01 had 5 permission denied events!"}'
            curl -s -X POST -H 'Content-type: application/json' \
                 --data "$PAYLOAD" $SLACK_WEBHOOK
            echo 0 > $COUNT_FILE
        fi
    fi
done

```
### tail 결과
<img width="533" height="101" alt="image" src="https://github.com/user-attachments/assets/d27f7a4e-b2e4-4c7c-b615-e13b6774ae3f" />


### systemd 서비스 등록

```
[Unit]
Description=Auditd Slack Forwarder for user01

[Service]
ExecStart=/home/ubuntu/audit/auditd-slack.sh
Restart=always

[Install]
WantedBy=multi-user.target

```

적용:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now auditd-slack.service
journalctl -u auditd-slack.service -f

```






## 🛠️ **트러블슈팅**
### ❗ 문제: 다른 사용자 로그가 보이지 않음
- **원인**: `auth.log`에는 일부 인증 관련 로그만 남고, 파일 접근 실패는 기록되지 않음
- **해결**: `auditd`를 설치하여 커널 레벨에서 권한 거부 이벤트 추적

설정 예시:
```bash
sudo apt install auditd -y
sudo auditctl -w /shared_dir/poem.txt -p rwxa -k denied_test
ausearch -k denied_test
```
---
### ❗ 문제: Slack에 알람이 오지 않는 문제
- **원인**: cron은 이미 저장되어있는 로그에서 확인하여 알람을 주어야하는 역할을 맞게 되었는데, 로그를 읽지 못하는 문제가 발생함
- **해결**: 모니터링 방법을 바꿔서 정기적인 보고서가 아닌 실시간 알림으로 이벤트 추적

설정 예시:
```bash
tail -Fn0 /var/log/audit/audit.log | while read line; do
```




