# TBNWS Admin Back

TBNWS Admin 백엔드 서버 애플리케이션입니다.

## Tech Stack

- **Java 17**
- **Spring Boot 3.4.8**
- **MyBatis** - ORM
- **MySQL** - Database
- **Redis** - 캐시 및 세션 관리
- **Elasticsearch** - 검색 엔진
- **Spring Security + OAuth2** - 인증/인가
- **Spring AI (OpenAI)** - AI 기능
- **AWS Bedrock** - AI 서비스
- **Prometheus + Micrometer** - 모니터링

## Project Structure

```
src/main/java/com/tbnws/gtgear/support/tbnws_admin_back/
├── config/       # 설정 클래스
├── controller/   # API 컨트롤러
├── service/      # 비즈니스 로직
├── dao/          # 데이터 접근 계층
├── dto/          # 데이터 전송 객체
├── model/        # 도메인 모델
├── enums/        # Enum 정의
├── builder/      # 빌더 패턴
└── util/         # 유틸리티 클래스
```

## Build & Run

```bash
# 빌드
./gradlew build

# 실행
./gradlew bootRun

# 테스트
./gradlew test
```

## External Integrations

- **POPBiLL** - 전자 세금계산서 API
- **Zendesk** - 고객 지원
- **Google API** - Gmail, Calendar
- **ShedLock** - 분산 스케줄러 락
