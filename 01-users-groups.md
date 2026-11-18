# 01. 사용자와 그룹 관리

## 목표
- 사용자와 그룹을 만들고, 그룹에 넣고 빼고, 삭제하는 흐름을 익힌다.
- `/etc/passwd`, `/etc/group`, `/etc/shadow`가 각각 무엇을 담는지 이해한다.

## 환경
- VMware Workstation Pro 26H1 (Windows 호스트)
- Rocky Linux 9 (Minimal Install), VM 2대 (`server01`, `client01`)
- Windows PowerShell에서 SSH로 접속해 작업

## 핵심 개념

| 파일 | 내용 |
|---|---|
| `/etc/passwd` | 사용자 목록 (이름, UID, GID, 홈 디렉터리, 로그인 셸) |
| `/etc/group` | 그룹 목록 |
| `/etc/shadow` | 해시된 비밀번호와 만료 정책 (root만 읽기 가능) |

- `/etc/passwd`의 두 번째 칸이 `x`인 이유: 비밀번호는 `/etc/shadow`에 따로 보관한다는 표시.
  누구나 읽을 수 있는 파일에 비밀번호 해시를 두지 않기 위해 분리했다.
- `sudo`는 root 비밀번호가 아니라 **내 비밀번호**를 묻는다. 관리자 그룹(`wheel`) 소속인지 확인하고,
  누가 어떤 관리자 명령을 썼는지 로그가 남는다.

## 실습 과정

### 1. 사용자 만들기와 비밀번호 설정
```bash
sudo useradd testuser
sudo passwd testuser
```
- `useradd`: 계정과 홈 디렉터리(`/home/testuser`), 같은 이름의 기본 그룹이 생긴다. 비밀번호가 없어 로그인은 불가.
- `passwd`: 비밀번호 설정. 입력해도 화면에 아무것도 안 보이는 게 정상.

### 2. 그룹 만들기와 사용자 추가
```bash
sudo groupadd devteam
sudo usermod -aG devteam testuser
id testuser
```
- `-G`: 추가로 소속시킬 그룹, `-a`: 기존 그룹을 유지하고 덧붙이기.
- `-a` 없이 `-G`만 쓰면 지정한 그룹으로 목록이 대체되므로 주의.
- 같은 일을 `sudo gpasswd -a testuser devteam`으로도 할 수 있다.

### 3. 정보 확인
```bash
id testuser
grep testuser /etc/passwd
grep devteam /etc/group
getent group devteam
```
- `cat`은 파일 전체를, `grep`은 찾는 글자가 들어 있는 줄만 출력한다.
- 파일 하나를 검색할 때는 `grep 글자 파일`로 충분하고, 명령어 출력을 거를 때 `|`를 쓴다.

### 4. 다른 사용자로 전환
```bash
su - testuser
whoami
exit
```
- `su - 사용자`: 그 사용자가 새로 로그인한 것처럼 환경까지 전환. 돌아올 때는 `exit`.

### 5. 그룹에서 제외
```bash
sudo gpasswd -d testuser devteam
id testuser
```
- 그룹 변경은 이미 로그인한 세션에는 바로 적용되지 않고, 다시 로그인해야 반영된다.

### 6. 그룹과 사용자 삭제
```bash
sudo groupdel devteam
sudo userdel testuser        # 계정만 삭제 (홈 폴더 유지)
sudo userdel -r testuser     # 계정 + 홈 폴더 삭제
```
- `groupdel`은 그룹 정보만 지운다. 그 그룹 소유 파일은 남고, 그룹 칸이 GID 숫자로 보인다.
- 사용자가 기본 그룹으로 쓰는 그룹은 삭제할 수 없다.

## 막혔던 점과 해결

### ① `usermod: cannot lock /etc/passwd`
- 원인: `sudo`를 빼고 실행 (`usermod -aG devteam testuser`). 시스템 파일을 수정하려면 관리자 권한이 필요하다.
- 해결: `sudo usermod -aG devteam testuser`

### ② 그룹 이름 오타
- `dvteam`으로 잘못 입력. 그룹이 존재하는지 `getent group devteam`으로 확인하고 다시 입력했다.

### ③ 파일 경로를 명령어처럼 입력
- `/etc/passwd`만 입력하면 아무 내용도 나오지 않는다. 파일 이름이지 명령어가 아니다.
- 내용을 보려면 `cat /etc/passwd`처럼 명령어와 함께 사용. (`/ect/passwd` 같은 오타도 주의)
- `pwd`(현재 위치 출력), `passwd`(비밀번호 변경), `/etc/passwd`(사용자 목록 파일)는 서로 다르다.

### ④ `userdel: user testuser is currently used by process`
- 원인: `su - testuser`로 전환한 뒤 `exit` 없이 `su - cheol`로 다시 전환해서, testuser의 셸이 뒤에 남아 있었다.
  전환은 겹쳐서 쌓이기 때문에 돌아갈 때는 `exit`를 사용해야 한다.
- 해결: `exit`로 정리하거나, `sudo pkill -u testuser`로 프로세스를 끊은 뒤 `sudo userdel testuser`.
- 삭제 확인: `grep testuser /etc/passwd`에 아무것도 나오지 않으면 성공.

## 배운 점
- 계정 정보는 `/etc/passwd`, 비밀번호 해시는 `/etc/shadow`에 나뉘어 있고, shadow는 root(sudo)만 읽을 수 있다.
- `useradd` → `passwd` → `groupadd` → `usermod -aG`로 이어지는 기본 흐름과, `id`로 결과를 확인하는 습관.
- 에러 메시지에 원인이 적혀 있으니 먼저 읽는다. 사용자를 지우기 전에는 로그인 중인 세션이 있는지 확인한다.
- `-a` 옵션 하나(`usermod -aG`)가 "추가"와 "대체"를 가른다.
