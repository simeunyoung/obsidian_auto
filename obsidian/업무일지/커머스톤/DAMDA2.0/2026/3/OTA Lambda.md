
#### HandleJobStatus
- `fetch_job_executions(job_id)`
	- ==이 job이 어떤 Thing들에서 어떤 상태인지”를 AWS에서 가져오는 함수==
	- AWS IoT Jobs에서 특정 `job_id`의 Job Execution 목록을 전부 조회(페이지네이션 포함) 해서 반환
- `update_job_executions(conn)`
	- ==AWS IoT Jobs의 실행 상태를 주기적으로 긁어서 DB에 동기화 하는 함수==
	- DB에서 완료되지 않은(job completed=0) 펌웨어 Job 들을 찾고, 각 Job의 실행 상태를 AWS에서 가져와서 `iot_device_firmware` 테이블에 반영(INSERT/UPDATE)

#### HandleIoTJobCreate
- extract_manufacture_model(job_id)
	- `job_id` 문자열에서 제조사/모델/버전 추출
- find_device_model_id(manufacture, model, conn)
	- 제조사/모델로 `iot_device_model.id` 조회
- thing_name_to_parts(thing_name)
	- ThingName(사물 이름)에서 **제조사/모델/uuid** 분리
- map_to_device_id(conn, manufacture, model, uuid)
	- Thing의 uuid를 DB의 serial로 변환해서 device_id 생성
- get_latest_active_model_firmware_id(conn, manufacture, model)
	- 해당 제조사/모델에 대해 현재 진행 중(rolling)인 firmware job의 model_firmware_id(iot_model_firmware.id) 를 찾음
- handle_create_job(detail, conn)
	- CloudTrail의 reateJob 이벤트를 받아 `iot_model_firmware`에 “잡 메타”를 저장
- ensure_queued_row_no_unique(conn, device_id, model_firmware_id)
	- `iot_device_firmware`에 초기 상태(QUEUED) row를 생성
- handle_add_thing_to_group(detail, conn)
	- CloudTrail의 AddThingToThingGroup 이벤트를 받아 “신규 등록 디바이스”를 OTA 대상에 바로 올림(QUEUED 생성)
- handle_delete_job(detail, conn)
	- CloudTrail의 **DeleteJob 이벤트** 처리 → `iot_model_firmware`에서 해당 job_id 제거
- lambda_handler(event, context)
	- 들어온 이벤트를 보고 어떤 핸들러를 실행할지 라우팅 + 예외/rollback/close 관리