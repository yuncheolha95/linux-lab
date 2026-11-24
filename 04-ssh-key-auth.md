# 04. SSH 키 인증과 비밀번호 로그인 차단

## 한 줄 요약
비밀번호 대신 **열쇠 쌍(개인키/공개키)**으로 SSH에 접속하게 만들고, 서버에서 **비밀번호 로그인을 꺼서** 키를 가진 사람만 들어오게 했다.

## 쉬운 비유
- **개인키** = 내 손에 든 열쇠 (접속하는 쪽에만 둔다. 절대 남에게 주지 않는다.)
- **공개키** = 서버 문에 달아 두는 자물쇠 (서버에 등록한다. 남에게 보여줘도 안전하다.)
- 접속할 때 서버가 "이 자물쇠에 맞는 열쇠를 가졌나?"를 확인하고, 맞으면 비밀번호 없이 들여보낸다.

## 목표
- SSH 키 쌍을 만들고 공개키를 서버에 등록해 비밀번호 없이 접속한다.
- 키 인증에서 파일 권한이 왜 중요한지 이해한다.
- 서버의 비밀번호 로그인을 안전하게 끈다.

## 환경
- VMware Workstation Pro 26H1 / Rocky Linux 9 (Minimal Install)
- `client01`(접속하는 쪽), `server01`(접속받는 쪽), 그리고 Windows PC(PowerShell)

## 전체 흐름

```
① 접속하는 쪽에서 키 쌍을 만든다      (ssh-keygen)
② 공개키를 서버의 authorized_keys에 등록한다   (ssh-copy-id 또는 scp + cat >>)
③ 비밀번호 없이 접속되는지 확인한다
④ 서버에서 비밀번호 로그인을 끈다      (sshd 설정 + 재시작)
⑤ 새 창으로 확인한다 (키 접속 O, 비밀번호 접속 X)
```

## 실습 과정

### 1. 키 만들기 (client01, Windows 각각)
```bash
ssh-keygen -t ed25519
```
- 질문 세 개(저장 위치, 패스프레이즈, 확인)는 모두 Enter. 패스프레이즈를 비워 두면 접속 때 아무것도 묻지 않는다.
- 결과: `~/.ssh/id_ed25519`(개인키), `~/.ssh/id_ed25519.pub`(공개키)
- `known_hosts`: 접속해 본 서버의 지문을 기록하는 파일. 처음 접속할 때 `yes`를 입력하면 생긴다.

### 2. 공개키 등록

**client01 → server01** (`ssh-copy-id` 사용)
```bash
ssh-copy-id cheol@서버IP
```
- 서버의 `~/.ssh` 폴더와 `authorized_keys`를 알아서 만들고 권한까지 맞춰 준다.

**Windows → server01** (`ssh-copy-id`가 없어서 파일을 전송)
```
scp $env:USERPROFILE\.ssh\id_ed25519.pub cheol@서버IP:~/win.pub
```
서버 안에서:
```bash
cat ~/win.pub >> ~/.ssh/authorized_keys
rm ~/win.pub
wc -l ~/.ssh/authorized_keys        # 줄 개수 확인 (키 2개 = 2줄)
```
- `>>`는 덧붙이기, `>`는 덮어쓰기. 덮어쓰면 기존에 등록한 키가 모두 사라진다.

### 3. 권한 확인

| 대상 | 권한 |
|---|---|
| `~/.ssh` 폴더 | `700` |
| 개인키 `id_ed25519` | `600` |
| 공개키 `id_ed25519.pub` | `644` |
| 서버의 `authorized_keys` | `600` |
| 홈 디렉터리 | 다른 사람이 쓸 수 없어야 함 |

- 이 권한이 느슨하면 SSH가 키를 **보안상 무시**한다. (개인키가 다른 사람에게 읽히면 복사될 수 있고, `authorized_keys`를 다른 사람이 쓸 수 있으면 자기 키를 몰래 추가할 수 있기 때문)
- 키 접속이 안 될 때 서버 로그에서 원인을 찾는다.
  ```bash
  sudo grep "bad ownership" /var/log/secure
  ```

### 4. 비밀번호 로그인 끄기 (server01)

**안전 절차**
1. 스냅샷을 찍는다.
2. **지금 접속한 창을 닫지 않는다.** `sshd`를 재시작해도 이미 접속한 창은 유지되므로, 문제가 생기면 이 창에서 되돌린다.
3. 설정 후 **새 창**으로 접속을 확인한다.

**설정 파일 추가**
```bash
echo "PasswordAuthentication no" | sudo tee /etc/ssh/sshd_config.d/10-key-only.conf
```
- `sudo echo ... > 파일`은 `>` 저장이 sudo가 아니라 내 셸에서 일어나서 권한 오류가 난다.
  `| sudo tee 파일`로 넘기면 root 권한으로 파일에 쓸 수 있다.
- `sshd_config.d/` 폴더의 파일은 기본 설정보다 **먼저** 읽히고, 같은 항목은 먼저 읽은 값이 적용된다.

**검증 후 적용**
```bash
sudo sshd -t                                          # 문법 검사 (출력이 없으면 정상)
sudo sshd -T | grep -i passwordauthentication         # 실제 적용될 값: no
sudo systemctl restart sshd
systemctl is-active sshd                              # active
```

### 5. 새 창에서 확인
```
ssh cheol@서버IP                                       # 비밀번호 없이 접속됨 (키 인증)
ssh -o PubkeyAuthentication=no cheol@서버IP            # 키를 일부러 안 쓰면 거부됨
```
```
Permission denied (publickey,gssapi-keyex,gssapi-with-mic).
```
- 괄호 안은 서버가 받는 인증 방식의 목록이다. `password`가 없는 것이 비밀번호 로그인이 꺼졌다는 증거.

**되돌리는 법** (기존 창에서)
```bash
sudo rm /etc/ssh/sshd_config.d/10-key-only.conf
sudo systemctl restart sshd
```

## 막혔던 점과 해결

### ① 공개키를 Windows PC 안의 파일에 추가함
- 서버에 등록하려던 명령(`echo ... >> ~/.ssh/authorized_keys`)을 server01이 아니라 **Windows PowerShell**에서 실행했다.
  PowerShell에서 `~`는 Windows 홈(`C:\Users\이름`)이라서 Windows 쪽 파일에 쌓였다.
- 구분 방법: **프롬프트**를 본다. `PS C:\Users\A>`는 Windows, `[cheol@server01 ~]$`는 server01.
- 해결: Windows의 잘못 만든 파일을 `Remove-Item`으로 지우고, server01에 제대로 등록했다.

### ② 공개키가 세 줄로 깨짐
- 화면에서 복사해 붙이는 과정에서 `종류 / 키 본체 / 꼬리표`가 세 줄로 나뉘었다. 공개키는 **반드시 한 줄**이어야 서버가 읽는다.
- 해결: 복사/붙여넣기 대신 `scp`로 **파일 자체를 전송**한 뒤 서버에서 `cat >>`로 덧붙였다.
- 공개키 한 줄의 구조: `ssh-ed25519`(종류) + `AAAA...`(키 본체) + `꼬리표`(없어도 동작하지만 누구 키인지 구분하는 용도).

### ③ 접속 명령 오타
- `ssh cheol.192.168.61.128`처럼 `@` 대신 `.`을 쓰면 전체가 서버 이름으로 읽혀 `Could not resolve hostname`이 난다. 형식은 `ssh 계정@서버주소`.

### ④ 따옴표 없이 `echo`로 추가
- 공백이 한 칸씩인 공개키는 따옴표 없이도 같은 결과가 나왔지만, 습관적으로 `echo "..." >> 파일`처럼 묶는 것이 안전하다.

## 배운 점
- 키 인증은 **개인키는 내 쪽에만, 공개키는 서버에** 두는 구조이고, 비밀번호보다 안전하다.
- 키 접속이 안 될 때는 **권한 → 줄바꿈/한 줄 여부 → 서버 로그** 순서로 확인한다.
- 설정을 바꿀 때는 **스냅샷, 기존 창 유지, 새 창으로 확인, 되돌리는 법**을 미리 준비한다.
- 명령을 치기 전에 **프롬프트로 어느 머신인지** 확인한다.
- `sshd -t`(문법 검사)와 `sshd -T`(적용될 값 확인)로 재시작 전에 검증한다.
