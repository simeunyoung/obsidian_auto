
---

**1. 주요 역할 및 활용 목적:**

- **대용량 학습 데이터 저장소:** GB~TB 단위의 영상, 음성, 텍스트 등 모든 학습 데이터의 원본 저장.
    
- **매니페스트 파일 저장소:** 데이터셋의 메타데이터를 담은 매니페스트 파일 저장.
    
- **임시 파일 저장소:** 온디맨드 압축 시 생성되는 임시 ZIP 파일 저장.
    
- **Lambda 트리거 소스:** S3 객체 생성/삭제 이벤트를 통해 Lambda 함수 자동 실행.
    
- **데이터 전송 목적지:** Snowball Edge, EC2 업로드 서버 등으로부터 데이터를 받는 최종 목적지.
    
- **사용자 직접 다운로드:** Pre-signed URL을 통해 사용자가 S3로부터 직접 데이터 다운로드.
    

---

**2. S3 버킷 구조 (예시):**

- s3://your-data-bucket/
    
    - raw-data/: 원본 데이터 (Snowball Edge, EC2로부터 업로드)
        
        - videos/: s3://your-data-bucket/raw-data/videos/UUID_filename.mp4
            
        - audios/: s3://your-data-bucket/raw-data/audios/UUID_filename.wav
            
        - ... (데이터 유형별 폴더)
            
    - manifests/: 매니페스트 파일 (s3://your-data-bucket/manifests/dataset_YYYY_MM_V#.jsonl)
        
    - pre-compressed-datasets/: 미리 압축된 데이터셋 (s3://your-data-bucket/pre-compressed-datasets/gender_age_type_all.zip)
        
    - on-demand-zips/: 온디맨드 압축 시 생성되는 임시 ZIP 파일 (s3://your-data-bucket/on-demand-zips/jobid_userid_timestamp.zip)
        
    - logs/: S3 접근 로그 (선택적)
        

---

**3. S3 핵심 기능 및 설정:**

- **객체 키(Object Key):**
    
    - **역할:** S3 내의 각 파일을 고유하게 식별하는 경로. s3://bucket-name/prefix/key_name.extension
        
    - **활용:** DB 메타데이터의 s3_object_key 필드와 1:1 매핑되는 핵심 연결 고리. 매니페스트 UUID를 객체 키에 포함시켜 고유성 및 연결 용이성 확보.
        
- **S3 Pre-signed URL:**
    
    - **역할:** S3 객체에 대한 임시적, 보안적인 접근 권한을 부여하는 URL.
        
    - **활용:** 사용자 다운로드 시, 백엔드 Lambda가 이 URL을 생성하여 프론트엔드로 전달, 사용자는 S3로부터 직접 다운로드.
        
    - **설정:** 유효 기간(TTL) 필수 설정 (보통 수십분~수시간). ResponseContentDisposition 헤더를 통해 다운로드 시 파일명 지정 가능.
        
- **S3 이벤트 알림 (Event Notifications):**
    
    - **역할:** S3 버킷에서 특정 이벤트(객체 생성, 삭제 등) 발생 시 Lambda 함수, SQS 큐, SNS 토픽 등으로 알림 전송.
        
    - **활용:**
        
        - raw-data/ 경로에 새로운 데이터 객체 생성 시 -> 메타데이터 추출 및 DB 저장 Lambda 트리거 (단, 매니페스트 기반이므로 이 트리거는 매니페스트 파일 업로드 시에만 활용하거나, 아니면 매니페스트 업로드 완료 후 수동 트리거/Batch 작업으로 파일 유효성 검사).
            
        - manifests/ 경로에 매니페스트 파일 생성 시 -> 매니페스트 파싱 및 DB 저장 Lambda 트리거 (권장).
            
        - on-demand-zips/ 경로에 압축 파일 생성 시 -> 다운로드 완료 알림 Lambda 트리거.
            
- **S3 스토리지 클래스 (Storage Classes):**
    
    - **역할:** 데이터 접근 빈도 및 보관 기간에 따라 비용 효율적인 스토리지 선택.
        
    - **활용:**
        
        - raw-data/, pre-compressed-datasets/: Standard (자주 접근), Standard-IA (자주 접근 안 하지만 빠르게 필요), Intelligent-Tiering (자동 최적화).
            
        - 오래된 백업/아카이브 데이터: Glacier, Glacier Deep Archive.
            
- **S3 라이프사이클 관리 (Lifecycle Management):**
    
    - **역할:** 일정 기간 후 스토리지 클래스 변경(비용 절감) 또는 객체 자동 삭제.
        
    - **활용:** on-demand-zips/ 경로의 임시 압축 파일들을 일정 기간(예: 1일 또는 7일) 후 자동으로 삭제하도록 설정.
        
- **보안:**
    
    - **버킷 정책 (Bucket Policy):** 특정 계정/사용자에게만 접근 허용, 퍼블릭 액세스 차단 필수.
        
    - **IAM Policy:** S3에 접근하는 모든 AWS 서비스(Lambda, Batch, EC2)는 최소 권한의 IAM Role을 통해 접근하도록 설정.
        
    - **서버 측 암호화 (Server-Side Encryption):** 모든 S3 객체는 저장 시 자동으로 암호화되도록 설정 (SSE-S3 또는 SSE-KMS).
        
    - **VPC Endpoint for S3:** VPC 내에서 S3에 안전하고 효율적으로 접근 (인터넷 게이트웨이를 거치지 않음).
        

---

**4. S3 데이터 전송/업로드 관련:**

- **AWS Snowball Edge:** 대량 초기 데이터(10TB 이상) 온프레미스 -> S3 전송용 물리 디바이스.
    
- **EC2 업로드 서버:** 중소규모 데이터 온프레미스 -> S3 전송용 중간 서버. (AWS CLI를 통한 멀티파트 업로드 활용)
    
- **AWS DataSync:** 온프레미스 -> S3 안정적이고 빠른 데이터 전송 (DataSync Agent 설치 시).
    
- **AWS Transfer Family:** SFTP/FTPS/FTP를 통해 공급자 -> S3로 직접 업로드 (공급자 측에서 간편하게 업로드 시).
    

---

**5. S3와 다른 서비스의 연동:**

- **Lambda:** S3 이벤트 트리거, S3 객체 읽기/쓰기, Pre-signed URL 생성.
    
- **Aurora DB:** S3 객체 키를 메타데이터의 PK로 사용하여 S3 객체와 연결.
    
- **AWS Batch/EC2:** S3에서 데이터 다운로드/업로드 (압축 작업), 압축된 파일 S3 저장.
    
- **CloudFront:** S3 객체를 CDN으로 캐싱하여 전 세계 사용자에게 빠르게 전송 (선택 사항, 웹사이트 정적 자원 등).
    
- **CloudWatch:** S3 사용량, 요청 수, 오류율 등 모니터링.
    

---