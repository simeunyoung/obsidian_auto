

#### NMI
	하드웨어가 “심각한 오류가 났다”는 걸 CPU에 알려줄 때 쓰는 긴급 신호

#### Transparent Huge Pages(THP)
	- 리눅스 커널 기능 중 하나로, 메모리 관리 효율을 높이기 위해 페이지 크기를 자동으로 크게(2MB, 1GB 단위) 묶어서 관리 하는 기능
	- 이론적으로는 성능 최적화에 도움이 되지만, DB, JVM, 캐시 서버 같은 애플리케이션에는 오히려 성능 저하나 서버 Hang(멈춤) 문제를 일으키는 경우가 많음
	- 특히 MongoDB, Oracle DB, Redis, Kafka, Elasticsearsh 같은 시스템에서 비활성화 권장 설정으로 유명