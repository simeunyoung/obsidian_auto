
| **#** | **Controller** | **Summary (API)**                | **Description**     | 필요  | 적용  |
| ----- | -------------- | -------------------------------- | ------------------- | --- | --- |
| User  |                |                                  |                     |     |     |
| 1     | User           | **Join**                         | 회원 가입               | X   | ⬜   |
| 2     | User           | **Withdrawal**                   | 회원 탈퇴               | O   | ⬜   |
| 3     | User           | **Request TempKey**              | 임시 인증 번호 요청         | X   | ⬜   |
| 4     | User           | **Verify TempKey**               | 임시인증번호 유효성 체크       | X   | ⬜   |
| 5     | User           | **Find Id**                      | 아이디 찾기              | X   | ⬜   |
| 6     | User           | **Reser Password**               | 비밀번호 초기화            | X   | ⬜   |
| 7     | User           | **Duplicate Check**              | id, email, tel 중복확인 | X   | ⬜   |
| 8     | User           | **Search User Info**             | 사용자 정보 조회           | O   | ⬜   |
| 9     | User           | **Modify User Info**             | 사용자 정보 편집           | O   | ⬜   |
| 10    | User           | **Search Event History**         | 이벤트 이력 조회           | O   | ⬜   |
| 11    | User           | **Update Event History**         | 이벤트 이력 업데이트         | O   | ⬜   |
| Group |                |                                  |                     |     |     |
| 12    | Group          | **Update Group NickName**        | 그룹 닉네임 편집           | O   | ⬜   |
| 13    | Group          | **Search GroupId**               | 그룹 아이디 조회           | O   | ⬜   |
| 14    | Group          | **Send Group Invitation**        | 그룹 초대               | O   | ⬜   |
| 15    | Group          | **Search Group Invitation**      | 그룹초대 조회 (수신)        | O   | ⬜   |
| 16    | Group          | **Search Send Group Invitation** | 그룹초대 발송 조회          | O   | ⬜   |
| 17    | Group          | **Delete Group Invitation List** | 그룹초대 리스트 삭제         | O   | ⬜   |
| 18    | Group          | **Result Group Invitation**      | 그룹초대 결과 입력          | O   | ⬜   |
| 19    | Group          | **Search Group Member**          | 그룹 구성원 조회           | O   | ⬜   |
| 20    | Group          | **Withdraw Group**               | 그룹 탈퇴               | O   | ⬜   |
| 21    | Group          | **Delete Group Member**          | 그룹 구성원 삭제(추방)       | O   | ⬜   |
