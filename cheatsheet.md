# 리눅스 치트시트 (Rocky Linux 9)

실습하면서 배운 명령어를 주제별로 모았습니다. 실습 문서가 늘어나면 같이 갱신합니다.

---

## 1. 기본 이동과 확인

| 명령어 | 하는 일 |
|---|---|
| `pwd` | 현재 위치 출력 |
| `whoami` | 현재 로그인한 계정 이름 |
| `ls` | 폴더 안의 이름만 |
| `ls -l` | 폴더 안 파일들의 상세 정보 (권한, 소유자, 그룹, 크기, 시간) |
| `ls -ld 폴더` | **폴더 자체**의 상세 정보 |
| `ls -a` | 숨김 파일(`.`으로 시작)까지 |
| `cd 경로` / `cd ~` / `cd ..` | 이동 / 홈으로 / 한 단계 위로 |
| `ip -br a` | 네트워크 어댑터별 IP를 한 줄씩 요약 |

**경로 표기**: `~` = 내 홈(`/home/계정`), `/` = 최상위, `.` = 현재 위치, `..` = 한 단계 위

## 2. 파일 다루기

| 명령어 | 하는 일 |
|---|---|
| `touch 파일` | 빈 파일 생성 (있으면 수정 시간만 갱신) |
| `mkdir 폴더` | 폴더 생성 |
| `cat 파일` | 파일 **전체 내용** 출력 |
| `echo 글자` | 입력한 **글자**를 출력 |
| `echo 내용 > 파일` | 내용을 파일에 저장 (덮어쓰기), `>>`는 끝에 덧붙이기 |
| `echo 내용 \| sudo tee 파일` | **관리자 권한이 필요한 파일**에 저장 (덮어쓰기) |
| `echo 내용 \| sudo tee -a 파일` | 관리자 권한이 필요한 파일 끝에 덧붙이기 |
| `grep 글자 파일` | 파일에서 글자가 든 **줄만** 출력 (`-i` 대소문자 무시, `-v` 제외) |
| `명령어 \| grep 글자` | 명령어의 출력에서 걸러내기 |
| `rm 파일` | 파일 삭제 (되돌릴 수 없음) |
| `rm -r 폴더` | 폴더와 안의 모든 것 삭제 |
| `rmdir 폴더` | 빈 폴더만 삭제 |
| `history \| tail -80` | 최근에 친 명령어 80개 |

- 파일 하나 검색은 `grep 글자 파일`, 명령어 출력 검색은 `명령어 | grep 글자`.

**sudo tee가 필요한 이유**
```bash
sudo echo "test" > /srv/proj/report.txt          # 실패 (Permission denied)
echo "test" | sudo tee /srv/proj/report.txt      # 성공
```

| 부분 | 하는 일 |
|---|---|
| `echo "test"` | `test`라는 글자를 내보냄 |
| `\|` (파이프) | 그 글자를 다음 명령으로 넘김 |
| `sudo tee 파일` | 넘어온 글자를 **관리자 권한으로 파일에 씀** |

- 첫 번째가 실패하는 이유: `sudo`는 `echo`에만 적용되고, 파일에 쓰는 `>`는 **내 쉘(일반 사용자 권한)**이 처리하기 때문.
- 두 번째는 파일에 쓰는 쪽(`tee`)에 sudo가 붙어 있어 권한이 맞다.
- `tee`는 받은 글자를 **파일에도 쓰고 화면에도 출력**한다 (T자처럼 두 갈래). 화면 출력이 싫으면 끝에 `> /dev/null`.
- `sudo tee 파일` = 덮어쓰기(`>`와 같음), `sudo tee -a 파일` = 덧붙이기(`>>`와 같음). `-a`를 빼면 기존 내용이 지워진다.
- 대안: 그 폴더에 쓸 권한이 있는 계정으로 `su - 계정` 후 `echo > 파일`.
- `rm -r`은 경로를 한 번 더 읽고 Enter. `rm -rf`는 습관적으로 쓰지 않는다.

## 3. 따옴표 규칙

- 셸은 **공백으로 단어를 나눠서** 인자로 넘긴다. **값 안에 공백이 있으면 따옴표로 묶는다.**
  ```bash
  grep "Failed password" /var/log/secure
  journalctl --since "1 hour ago"
  ```
- 단어 하나는 없어도 되고, 헷갈리면 붙여도 해가 없다.
- `"..."`는 `$변수`를 해석하고, `'...'`는 글자 그대로 둔다.

## 4. 사용자와 그룹

| 하고 싶은 일 | 명령어 |
|---|---|
| 사용자 만들기 | `sudo useradd 이름` |
| 비밀번호 설정 | `sudo passwd 이름` |
| 그룹 만들기 | `sudo groupadd 그룹` |
| 그룹에 **추가** | `sudo usermod -aG 그룹 사용자` 또는 `sudo gpasswd -a 사용자 그룹` |
| 그룹에서 **제외** | `sudo gpasswd -d 사용자 그룹` (그룹 이름까지 써야 한다) |
| 그룹 멤버를 한 번에 지정 | `sudo gpasswd -M alice,bob 그룹` (기존 멤버는 덮어써짐) |
| 사용자 삭제 | `sudo userdel 이름` (계정만) / `sudo userdel -r 이름` (홈 폴더까지) |
| 그룹 삭제 | `sudo groupdel 그룹` |
| 소속 확인 | `id 사용자` |
| 시스템에 있는지 확인 | `getent passwd 이름`, `getent group 그룹` |
| 다른 사용자로 전환 | `su - 이름` (돌아올 때 **`exit`**) |

- `usermod -G`는 `-a` 없이 쓰면 그룹 목록을 **대체**한다. 추가할 때는 항상 `-aG`.
- 그룹 변경은 이미 로그인한 세션에는 적용되지 않는다. 다시 로그인한다.
- 사용자 삭제 전에 `ps -u 이름`으로 실행 중인 프로세스가 없는지 확인한다. (`su` 중첩 주의)
- `sudo`는 **내 계정의 비밀번호**를 묻는다. root 비밀번호는 `su -`에서만 쓴다.

**관련 파일**: `/etc/passwd`(계정 목록, 두 번째 칸 `x`), `/etc/group`(그룹 목록), `/etc/shadow`(해시된 비밀번호, root만 읽기 가능, 직접 편집 금지)

## 5. 파일 권한

```
-rw-r--r--.   ← 종류 / 소유자 rw- / 그룹 r-- / 그 외 r--
```

| 숫자 | 권한 | | 예시 |
|---|---|---|---|
| 4 | r (읽기) | | 600 = `rw-------` |
| 2 | w (쓰기) | | 644 = `rw-r--r--` |
| 1 | x (실행) | | 770 = `rwxrwx---` |

| 권한 | 파일 | **폴더** |
|---|---|---|
| r | 내용 읽기 | 안의 **목록** 보기 |
| w | 내용 수정 | 안에 파일을 **만들고 지우기** |
| x | 실행 | 안으로 **들어가기** |

| 명령어 | 하는 일 |
|---|---|
| `chmod 600 파일` | 권한 변경 |
| `sudo chown 소유자:그룹 대상` | 소유자와 그룹 변경 |
| `sudo chown :그룹 대상` | **그룹만** 변경 (콜론 앞을 비움). 파일 소유자가 자기가 속한 그룹으로 바꿀 때는 sudo 불필요 (`chgrp 그룹 파일`도 같음) |

**권한 문제 점검 순서**: `ls -ld 폴더`(소유자/그룹/권한) → `id 사용자`(그룹 소속) → 필요하면 그룹에 추가

## 6. 서비스 관리 (systemctl)

| 하고 싶은 일 | 명령어 |
|---|---|
| 상태 보기 | `systemctl status 서비스` (`q`로 나오기) |
| 한 단어로 확인 | `systemctl is-active 서비스`, `systemctl is-enabled 서비스` |
| 지금 켜기 / 끄기 | `sudo systemctl start 서비스` / `stop` |
| 재시작 (설정 변경 후) | `sudo systemctl restart 서비스` |
| 부팅 시 자동 시작 켜기 / 끄기 | `sudo systemctl enable 서비스` / `disable` |
| 자동 시작 + 지금 바로 시작 | `sudo systemctl enable --now 서비스` |

| | 지금 상태 | 부팅 후 |
|---|---|---|
| start / stop | 바꿈 | 영향 없음 |
| enable / disable | 영향 없음 | 바꿈 |

- `status`는 사람용, `is-active`/`is-enabled`는 스크립트용 (`if systemctl is-active --quiet 서비스`).
- `status`, `is-active`는 `sudo` 불필요, 서비스를 바꾸는 명령은 필요.
- 서비스가 꺼지면 새 접속은 `Connection refused`. (이미 접속한 SSH 창은 유지된다.)

## 7. 로그

| 알고 싶은 것 | 이렇게 |
|---|---|
| 서비스가 왜 안 켜지나, 왜 죽었나 | `sudo journalctl -u 서비스 -n 50` |
| 특정 시간대 | `sudo journalctl -u 서비스 --since "1 hour ago"` |
| 이번 부팅 이후 | `sudo journalctl -b` |
| 실시간으로 따라가기 | `sudo journalctl -u 서비스 -f` / `sudo tail -f 파일` (`Ctrl+C`로 종료) |
| 누가 로그인했나, sudo 기록 | `sudo grep "Accepted" /var/log/secure`, `sudo grep "Failed password" /var/log/secure` |
| 서비스가 남긴 기록 (접속, 요청) | 그 서비스의 로그 파일 (`/var/log/...`) |

- **서비스 자체가 문제 → `journalctl -u`**, **서비스가 처리한 내용 → 로그 파일**
- 장애 점검 순서: `systemctl status 서비스`(아래에 최근 로그) → `journalctl -u 서비스 -n 50` → `ls /var/log/`
- 로그 한 줄 읽는 법: **시간 → 호스트 → 프로그램[PID] → 내용**

## 8. 방화벽 (firewalld)

```
sudo firewall-cmd  [동작]  [대상]  [--permanent]
```

| 하고 싶은 일 | 명령어 |
|---|---|
| 상태 보기 | `sudo firewall-cmd --list-all` |
| 포트 열기 | `--add-port=번호/tcp` |
| 서비스 이름으로 열기 | `--add-service=이름` |
| 포트 닫기 | `--remove-port=번호/tcp` |
| 서비스 닫기 | `--remove-service=이름` |
| 적용 | `sudo firewall-cmd --reload` |

**열기 루틴 (저장 → 적용 → 확인)**
```bash
sudo firewall-cmd --add-port=8080/tcp --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```
**닫기 루틴**: `add`를 `remove`로 바꾼다. 연습이 끝나면 열어 둔 포트는 닫는다.

- `--permanent`는 저장만 하고 `--reload`를 해야 적용된다. `--permanent` 없이 추가하면 바로 적용되지만 reload/재부팅 후 사라진다.
- 포트를 열어도 그 포트에서 기다리는 **서비스가 떠 있어야** 접속된다.
- 접속이 안 될 때: `systemctl is-active 서비스` → `sudo ss -tlnp`(포트 대기 확인) → `sudo firewall-cmd --list-all`

## 9. SSH 키 인증

- **개인키**(`id_ed25519`)는 접속하는 쪽에만, **공개키**(`id_ed25519.pub`)는 서버의 `~/.ssh/authorized_keys`에 한 줄로 등록한다.

| 하고 싶은 일 | 명령어 |
|---|---|
| 키 쌍 만들기 | `ssh-keygen -t ed25519` (질문은 Enter) |
| 공개키 내용 확인 (Windows) | `type $env:USERPROFILE\.ssh\id_ed25519.pub` |
| 공개키 내용 확인 (리눅스) | `cat ~/.ssh/id_ed25519.pub` |
| 공개키 등록 (리눅스) | `ssh-copy-id 계정@서버IP` |
| 공개키 등록 (Windows) | `scp $env:USERPROFILE\.ssh\id_ed25519.pub 계정@서버IP:~/win.pub` 후 서버에서 `cat ~/win.pub >> ~/.ssh/authorized_keys` |
| 등록된 키 개수 확인 | `wc -l ~/.ssh/authorized_keys` |
| 키를 안 쓰고 접속 시도 | `ssh -o PubkeyAuthentication=no 계정@서버IP` |
| 접속 과정 자세히 보기 | `ssh -v 계정@서버IP` |

**명령어 해석: 공개키 출력 (Windows)**
```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub
```

| 부분 | 뜻 |
|---|---|
| `type` | 파일 내용을 화면에 출력 (리눅스의 `cat`과 같은 역할. PowerShell에서는 `cat`도 같은 결과) |
| `$env:USERPROFILE` | 내 사용자 폴더 경로가 담긴 환경 변수 (예: `C:\Users\A`). 리눅스의 `~`와 비슷하며, 사용자 이름이 달라도 같은 명령이 통한다 |
| `\.ssh\` | 키가 저장되는 폴더 |
| `id_ed25519.pub` | 공개키 파일 (`.pub` = public) |

- 풀어 쓰면 `type C:\Users\A\.ssh\id_ed25519.pub`
- **왜 출력하나**: 공개키가 제대로 만들어졌는지(`ssh-ed25519 AAAA... 주석` 한 줄) 눈으로 확인하려고.
- 공개키는 보여 줘도 안전하다. **개인키(`id_ed25519`)는 절대 출력해서 복사하거나 공유하지 않는다.**
- 서버로 옮길 때는 복사·붙여넣기(줄바꿈이 끼어 키가 깨질 수 있음) 대신 `scp`를 쓴다.

| 대상 | 권한 |
|---|---|
| `~/.ssh` 폴더 | `700` |
| 개인키 | `600` |
| 공개키 | `644` |
| `authorized_keys` | `600` |

**비밀번호 로그인 끄기** (안전 절차: 스냅샷, 기존 창 유지, 새 창으로 확인)
```bash
echo "PasswordAuthentication no" | sudo tee /etc/ssh/sshd_config.d/10-key-only.conf
sudo sshd -t                                     # 문법 검사 (출력 없으면 정상)
sudo sshd -T | grep -i passwordauthentication    # 적용될 값 확인
sudo systemctl restart sshd
```
되돌리기: `sudo rm /etc/ssh/sshd_config.d/10-key-only.conf` 후 `sudo systemctl restart sshd`

- `sudo echo ... > 파일`은 `>`가 sudo 없이 실행돼 실패한다. `| sudo tee 파일`을 쓴다.
- `>`는 덮어쓰기, `>>`는 덧붙이기. `authorized_keys`에는 항상 `>>`.
- 프롬프트로 어느 머신인지 확인한다. (`PS C:\...>` = Windows, `[계정@server01 ~]$` = server01)

## 10. 터미널 단축키

| 키 | 동작 |
|---|---|
| `Tab` | 명령어/파일 이름 자동완성 (두 번 누르면 후보 목록) |
| `↑` / `↓` | 이전 / 다음 명령어 |
| `Ctrl + R` | 과거 명령어 검색 |
| `Ctrl + C` | 실행 중인 명령 중단 |
| `Ctrl + A` / `Ctrl + E` | 줄 맨 앞 / 맨 끝으로 |
| `Ctrl + U` | 커서 앞쪽 지우기 |
| `Ctrl + L` | 화면 지우기 |
| `sudo !!` | 직전 명령을 `sudo`를 붙여 다시 실행 |

- 옵션 자동완성은 `bash-completion`이 필요하다. (`rpm -q bash-completion`, 없으면 `sudo dnf install -y bash-completion` 후 재로그인)
- 모르는 옵션은 `명령어 --help`, `man 명령어`로 확인한다.

## 11. 자주 쓰는 에러 메시지

| 메시지 | 보통의 원인 |
|---|---|
| `Permission denied` | 권한 부족. `ls -ld`로 권한 확인, 필요하면 `sudo` |
| `cannot lock /etc/passwd` | 계정 명령에 `sudo`를 빼먹음 |
| `user ... is currently used by process` | 그 사용자의 로그인 세션/프로세스가 남아 있음 (`ps -u`, `exit`) |
| `Is a directory` | 폴더를 `rm`할 때 `-r`을 빼먹음 |
| `command not found` | 명령어 오타, 또는 파일 경로만 입력, 또는 `sudo -tee`처럼 명령 이름 앞에 대시를 붙임 |
| `No such file or directory` | 경로 오타, 또는 공백 때문에 인자가 쪼개짐 (따옴표 확인) |
| `Connection refused` | 접속하려는 포트에서 기다리는 서비스가 없음 (서비스 꺼짐) |
| `Could not resolve hostname` | `계정@서버IP`에서 `@`를 빼먹거나 서버 주소 오타 |
| `UNPROTECTED PRIVATE KEY FILE` | 개인키 권한이 느슨함 (`chmod 600`) |
| 키를 등록했는데 계속 비밀번호를 물음 | 권한(`~/.ssh` 700, `authorized_keys` 600), 공개키가 한 줄인지, `/var/log/secure`의 `bad ownership` 확인 |
| `Permission denied (publickey,...)` | 서버가 키 인증만 허용 중인데 키가 없거나 맞지 않음 |
