# 테스트 범위와 종류

## 테스트 범위

일반적인 웹 애플리케이션은 다음과 같은 구성 요소를 갖는다.

![image](https://user-images.githubusercontent.com/40031858/135755987-ad40bbc7-0fd1-43c9-b009-01a2b6975c27.png)

개발자가 코드를 조금씩 수정할 때마다 브라우저부터 DB까지의 전 범위를 테스트해야 하는 것은 아니며 출시 전에 DB와 연동할 때

사용한 쿼리만 확인해서도 안된다. 테스트의 범위는 테스트의 목적과 수행하는 사람에 따라 달라진다. 테스트 범위에 따른 테스트 종류는

아래와 같이 세가지로 나눠볼 수 있다.

![image](https://user-images.githubusercontent.com/40031858/135756130-4043c159-92c8-4479-bae9-51bd8d825a80.png)

### 기능 테스트와 E2E테스트

기능테스트는 사용자 입장에서시스템이 제공하는 기능이 올바르게 동작하는지 확인한다. 이 테스트를 수행하려면 시스템을 구동하고 사용하는데

필요한 모든 구성 요소가 필요하다. 예를 들어 회원 가입 기능이 올바르게 작동하는지 확인하려면 웹 서버, 데이터베이스, 웹브라우저가 필요하다.

기능 테스트는 끝에서 끝까지 올바른지 검사하기 때문에 E2E(End to end) 테스트로도 볼 수 있다.

### 통합 테스트

통합 테스트는 시스템의 각 구성 요소가 올바르게 연동되는지 확인한다. 기능 테스트가 사용자 입장에서 테스트하는 데 반해 통합 테스트는 소프트웨어의

코드를 직접 테스트한다. 모바일 앱을 예로 들면 기능 테스트는 앱을 통해 가입 기능을 테스트한다면 통합 테스트는 서버의 회원 가입 코드를 직접

테스트하는 식이다.

### 단위 테스트

단위 테스트는 개별 코드나 컴포넌트가 기대한대로 동작하는지 확인한다. 단위 테스트는 한 클래스나 한 메소드와 같은 작은 범위를 테스트한다. 일부 의존

대상은 스텁이나 모의 객체 등을 이용해서 대역으로 대체한다.

### 테스트 범위 간 차이

각 테스트는 다음과 같은 차이가 있다

- 통합 테스트를 실행하려면 DB나 캐시 서버와 같은 연동 대상을 구성해야 한다. 기능 테스트를 실행하려면 웹 서버를 구동하거나 모바일 앱을 폰에 설치해야 할 수도 있다. 또한, 통합 테스트나 기능 테스트는 테스트
  상황을 만들어내기 위해 많은 노력이 필요하다. 반면에 단위 테스트는 테스트 코드를 빼면 따로 준비할 것이 없다.
- 통합 테스트는 DB연결 , 소켓 통신, 스프링 컨테이너 초기화와 같이 테스트 실행 속도를 느리게 만드는 요인이 많다. 기능 테스트는 추가로 브라우저나 앱을 구동하고 화면의 흐름에 따라 알맞은 상호 작용을
  해야한다. 반면에 단위 테스트는 서버를 구동하거나 DB를 준비할 필요가 없다. 테스트 대상이 의존하는 기능을 대역으로 처리하면 되므로 테스트 실행 속도가 빠르다
- 통합 테스트나 기능 테스트로는 상황을 준비하거나 결과 확인이 어렵거나 불가능할 때가 있다. 외부 시스템과 연동해야 하는 기능이 특히 그렇다. 이런 경우에는 단위 테스트와 대역을 조합해서 상황을 만들고 그 결과를
  확인해야 한다.

## 외부 연동이 필요한 테스트 예

소프트웨어는 다양한 외부 연동이 필요하다. 모든 외부 연동 대상을 통합 테스트에서 다룰 수 없지만, 일부 외부 대상은 어느 정도 수준에서 제어가 가능하다.

### 스프링 부트와 DB 통합 테스트

통합 테스트 예를 살펴보자. 먼저 7장에서 대역을 설명할때 사용한 회원 가입 예를 보자.

일단 스프링에 빈 객체로 등록된 타입은 다음과 같다

- UserRegister
- SimpleWeakPasswordChecker
- UserRepository(스프링 데이터 JPA를 이용해서 등록)
- VirtualEmailNotifier

스프링을 이용해서 각 빈 객체를 생성하고 연결한 뒤 UserRegister를 테스트하는 코드는 다음과 같이 작성할 수 있다.

```java
@SpringBootTest
public class UserRegisterIntTest {
    @Autowired
    private UserRegister register;
    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Test
    void 동일ID가_이미_존재하면_익셉션() {
        // 상황
        jdbcTemplate.update(
                "insert into user values (?,?,?) " +
                "on duplicate key update password = ?, email = ?",
                "cbk", "pw", "cbk@cbk.com", "pw", "cbk@cbk.com");

        // 실행, 결과 확인
        assertThrows(DupIdException.class,
                () -> register.register("cbk", "strongpw", "email@email.com")
        );
    }

    @Test
    void 존재하지_않으면_저장함() {
        // 상황
        jdbcTemplate.update("delete from user where id = ?", "cbk");
        // 실행
        register.register("cbk", "strongpw", "email@email.com");
        // 결과 확인
        SqlRowSet rs = jdbcTemplate.queryForRowSet("select * from user where id = ?", "cbk");
        rs.next();
        assertEquals("email@email.com", rs.getString("email"));
    }
}
```

통합 테스트는 실제로 DB를 사용한다. 동일한 테스트를 여러 번 실행해도 결과가 같게 나와야 하므로 테스트 코드에서 DB 데이터를 알맞게 제어해야 한다.

데이터가 존재하는 상황을 만들기 위해 DB에 데이터를 추가해야 하고 존재하지 않는 상황을 만들기 위해 DB에서 데이터를 삭제해야 한다.

먼저 동일 ID가 존재할 때 가입에 실패하는지 검증하는 테스트 메소드를 보자 .실제 ID가 "cbk" 인 데이터가 이미 존재할 수도 있으므로 

ON DUPLICATE KEY를 사용해서 INSERT쿼리에서 오류가 발생하지 않도록 했다. 동일 ID가 존재하지 않을 때 회원 정보를 올바르게 저장하는지 검증하는

테스트 메소드를 보자. 테스트에서 사용할 ID를 가진 데이터가 실제로 존재할 수도 있기 때문에 DELETE 쿼리를 실행해서 동일 ID를 가진 회원 데이터가

존재하지 않는 상황을 만든다. 기능을 실행한 뒤에는 회원 데이터가 올바르게 저장되었는지 확인하기 위해 SELECT쿼리로 데이터를 읽어와 기대한 값과

같은지 확인한다

```java
public class UserRegisterTest{
  private UserRegister userRegister;
  private MemoryUserRepository fakeRepository = new MemoryUserRepository();
  ...생략
    
    @Test
    void 이미_같은ID_존재하면_가입_실패(){
    	//단위 테스트는 대역을 이용한 상황 구성
    	fakeRepository.save(new User("id","pw1","eamil@eamil.com"));
    
    	assertThrows(DupIdException.class, () -> {
        userRegister.register("id","pw2","email");
      });
  	}
}
```

통합 테스트와 단위 테스트는 실행 시간에도 차이가 있다. 스프링 부트를 이용한 통합 테스트는 테스트 메소드를 실행하기 전에 스프링 컨테이너를 생성하는

과정이 필요하다. 반면에 단위 테스트는 이런 과정이 없으므로 테스트를 실행하는 시간이 매우 짧다.

### WireMock을 이용한 REST 클라이언트 테스트

통합 테스트하기 어려운 대상이 외부 서버이다. 카드사 API를 이용해 카드번호가 유효한지 확인하는 코드를 아래와 같이 표시했다.

```java
public class CardNumberValidator {

    private String server;

    public CardNumberValidator(String server) {
        this.server = server;
    }

    public CardValidity validate(String cardNumber) {
        HttpClient httpClient = HttpClient.newHttpClient();
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(server + "/card"))
                .header("Content-Type", "text/plain")
                .POST(BodyPublishers.ofString(cardNumber))
                .timeout(Duration.ofSeconds(3))
                .build();
        try {
            HttpResponse<String> response = httpClient.send(request, BodyHandlers.ofString());
            switch (response.body()) {
                case "ok": return CardValidity.VALID;
                case "bad": return CardValidity.INVALID;
                case "expired": return CardValidity.EXPIRED;
                case "theft": return CardValidity.THEFT;
                default: return CardValidity.UNKNOWN;
            }
        } catch (HttpTimeoutException e) {
            return CardValidity.TIMEOUT;
        } catch (IOException | InterruptedException e) {
            return CardValidity.ERROR;
        }
    }
}
```

CardNumberValidator 자체를 테스트하려면 정해진 규칙에 맞게 통신할 수 있는 서버가 필요하다. 테스트하려면 외부의 카드 정보 제공 API와 통신해야

하는데 원하는 상황을 쉽게 만들 수 없다. 예를 들어 외부 업체에 테스트할 때마다 타임아웃이 발생하게 처리 시간을 늘려달라고 부탁할 수는 없다.

WireMock을 사용하면 서버 API를 스텁으로 대체할 수 있다. CardNumberValidator의 대역을 사용해서 정상 카드번호와 도난 카드번호에 대해

AutoDebitRegister를 테스트했던 것처럼 WireMock을 사용하면 올바른 응답이나 타임아웃과 같은 상황에 대해 CardNumberVali-dator를 테스트

할 수 있다. 

```java
public class CardNumberValidatorTest {

    private WireMockServer wireMockServer;

    @BeforeEach
    void setUp() {
        wireMockServer = new WireMockServer(options().port(8089));
        wireMockServer.start();
    }

    @AfterEach
    void tearDown() {
        wireMockServer.stop();
    }

    @Test
    void valid() {
        wireMockServer.stubFor(post(urlEqualTo("/card"))
                .withRequestBody(equalTo("1234567890"))
                .willReturn(aResponse()
                        .withHeader("Content-Type", "text/plain")
                        .withBody("ok"))
        );

        CardNumberValidator validator =
                new CardNumberValidator("http://localhost:8089");
        CardValidity validity = validator.validate("1234567890");
        assertEquals(CardValidity.VALID, validity);
    }

    @Test
    void timeout() {
        wireMockServer.stubFor(post(urlEqualTo("/card"))
                .willReturn(aResponse()
                        .withFixedDelay(5000))
        );

        CardNumberValidator validator =
                new CardNumberValidator("http://localhost:8089");
        CardValidity validity = validator.validate("1234567890");
        assertEquals(CardValidity.TIMEOUT, validity);
    }
}
```

WireMockServer는 HTTP 서버를 흉내 낸다. 일반적인 사용법은 다음과 같다

- 테스트 실행 전에 WireMockServer를 시작한다. 실제 HTTP 서버가 뜬다
- 테스트에서 WireMockServer의 동작을 기술한다
- HTTP 연동을 수행하는 테스트를 실행한다
- 테스트 실행 후에 WireMockServer를 중지한다

위의 코드를 보면 WireMockServer가 다음과 같이 동작하도록 기술한다.

- 요청이 다음과 같으면
  - URL이 "/card"
  - POST 요청
  - 요청 몸체가 "1234567890"
- 아래와 같이 응답
  - Content-Type이 text/plain이고
  - 응답 몸체가 "ok"

이 동작은 카드번호가 유효한 상황을 기술한다. CardNumberValidator를 생성할 때 서버로 "http://localhost:8089" 를 지정해서 WireMockServer

가 제공하는 HTTP 서버에 연결한다. 

### 스프링 부트의 내장 서버를 이용한 API 기능 테스트

모바일 앱에서 회원 가입을 위해 사용하는 회원 가입 API가 올바르게 JSON을 응답하는지 검증해야 한다고 하자. 회원 가입은 매우 중요하기 때문에

회원 가입 API를 검증하는 테스트 코드를 작성해서 검증 과정을 자동화하면 담당자가 수동으로 테스트하는 시간을 줄일 수 있다.

스프링 부트를 사용한다면 내장 톰캣을 이용해서 API에 대한 테스트를 JUnit 코드로 작성할 수 있다.

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class UserApiE2ETest {
    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void weakPwResponse() {
        String reqBody = "{\"id\": \"id\", \"pw\": \"123\", \"email\": \"a@a.com\" }";
        RequestEntity<String> request =
                RequestEntity.post(URI.create("/users"))
                        .contentType(MediaType.APPLICATION_JSON_UTF8)
                        .body(reqBody);

        ResponseEntity<String> response = restTemplate.exchange(
                request,
                String.class);

        assertEquals(HttpStatus.BAD_REQUEST, response.getStatusCode());
        assertTrue(response.getBody().contains("WeakPasswordException"));
    }
}
```

스프링 부트는 테스트에서 웹 환경을 구동할 수 있는 기능을 제공한다. 이 테스트는 이 기능을 사용해서 내장 서버를 구동하고 스프링 웹 애플리케이션을

실행한다. TestRestTemplate은 스프링 부트가 테스트 목적으로 제공하는 것으로서 내장 서버에 연결하는 RestTemplate이다. 
