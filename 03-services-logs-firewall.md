# 03. 서비스 관리, 로그, 방화벽

## 목표
- `systemctl`로 서비스의 상태를 읽고, 켜고 끄고, 부팅 시 자동 시작을 설정한다.
- 로그에서 접속 성공/실패 기록을 찾아 읽는다.
- 방화벽(firewalld)에서 포트를 열고 닫으며, 방화벽과 서비스가 모두 맞아야 접속된다는 것을 확인한다.

## 환경
- VMware Workstation Pro 26H1 / Rocky Linux 9 (Minimal Install)
- `server01`에 관리자 계정(`cheol`)으로 SSH 접속해 작업 (창 2~3개 사용)

## 핵심 개념

### 서비스와 systemd
- **서비스(데몬)**: 서버가 켜져 있는 동안 계속 돌면서 요청을 기다리는 프로그램 (`sshd`, `firewalld`, `crond` 등).
- **systemd**가 서비스를 관리하고, **`systemctl`**이 systemd에게 명령을 내리는 도구이다.
- `enabled`와 `active`는 서로 다른 개념이다.

| | 뜻 | 관련 명령 |
|---|---|---|
| active / inactive | **지금** 돌고 있는가 | `start`, `stop` |
| enabled / disabled | **부팅할 때** 자동으로 켜지는가 | `enable`, `disable` |

### 로그가 쌓이는 곳

| 위치 | 보는 법 | 내용 |
|---|---|---|
| journald | `journalctl` | 서비스별, 시간별로 걸러서 보기 좋음 |
| `/var/log/secure` | `tail`, `grep` | 로그인, sudo 등 인증 기록 |

### 방화벽
- 포트는 서버의 문 번호이고(SSH 22, HTTP 80, HTTPS 443), 방화벽은 포트 기준으로 접속을 허용하거나 막는다.
- 방화벽은 **문만 열어 줄 뿐**이고, 그 포트에서 기다리는 서비스가 실제로 떠 있어야 접속된다.

## 실습 과정

### A. 서비스 상태 읽기
```bash
systemctl status sshd
systemctl is-active sshd
systemctl is-enabled sshd
```
- `Loaded:` 줄의 `enabled`는 부팅 시 자동 시작, `Active:` 줄의 `active (running)`은 지금 실행 중.
- `status`는 사람이 읽기 좋은 상세 출력이고, `is-active`/`is-enabled`는 한 단어만 나와서 스크립트에서 조건 판단에 쓰기 좋다.
  ```bash
  if systemctl is-active --quiet sshd; then echo "정상"; fi
  ```

### B. 서비스 끄기, 켜기, 재시작
```bash
sudo systemctl stop sshd       # 끄기
sudo systemctl start sshd      # 켜기
sudo systemctl restart sshd    # 껐다가 바로 켜기 (설정 변경 후 적용)
```
- `sshd`를 끄면 **이미 접속한 창은 유지**되고, **새 접속만 `Connection refused`로 거부**된다.
- 꺼진 상태에서도 `Loaded:`는 `enabled`로, 부팅하면 다시 켜지도록 설정은 그대로 남는다.
- 재시작하면 새 프로세스가 뜨므로 PID가 바뀐다.
  ```bash
  systemctl status sshd | grep "Main PID"    # 재시작 전후로 번호 비교
  ```

### C. 부팅 시 자동 시작: enable / disable
연습 대상은 `crond`(접속에 영향이 없는 서비스)를 사용했다.
```bash
sudo systemctl disable crond   # 자동 시작 끄기 (지금 상태는 그대로 active)
sudo reboot
systemctl is-active crond      # 재부팅 후: inactive
systemctl is-enabled crond     # disabled
sudo systemctl enable --now crond   # 자동 시작 켜기 + 지금 바로 시작
```

| | 지금 상태 | 부팅 후 상태 |
|---|---|---|
| `start` / `stop` | 바꿈 | 영향 없음 |
| `enable` / `disable` | 영향 없음 | 바꿈 |

- 서비스를 새로 설치하면 보통 `enable --now`를 쓴다. `enable`을 빼먹으면 재부팅 후에 서비스가 안 올라오는 문제가 생긴다.

### D. 로그 읽기
```bash
sudo journalctl -u sshd -n 20                   # sshd 로그 마지막 20줄
sudo journalctl -u sshd --since "1 hour ago"    # 최근 1시간
sudo tail -f /var/log/secure                    # 실시간으로 따라가며 보기 (Ctrl+C로 종료)
sudo grep "Failed password" /var/log/secure     # 로그인 실패 기록만
sudo grep "Accepted" /var/log/secure            # 로그인 성공 기록만
sudo journalctl -u sshd | grep Failed           # 명령어 출력에서 걸러내기
```
- 다른 창에서 일부러 틀린 비밀번호로 SSH를 시도하면, `tail -f` 창에 `Failed password for ... from (IP) port ...` 줄이 바로 올라온다.
- 로그 한 줄은 **시간 → 호스트 이름 → 프로그램[PID] → 내용** 순서이다.
  - `Accepted password for cheol from ...`: 접속 성공
  - `Failed password for cheol from ...`: 접속 실패, 어느 IP에서 시도했는지까지 남는다.
- 파일을 검색할 때는 `grep 글자 파일`, 명령어의 출력을 거를 때는 `명령어 | grep 글자`.

### E. 방화벽
```bash
sudo firewall-cmd --list-all
```
- `services:`에 `ssh`가 있어서 SSH 접속이 되고, `ports:`는 번호로 직접 연 포트이다.

**닫혀 있을 때와 열었을 때 비교** (임시 웹서버 사용)
```bash
mkdir ~/web && cd ~/web
echo "hello firewall" > index.html
python3 -m http.server 8080           # 임시 웹서버, Ctrl+C로 종료
```
1. 방화벽에서 8080을 열기 **전**: Windows에서 `curl.exe http://서버IP:8080` 접속이 막힌다.
2. 포트를 열기:
   ```bash
   sudo firewall-cmd --add-port=8080/tcp --permanent
   sudo firewall-cmd --reload
   ```
3. 연 **후**: 같은 접속이 성공하고 `hello firewall`이 출력된다.

**정리**
```bash
sudo firewall-cmd --remove-port=8080/tcp --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-all          # ports: 줄이 비어 있는지 확인
rm -r ~/web
```

- `--permanent`는 설정 파일에 저장만 하고 바로 적용하지 않는다. `--reload`를 해야 적용된다.
- `--permanent` 없이 추가하면 바로 적용되지만 reload나 재부팅하면 사라진다.
- 서비스 이름으로 열 수도 있다. (`--add-service=http`)

## 명령어 정리

### 따옴표를 쓰는 때
- 셸은 **공백으로 단어를 나눠서** 인자로 넘긴다. 그래서 값 안에 공백이 있으면 따옴표로 묶어야 하나의 인자가 된다.
  ```bash
  grep Failed password /var/log/secure     # password를 파일 이름으로 착각해서 에러
  grep "Failed password" /var/log/secure   # 정상
  journalctl --since "1 hour ago"
  ```
- 단어 하나(공백 없음)는 따옴표가 없어도 되고, 헷갈리면 붙여도 해가 없다.
- 따옴표 자체는 셸이 해석하고 버려서 프로그램에는 넘어가지 않는다.
- 큰따옴표는 `$변수`를 해석하고, 작은따옴표는 글자 그대로 둔다.

### `echo 내용 > 파일`
- 글자를 화면 대신 **파일에 저장**한다. 이미 있는 파일은 덮어쓰고, `>>`는 끝에 덧붙인다. (`index.html`을 만들 때 사용)

## 막혔던 점과 해결

### ① 따옴표를 붙일 때와 안 붙일 때의 구분
- 처음에는 규칙이 없어 보였지만, 기준은 **값 안에 공백이 있는지**였다. 셸이 공백으로 인자를 쪼개기 때문이다.
- 안 붙였을 때 이상한 에러(파일 없음 등)가 나면 "공백 때문에 쪼개졌나?"를 먼저 의심한다.

### ② 명령어 오타와 구성
- `restart`, `sud restart`처럼 쳐서 실행되지 않았다. 서비스 재시작은 **`sudo systemctl restart 서비스이름`** 전체가 하나의 명령이다.

### ③ 서비스 조작에 `sudo`가 필요
- `systemctl start sshd`를 `sudo` 없이 시도한 뒤 `sudo systemctl start sshd`로 다시 실행했다.
- 상태를 보는 명령(`status`, `is-active`)은 `sudo`가 필요 없지만, 서비스를 바꾸는 명령(`start`, `stop`, `restart`, `enable`, `disable`)은 필요하다.

### ④ 일부 명령이 history에 안 보임
- 방화벽 포트를 여는 명령은 별도로 연 창에서 쳤기 때문에 한 창의 `history`에는 나타나지 않았다. history는 셸(창)마다 따로 관리되고, 창을 닫을 때 저장된다.
- 실습 기록을 남길 때는 **한 창에서** 진행하거나, 다른 창을 `exit`로 닫은 뒤 `history`를 확인한다.

## 배운 점
- 서비스는 **지금 상태(active)**와 **부팅 시 설정(enabled)**을 따로 본다.
- 서비스가 꺼지면 `Connection refused`, 방화벽이 막으면 접속이 안 된다. 접속이 안 될 때는 **서비스 → 포트 대기 → 방화벽** 순서로 확인한다.
- 문제가 생기면 로그부터 본다. 로그인 실패 기록에는 시간, 계정, 접속 IP까지 남는다.
- 방화벽은 `--permanent`로 저장하고 `--reload`로 적용하는 두 줄을 짝으로 쓰고, 연습 후에는 열어 둔 포트를 닫는다.
