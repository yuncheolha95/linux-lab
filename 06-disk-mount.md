# 06. 디스크 추가, 파티션, 마운트

## 한 줄 요약
가상 디스크 5GB를 추가해 **파티션 → 파일시스템(xfs) → 마운트 → `/etc/fstab` 등록**까지 하고, 재부팅 후에도 `/mnt/data`에 자동으로 연결되는 것을 확인했다.

## 쉬운 비유
- 디스크 추가 = 빈 땅이 생김
- 파티션 = 땅을 구획으로 나눔
- 파일시스템 = 구획에 선반을 설치함
- 마운트 = 건물(디렉터리 구조)에 출입문을 임시로 단다
- `/etc/fstab` = 설계도에 출입문을 정식으로 그려 넣음 (재부팅해도 유지)

## 목표
- 새 디스크를 인식하고 파티션과 파일시스템을 만든다.
- 디렉터리에 마운트하고, 재부팅 후에도 유지되도록 `fstab`에 등록한다.
- `fstab` 오류를 재부팅 전에 `mount -a`로 검증하는 습관을 들인다.

## 환경
- VMware Workstation Pro 26H1 / Rocky Linux 9 (Minimal Install)
- `server01`에 5GB 가상 디스크 추가 (VMware가 NVMe 방식으로 인식)

## 전체 흐름

```
lsblk (확인) → fdisk (파티션) → mkfs.xfs (파일시스템) → mkdir + mount (연결)
→ blkid로 UUID 확인 → /etc/fstab 등록 → umount → mount -a (테스트) → 재부팅 후 df -h
```

## 실습 과정

| 단계 | 명령어 | 확인한 것 |
|---|---|---|
| 0 | VMware → Add → Hard Disk (5GB), 스냅샷 `before-disk` | 설정에 새 디스크가 보임 |
| 1 | `lsblk` | `nvme0n2` 5G, 파티션 없음 |
| 2 | `sudo fdisk /dev/nvme0n2` → `n` → Enter 반복 → `w` | `nvme0n2p1` 생성 |
| 3 | `sudo mkfs.xfs /dev/nvme0n2p1` | `lsblk -f`에 `xfs`와 UUID |
| 4 | `sudo mkdir /mnt/data` → `sudo mount /dev/nvme0n2p1 /mnt/data` → `df -h` | `/mnt/data`에 5G |
| 5 | `sudo cp /etc/fstab /etc/fstab.bak` → `sudo blkid` → `sudo vi /etc/fstab` | 맨 아래에 한 줄 추가 |
| 6 | `sudo umount /mnt/data` → `sudo mount -a` → `df -h` | 메시지 없이 다시 마운트 |
| 7 | `sudo reboot` → `df -h` | 직접 `mount`하지 않아도 `/mnt/data`가 보임 |

`fstab`에 추가한 줄 (6칸):
```
UUID=<UUID 값>   /mnt/data   xfs   defaults   0 0
```

| 칸 | 값 | 뜻 |
|---|---|---|
| 1 | `UUID=...` | 어떤 장치 (이름은 바뀔 수 있어도 UUID는 그대로) |
| 2 | `/mnt/data` | 마운트할 디렉터리 |
| 3 | `xfs` | 파일시스템 종류 |
| 4 | `defaults` | 옵션 |
| 5, 6 | `0 0` | 백업 여부, 부팅 시 검사 순서 |

재부팅 후 결과:
```
/dev/mapper/rl-root   17G  ...  /
/dev/nvme0n1p1       960M  ...  /boot
/dev/nvme0n2p1       5.0G  ...  /mnt/data
```

## 막혔던 점과 해결

### 1. 디스크 이름이 `sdb`가 아니라 `nvme0n2`
- VMware에서 디스크가 NVMe 방식으로 인식돼 이름이 달랐다. OS 디스크는 `nvme0n1`, 새 디스크는 `nvme0n2`.
- 파티션 이름에는 **`p`**가 붙는다: `nvme0n2p1`. (`sdb1`과 달리 `nvme0n21`이 아니다.)
- 이름이 한 글자 차이라서 `mkfs` 전에 `lsblk`로 `n2`인지 확인했다.

### 2. `mount -a` 에러: `wrong fs type, bad option, bad superblock`
```
mount: /mnt/data: wrong fs type, bad option, bad superblock on /dev/nvme0n2p1, ...
```
- 원인: `fstab` 줄이 **5칸**이었다. 4번째 칸(`defaults`)이 빠져서 `0`이 옵션 자리로 밀려 들어갔고, 시스템이 `0`을 마운트 옵션으로 읽어 `bad option`이 났다.
  ```
  UUID=...  /mnt/data  xfs  0 0           ← 잘못 (5칸)
  UUID=...  /mnt/data  xfs  defaults  0 0 ← 올바름 (6칸)
  ```
- 에러 메시지에 장치 이름(`/dev/nvme0n2p1`)이 나온 것은 UUID는 제대로 찾았다는 뜻이었다. 문제는 그 뒤의 종류/옵션 칸이었다.
- 해결: `vi`로 `defaults`를 추가하고 `mount -a`를 다시 실행했다.
- 교훈: 에러가 나면 `fstab` 줄의 **칸 수를 세어 본다**. 이 문제를 재부팅 전에 `mount -a`로 잡았다. 그대로 재부팅했다면 부팅이 멈출 수 있었다.

### 3. 힌트 메시지
```
mount: (hint) your fstab has been modified, but systemd still uses the old version; use 'systemctl daemon-reload' to reload.
```
- 에러가 아니라 안내다. `sudo systemctl daemon-reload`로 systemd가 `fstab`을 다시 읽게 하면 사라진다.
- `fstab` 파일을 저장(`:wq`)한 것이 영구 저장이고, `daemon-reload`는 지금 실행 중인 systemd에 알려 주는 보조 명령이다. 재부팅하면 systemd는 `fstab`을 새로 읽는다.

## 배운 점
- **`mount`는 임시, `fstab`은 영구**: 재부팅해도 유지하려면 `/etc/fstab`에 등록한다.
- **재부팅 전 테스트**: `umount` 후 `mount -a`. 이미 연결된 것은 건너뛰므로 먼저 연결을 풀어야 `fstab`을 제대로 시험할 수 있다. 메시지가 없으면 정상.
- **`fstab` 수정 습관**: 백업(`.bak`) → 기존 줄은 그대로 두고 끝에 한 줄만 추가 → `cat`으로 6칸 확인 → `mount -a` → 재부팅. `tee`로 추가할 때 `-a`를 빼면 파일 전체가 덮어써진다.
- **UUID를 쓰는 이유**: 장치 이름(`sdb`, `nvme0n2`)은 디스크 순서가 바뀌면 달라질 수 있지만 UUID는 변하지 않는다. `fstab`에는 `UUID=값` 형태로 쓴다 (값만 쓰면 안 되고, 따옴표와 `=` 주변 공백은 넣지 않는다).
- **`mkfs`는 데이터를 지운다**: 실행 전 `lsblk`로 장치를 확인한다. 경고가 떠도 `-f`를 습관으로 붙이지 않는다. 지워진 데이터는 정상 복구가 어렵다.
- **마운트하지 않은 디렉터리는 상위 마운트를 따라간다**: 대부분 루트(`/`) 디스크에 저장되며 `df -h 경로`로 확인한다. 마운트는 용량, 성능, 장애 분리가 필요할 때의 예외다.
- **마운트하면 그 디렉터리의 기존 내용이 가려진다** (삭제되지 않음). 빈 디렉터리에 마운트한다. 마운트한 디렉터리의 소유자는 root라 일반 계정이 쓰려면 `chown`이 필요하다.
- 용어: `/`(루트 디렉터리, 폴더 구조의 꼭대기), root 사용자(슈퍼유저), `/root`(root의 홈)는 서로 다른 개념이다.
