## 아키텍처 상세 구성

### 1. 전체 구조
EC2 한 대에 Nginx, Spring Boot, MySQL을 함께 올렸습니다. React 빌드 결과는 Spring Boot JAR 안의 `static/`에 포함해서 **배포 산출물이 JAR 하나**가 되도록 했습니다.

프론트와 API를 같은 도메인(same-origin)에서 내보내기 때문에 CORS 설정이 필요 없고, 세션 쿠키도 그대로 동작합니다.

Browser → Nginx(:443) → Spring Boot(:8080) → MySQL(:3306)

### 2. Nginx — 앞단 게이트웨이
- **TLS termination**: Let's Encrypt 인증서로 HTTPS를 처리하고, 80번으로 들어온 요청은 443으로 리다이렉트합니다.
- **Reverse proxy**: 모든 요청을 `127.0.0.1:8080`으로 넘깁니다. 8080 포트는 외부에 열지 않았습니다.
- **Rate limiting** (IP당 `limit_req`, 초과하면 429):
  - 로그인·회원가입: 분당 5회 (burst 5) — 무차별 대입과 계정 양산 완화
  - 점수 저장: 분당 20회 (burst 10) — 랭킹 도배 완화
  - 랭킹 조회는 제한 없음 (읽기 전용이라 캐시로 대응)
- **gzip**: JS·CSS·JSON 응답 압축
- **정적 파일 캐시**: 파일 이름에 해시가 붙는 `/assets/`는 1년 `immutable` 캐시, `index.html`은 `no-store`로 두어 배포하면 바로 새 버전을 받게 했습니다.

### 3. Spring Boot — 애플리케이션
- **계층 구조**: Controller → Service → Repository (Spring Data JPA)
- **인증**: Spring Security 세션 기반 인증, 비밀번호는 BCrypt로 해시
  - 공개 API: `POST /api/auth/signup`, `POST /api/auth/login`, `GET /api/scores/top`
  - 그 외 `/api/**`는 로그인이 필요하고, 인증 실패 시 401 JSON 응답
- **랭킹 캐시**: Spring Cache + Caffeine
  - `getTop()`에 `@Cacheable` (최대 100개 항목, 60초 후 만료)
  - 점수가 저장되면 `@CacheEvict`로 캐시를 비워서 새 점수가 바로 랭킹에 반영됩니다.
- **DB 접근**: JPA(Hibernate) → JDBC 드라이버 → MySQL. 로컬은 H2, 운영은 MySQL 8로 프로필을 나눴습니다.

### 4. CI/CD — GitHub Actions
1. push하면 프론트 테스트(Vitest)와 백엔드 테스트·빌드(Gradle)를 실행합니다.
2. `main` 브랜치일 때만 배포합니다.
   - 배포하는 동안만 보안 그룹에 GitHub Runner IP의 SSH(22) 접근을 열고, 끝나면 성공·실패와 관계없이 닫습니다.
   - JAR를 SCP로 올린 뒤 직전 버전은 `app.jar.bak`으로 백업하고 systemd 서비스를 재시작합니다.
   - `/api/scores/top`이 응답하는지 최대 2분 30초 동안 확인해서 배포 성공 여부를 판단합니다.

### 5. 모니터링 — AWS CloudWatch
- EC2 CPU 80% 이상, 상태 검사 실패 시 경보
- SNS(`tetris-alerts`)로 메일 알림

### 6. 성능 검증 — k6 부하 테스트
랭킹 조회 API(`GET /api/scores/top?limit=10`)를 로컬에서 50 VU로 30초 동안 측정했습니다.

| 지표 | 캐시 적용 전 | 캐시 적용 후 |
| --- | --- | --- |
| 초당 처리 수 | 393 req/s | 472 req/s (+20%) |
| 응답 시간 p95 | 30.6 ms | 8.2 ms (-73%) |
| 오류율 | 0% | 0% |

### 7. 설계 선택과 한계
- **S3 + CloudFront 대신 JAR 단일 배포**: 서버 한 대, 산출물 하나로 배포와 롤백(`.bak`)을 단순하게 유지했습니다.
- **Rate limit을 앱이 아닌 Nginx에서 처리**: 과도한 요청이 Spring까지 오기 전에 차단합니다.
- **Redis 대신 Caffeine**: 서버가 한 대라 로컬 메모리 캐시로 충분했습니다. 서버를 여러 대로 늘리면 캐시와 세션을 Redis로 옮겨야 합니다.
- **현재 한계**: EC2 한 대라 단일 장애점(SPOF)이 있고, DB도 같은 인스턴스에 있습니다. 이후 RDS 분리, ALB + Auto Scaling, Flyway 마이그레이션을 검토하고 있습니다.
