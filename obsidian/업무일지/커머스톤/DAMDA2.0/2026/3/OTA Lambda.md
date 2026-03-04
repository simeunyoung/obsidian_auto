
#### HandleJobStatus
- ## `fetch_job_executions(job_id)`
	- ==이 job이 어떤 Thing들에서 어떤 상태인지”를 AWS에서 가져오는 함수==
	- AWS IoT Jobs에서 특정 `job_id`의 Job Execution 목록을 전부 조회(페이지네이션 포함) 해서 반환
- ## `update_job_executions(conn)`
	- ==AWS IoT Jobs의 실행 상태를 주기적으로 긁어서 DB에 동기화 하는 함수==
	- DB에서 완료되지 않은(job completed=0) 펌웨어 Job 들을 찾고, 각 Job의 실행 상태를 AWS에서 가져와서 `iot_device_firmware` 테이블에 반영(INSERT/UPDATE)

#### HandleIoTJobCreate