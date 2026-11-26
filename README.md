# linux-lab

리눅스 서버 운영 실습 기록입니다. IT 인프라 직무를 목표로, 가상 머신에 Rocky Linux를 설치하고 서버 운영의 기본을 직접 해 보며 정리합니다.

## 환경
- 호스트: Windows PC (RAM 16GB)
- 가상화: VMware Workstation Pro 26H1
- OS: Rocky Linux 9 (Minimal Install, GUI 없음)
- 구성: `server01`, `client01` 2대 (NAT + Host-only 네트워크), Windows PowerShell에서 SSH로 접속

## 목차
- [01. 사용자와 그룹 관리](01-users-groups.md)
- [02. 파일과 폴더 권한](02-permissions.md)
- [03. 서비스 관리, 로그, 방화벽](03-services-logs-firewall.md)
- [04. SSH 키 인증과 비밀번호 로그인 차단](04-ssh-key-auth.md)
- [05. 총복습: 사용자, 권한, 서비스, 방화벽, SSH 키](05-review.md)
- [06. 디스크 추가, 파티션, 마운트](06-disk-mount.md)
- 이후: 웹서버(nginx), DB 설치와 백업 자동화

## 참고
- [리눅스 치트시트](cheatsheet.md): 실습하며 배운 명령어를 주제별로 정리 (실습이 늘면 같이 갱신)

## 기록 방식
각 문서는 목표, 환경, 실습 과정, 막혔던 점과 해결, 배운 점 순서로 정리합니다.
