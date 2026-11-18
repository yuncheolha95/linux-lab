# 02. 파일과 폴더 권한

## 목표
- `ls -l`로 권한을 읽고, `chmod`와 `chown`으로 바꿀 수 있다.
- 그룹 공유 폴더를 만들고, 사용자를 그룹에 넣고 빼면서 접근이 달라지는 것을 확인한다.

## 환경
- VMware Workstation Pro 26H1 / Rocky Linux 9 (Minimal Install)
- `server01`에서 관리자 계정(`cheol`)으로 SSH 접속해 작업
- 실습용 계정 `testuser`, 그룹 `devteam` ([01. 사용자와 그룹 관리](01-users-groups.md)에서 만든 것)

## 핵심 개념

### 권한 읽는 법
`ls -l`의 맨 앞 10글자는 종류와 권한이다.

```
-rw-r--r--.
```

| 위치 | 의미 | 예시 |
|---|---|---|
| 1번째 | 종류 (`-` 파일, `d` 폴더) | `-` |
| 2~4번째 | 소유자 권한 | `rw-` |
| 5~7번째 | 그룹 권한 | `r--` |
| 8~10번째 | 그 외 사람 권한 | `r--` |

- 숫자로는 r=4, w=2, x=1을 더한다. `rw-`는 6, `r--`는 4이므로 `-rw-r--r--`는 644.
- 맨 끝의 `.`은 SELinux 보안 정보가 붙어 있다는 표시.

### 폴더에서는 rwx의 뜻이 다르다

| 권한 | 파일 | 폴더 |
|---|---|---|
| r | 내용 읽기 | 안의 **목록** 보기 (`ls`) |
| w | 내용 수정 | 안에 파일을 **만들고 지우기** |
| x | 실행 | 안으로 **들어가기** (`cd`) |

- 파일을 지우는 데 필요한 것은 그 파일의 권한이 아니라, 파일이 들어 있는 **폴더의 w 권한**이다.

### chmod와 chown
- `chmod 600 파일`: 권한 변경 (소유자만 읽기/쓰기).
- `chown 소유자:그룹 대상`: 소유자와 그룹 변경 (root 권한 필요).
  - `chown testuser:devteam a.txt`: 소유자와 그룹 모두 변경
  - `chown :devteam /srv/share`: 콜론 앞이 비어 있으면 **그룹만** 변경

## 실습 과정

### A. 파일 권한 읽고 바꾸기
```bash
mkdir ~/lab && cd ~/lab
touch a.txt
ls -l                               # 기본 권한 644 확인
chmod 600 a.txt                     # 소유자만 읽기/쓰기
sudo chown testuser:devteam a.txt   # 소유자와 그룹 변경
cat a.txt                           # 관리자 계정으로는 Permission denied
sudo cat a.txt                      # sudo로는 읽힘
```
- 파일의 주인이 testuser로 바뀌고 권한이 600이면, 관리자 계정이라도 `sudo` 없이는 읽을 수 없다.

### B. 그룹 공유 폴더 만들기
```bash
sudo mkdir /srv/share
sudo chown :devteam /srv/share
sudo chmod 770 /srv/share
ls -ld /srv/share
```
```
drwxrwx---. 2 root devteam ... /srv/share
```
- `/srv`는 서버가 제공하는 데이터를 두는 표준 위치. 개인 홈(`~/lab`)이 아니라 공용 위치에 만들었다.
- 소유자는 root, 그룹은 devteam, 권한은 770 (소유자/그룹 전체 허용, 그 외 차단).
- 이렇게 만들면 폴더 권한은 두고 **그룹에 사람을 넣고 빼는 것만으로** 접근을 관리할 수 있다.

### C. 그룹 소속에 따른 접근 확인
```bash
su - testuser                        # devteam 소속일 때: 폴더에 파일 생성 가능
touch /srv/share/t.txt
ls -l /srv/share
exit

sudo gpasswd -d testuser devteam     # 그룹에서 제외
id testuser

su - testuser                        # 제외한 뒤: 접근 불가 확인
touch /srv/share/t2.txt
exit
```
- 같은 사용자가 그룹 소속 하나만 바뀌었는데 접근 결과가 달라진다.
- 그룹 변경은 이미 로그인한 세션에는 적용되지 않으므로, `su - 사용자`로 새로 들어가서 확인했다.

### D. 정리
```bash
ps -u testuser                       # testuser의 프로세스가 없는지 확인 (헤더만 나오면 OK)
sudo rm -r ~/lab /srv/share
sudo userdel -r testuser
sudo groupdel devteam
ls /srv                              # 비어 있으면 정리 완료
```

## 명령어 정리

### ls, ls -l, ls -ld의 차이

| 명령어 | 보여주는 것 |
|---|---|
| `ls` | 폴더 안의 **이름**만 |
| `ls -l` | 폴더 안 파일들의 **상세 정보** (권한, 소유자, 그룹, 크기, 수정 시간) |
| `ls -ld 폴더` | 폴더 **자체**의 상세 정보 |

- `ls`는 대상이 폴더이면 그 안의 내용을 보여준다. 폴더 자체의 권한을 보려면 `-d`가 필요하다.
- 관리자 계정이 `/srv/share`에 대해 `ls -l`을 하면 폴더 안을 열어야 하므로 `Permission denied`가 나지만,
  `ls -ld`는 폴더 안을 열지 않아서 권한을 볼 수 있다.
- `ls -a`는 숨김 파일(이름이 `.`으로 시작)까지 보여준다.

### echo와 cat의 차이

| 명령어 | 출력하는 것 |
|---|---|
| `echo` | 내가 입력한 **글자** (파일이 필요 없음) |
| `cat` | **파일의 내용** |

```bash
echo hello > b.txt     # hello를 b.txt에 저장
echo b.txt             # b.txt        (글자 그대로)
cat b.txt              # hello        (파일 안의 내용)
```
- `touch 파일`은 출력이 아니라 **빈 파일을 만드는** 명령어다.
- `echo 내용 > 파일`은 파일에 덮어쓰기, `>>`는 끝에 덧붙이기.

### rm의 -r 옵션
- `rm 파일`: 파일 삭제. 폴더에 쓰면 `Is a directory` 에러가 난다.
- `rm -r 폴더`: 폴더와 그 안의 모든 것을 삭제. 휴지통이 없어서 되돌릴 수 없다.
- `rmdir 폴더`: 빈 폴더만 삭제.
- `userdel -r`의 `-r`은 recursive가 아니라 remove로, 홈 폴더까지 함께 삭제한다. 옵션 글자가 같아도 명령어마다 뜻이 다르다.

## 막혔던 점과 해결

### ① `/srv/share`의 소유자를 testuser로 착각
- 소유자는 `root`이고 **그룹만** devteam이다. testuser가 소유자인 것은 `a.txt` 파일이다.
- `sudo mkdir`로 만들면 소유자가 root가 되고, `chown :devteam`은 그룹만 바꾼다.
- 확인: `ls -ld /srv/share`에서 `root devteam` 순서로 나오는지 본다.

### ② 관리자 계정에서 `ls -l /srv/share`가 `Permission denied`
- 오류가 아니라 설계한 대로 동작한 것. 관리자 계정은 devteam 소속이 아니라 "그 외 사람"에 해당하고, 770에서 그 외는 `---`이다.
- 폴더 안의 목록을 읽으려면 r 권한이 필요한데 없기 때문이다. 폴더 자체의 정보는 `ls -ld`로 볼 수 있다.

### ③ `ls -l` 결과가 `total 0`
- `total`은 파일 개수가 아니라 목록 파일들이 차지하는 디스크 블록의 합계이다.
- `touch`로 만든 빈 파일은 공간을 쓰지 않아서 파일이 있어도 `total 0`으로 나온다. 그 아래에 파일 줄이 보이면 정상.

### ④ `/srv`는 권한이 `r-x`인데 `ls /srv`가 에러 없이 실행
- `drwxr-xr-x`에서 그 외 사람의 권한은 `r-x`로, 실행(x)만이 아니라 **읽기(r)도 있다**. 목록 보기는 r 권한이 필요하므로 정상 동작한다.
- `ls /srv` 결과가 비어 있던 것은 에러가 아니라 `/srv/share`를 지워서 안이 비었기 때문이다.
- 반면 w가 없어서 `/srv`에 폴더를 새로 만드는 것은 불가능하다.

### ⑤ 다른 사용자로 전환한 셸의 명령은 history에 남지 않는다
- `su - testuser`로 들어가서 친 명령(`touch /srv/share/t.txt` 등)은 관리자 계정의 `history` 출력에 보이지 않았다.
- history는 사용자별, 셸별로 따로 관리된다. 기록을 남길 때는 어느 계정에서 쳤는지 따로 적어 두어야 한다.
- `su 사용자`(`-` 없음)는 현재 위치와 환경이 그대로 남고, `su - 사용자`는 그 사용자가 새로 로그인한 것과 같은 환경이 된다.

## 배운 점
- 권한은 소유자 / 그룹 / 그 외 세 부류 × r, w, x이고, 숫자(4, 2, 1)로 읽고 쓸 수 있다.
- 폴더에서 r은 목록, w는 만들고 지우기, x는 들어가기이다.
- 공유 폴더는 개인 홈이 아니라 공용 위치에 두고, **그룹으로 접근을 관리**한다. 사람을 그룹에 넣고 빼는 것만으로 권한이 바뀐다.
- 권한 문제는 `ls -ld 폴더`로 소유자/그룹/권한을 보고, `id 사용자`로 그룹 소속을 확인하는 순서로 점검한다.
- 삭제는 되돌릴 수 없으니 `rm -r`을 치기 전에 경로를 한 번 더 읽는다. 사용자를 지우기 전에는 `ps -u`로 실행 중인 프로세스를 확인한다.
