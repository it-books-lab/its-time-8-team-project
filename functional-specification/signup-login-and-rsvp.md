# MVP - Google 로그인 및 RSVP

## 1. **Google 로그인(id_token 검증)**
    - 간단히: 프론트엔드가 받은 `id_token`을 백엔드에서 검증해 사용자 식별.
    - 왜 필요한가: 무마찰 로그인(회원가입/로그인) + 신뢰도 지표(`email_verified`).
    - 핵심 포인트: `iss/aud/exp/sub` 확인, 만료/서명 체크.
    - 첫 실습: 프론트에서 Google 로그인으로 id_token 받아서, 백엔드에서 검증해 사용자 생성/조회.
## 2. **기본 DB + 트랜잭션 이해**
    - 간단히: 이벤트 RSVP(정원 관리) 같은 작업은 동시성 문제가 있음 → 트랜잭션으로 보호.
    - 왜 필요한가: 동시에 여러 명이 RSVP 하면 초과 예약이 발생할 수 있음.
    - 핵심 포인트: `SELECT ... FOR UPDATE`(혹은 JPA의 PESSIMISTIC WRITE), `@Transactional`.
    - 첫 실습: 이벤트 테이블 하나 만들고 동시성 테스트(두 요청 동시에 RSVP) 해보기.
  
## 3. Spring Boot 예제 1 — Google id_token 검증 (간단)

> 의존성: com.google.api-client:google-api-client 계열 사용 (또는 Spring Security OAuth2 활용).
> 
> 
> (라이브러리 이름/버전은 프로젝트에 맞춰 추가하면 됨.)
> 

```java
// AuthController.java
@RestController
@RequestMapping("/api/auth")
public class AuthController {

    private final GoogleIdTokenVerifier verifier; // 초기화는 생성자에서
		
    public AuthController() {
		    // verifier: 나중에 클라이언트가 보낸 Google 로그인 토큰이 진짜 유효한지 검사할 때 쓰는 도구.
        verifier = new GoogleIdTokenVerifier.Builder(new NetHttpTransport(), new JacksonFactory())
                .setAudience(Collections.singletonList("YOUR_GOOGLE_CLIENT_ID"))
                .build();
    }

    @PostMapping("/google")
    public ResponseEntity<?> loginWithGoogle(@RequestBody Map<String, String> body) throws GeneralSecurityException, IOException {
        String idTokenString = body.get("idToken");
        GoogleIdToken idToken = verifier.verify(idTokenString);

        if (idToken == null) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).body("Invalid ID token");
        }
				
				// payload: 구글 토큰 안에 들어있는 사용자 정보(데이터 덩어리).
				// sub → 구글에서 발급한 고유 사용자 ID (숫자/문자 조합, 우리 서비스에서 PK처럼 사용 가능)
				// email → 사용자의 구글 이메일
				// emailVerified → 이메일 인증 여부
				// name → 사용자 이름 (구글 프로필에 등록된 이름)
				// picture → 프로필 사진 URL
        Payload payload = idToken.getPayload();
        String sub = payload.getSubject(); // 고유 구글 사용자 ID
        String email = payload.getEmail();
        boolean emailVerified = Boolean.TRUE.equals(payload.getEmailVerified());
        String name = (String) payload.get("name");
        String picture = (String) payload.get("picture");

        // TODO: user 조회/생성 로직 (DB)
        // 예: userService.findOrCreateByGoogleSub(sub, email, name, picture, emailVerified);

        return ResponseEntity.ok(Map.of("userId", sub, "emailVerified", emailVerified));
    }
}

```

설명: 프론트가 받아온 `idToken`을 이 API로 보내면, 백엔드에서 구글 라이브러리로 검증 후 유저를 생성·로그인 처리하면 됨.

---

# 4. Spring Boot 예제 2 — RSVP 트랜잭션(정원 + 대기열 처리, 의사코드)

> 핵심: 이벤트 행을 트랜잭션 동안 락(PESSIMISTIC_WRITE)으로 잠궈 동시성 제어.
> 

```java
@Service
public class EventService {

    private final EventRepository eventRepository;
    private final RsvpRepository rsvpRepository;
    private final WaitlistRepository waitlistRepository;

    public EventService(EventRepository eventRepository, RsvpRepository rsvpRepository, WaitlistRepository waitlistRepository) {
        this.eventRepository = eventRepository;
        this.rsvpRepository = rsvpRepository;
        this.waitlistRepository = waitlistRepository;
    }

    @Transactional
    public String rsvp(Long eventId, Long userId) {
        // JPA: 이벤트를 배타적 락으로 조회 (Repository에서 @Lock 사용)
        Event event = eventRepository.findByIdForUpdate(eventId)
                .orElseThrow(() -> new IllegalArgumentException("Event not found"));

        long currentCount = rsvpRepository.countByEventId(eventId);
        if (currentCount < event.getCapacity()) {
            // 자리 있음 -> RSVP 저장
            Rsvp r = new Rsvp(eventId, userId, LocalDateTime.now());
            rsvpRepository.save(r);
            return "CONFIRMED";
        } else {
            // 가득 참 -> 대기열 추가
            Waitlist w = new Waitlist(eventId, userId, LocalDateTime.now());
            waitlistRepository.save(w);
            return "WAITLISTED";
        }
    }
}

```

```bash
1) 아직 정원이 남았다면
→ new Rsvp(eventId, userId, 시간) 생성 후 DB 저장 → "CONFIRMED"
2) 정원이 다 찼다면
→ new Waitlist(eventId, userId, 시간) 생성 후 DB 저장 → "WAITLISTED"
```

*주의*: `findByIdForUpdate`는 JPA에서 `@Lock(LockModeType.PESSIMISTIC_WRITE)`로 구현하거나 native query로 `SELECT ... FOR UPDATE`를 사용할 것. 트랜잭션 범위를 너무 넓게 잡지 않도록 주의.


