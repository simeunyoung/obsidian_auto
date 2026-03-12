
`ssh.sh` 스크립트를 매번 `./ssh.sh`로 실행하는 대신  
`list` 명령으로 실행할 수 있도록 alias를 등록한다.

---

# alias 등록

설정 파일

~/.bashrc

추가

alias list='/home/ec2-user/ssh.sh'

---

# 적용

설정 반영

source ~/.bashrc

또는

새 SSH 로그인

---

# 확인

alias 확인

type list

출력

list is aliased to `/home/ec2-user/ssh.sh`

---

# 실행

list

동작

host.list 읽기  
→ 서버 목록 출력  
→ 번호 선택  
→ ssh 접속

---

# 관련 파일

|파일|설명|
|---|---|
|`/home/ec2-user/ssh.sh`|SSH 접속 스크립트|
|`/home/ec2-user/host.list`|접속 서버 목록|
|`~/.bashrc`|alias 설정 파일|

---

# 정리

alias list='/home/ec2-user/ssh.sh'  
→ list 입력  
→ ssh.sh 실행  
→ 서버 선택 후 SSH 접속