# 대역

## 대역의 필요성

테스트를 작성하다 보면 외부 요인이 필요한 시점이 있다. 다음은 외부 요인이 테스트에 관여하는 주요 예이다.

- 테스트 대상에서 파일 시스템을 사용
- 테스트 대상에서 DB로부터 데이터를 조회하거나 데이터를 추가
- 테스트 대상에서 외부의 HTTP 서버와 통신

테스트 대상이 이런 외부 요인에 의존하면 테스트를 작성하고 실행하기 어려워진다. 테스트 대상 코드에서 사용하는 외부 API 서버가

일시적으로 장애가 나면 테스트를 원활하게 수행할 수 없다.

6장에서 언급한 자동이체기능을 예로보면 외부 업체가 제공하는 API를 이용해서 카드번호가 유효한지 확인하고 그 결과에 따라 자동이체

정보를 등록한다. 이 기능을 테스트하려면 정상 카드번호, 도난 카드번호, 만료일이 지난 카드번호가 필요하기에 외부 업체에서 

상황별로 테스트할 수 있는 카드번호를 받아와야한다.

![image](https://user-images.githubusercontent.com/40031858/135449381-f0d7bbc4-c8e5-427e-9be0-f21bde4b2ab9.png)

실제 코드로 살펴보자. 자동이체 등록 정보 기능을 구현했고 CardNumberValidator는 외부 API를 이용해서 카드 번호가 유효한지 확인한다.

AutoDebitRegister 클래스는 CardNumberValidator를 이용해서 카드번호가 유효한지 검사한 뒤에 그 결과에 따라 자동이체 정보를 저장한다

```java
public class AutoDebitRegister{
  private CardNumberValidator validator;
  private AutoDebitInfoRepository repository;
  
  public AutoDebitRegister(CardNumberValidator validator,
                          AutoDebitInfoRepository repository){
    this.validator = validator;
    this.repository = repository;
  }
  
  public RegisterResult register(AutoDebitReq req){
    CardValidity validity = validator.validate(req.getCardNumber());
    if(validity != CardValidity.VALID){
      return RegisterResult.error(validity);
    }
    AutoDebitInfo info = repository.findOne(req.getUserId());
    if(info != null){
      info.changeCardNumber(req.getCardNumber());
    }else{
      AutoDebitInfo newInfo = new AutoDebitInfo(req.getUserId(),req.getCardNumber(),
                                               LocalDateTime.now());
      repository.save(newInfo);
    }
    return RegisterResult.success();
  }
}
```

```java
public class CardNumberValidator {
    public CardValidity validate(String cardNumber) {
        HttpClient httpClient = HttpClient.newHttpClient();
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create("https://some-external-pg.com/card"))
                .header("Content-Type", "text/plain")
                .POST(BodyPublishers.ofString(cardNumber))
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
        } catch (IOException | InterruptedException e) {
            return CardValidity.ERROR;
        }
    }
}
```

AutoDebitRegister 클래스를 테스트하는 코드는 다음과 같이 작성할 수 있다.

```java
public class AutoDebitRegisterTest {
    private AutoDebitRegister register;

    @BeforeEach
    void setUp() {
        CardNumberValidator validator = new CardNumberValidator();
        AutoDebitInfoRepository repository = new JpaAutoDebitInfoRepository();
        register = new AutoDebitRegister(validator, repository);
    }

    @Test
    void validCard() {
        // 업체에서 받은 테스트용 유효한 카드 번호 사용
        AutoDebitReq req = new AutoDebitReq("user1", "1234123412341234");
        RegisterResult result = this.register.register(req);
        assertEquals(VALID, result.getValidity());
    }

    @Test
    void theftCard() {
        // 업체에서 받은 도난 테스트용 카드 번호 사용
        AutoDebitReq req = new AutoDebitReq("user1", "1234567890123456");
        RegisterResult result = this.register.register(req);
        assertEquals(THEFT, result.getValidity());
    }
}
```

validCard() 테스트를 통과시키려면 외부 업체에서 테스트 목적의 유효한 카드번호를 받아야 한다. 만약 이 카드번호가 한 달 뒤에 만료되면 validCard()

테스트는 한 달 뒤부터 실패한다. 비슷하게 테스트 용도의 도난 카드 정보를 외부 업체에서 삭제하면 theftCard() 테스트는 통과하지 못하고 실패한다.

물론 외부 서비스가 개발 환경을 제공하지 않으면 테스트는 더욱 힘들어진다.

이렇게 테스트 대상에서 의존하는 요인 때문에 테스트가 어려울 때는 대역을 써서 테스트를 진행할 수 있다. 난이도가 높은 액션이 필요할 때 배우를 

대신해서 연기하는 스턴트맨처럼 테스트에서는 외부 요인으로 인해 테스트가 어려울 때 외부 요인을 대신하는 대역이 외부 요인을 대신해서 테스트에 참여한다.

## 대역을 이용한 테스트

대역을 이용해서 AutoDebitRegister를 테스트하는 코드를 다시 작성해보자. 먼저 CardNumberValidator를 대신할 대역 클래스를 작성하자.

```java
public class StubCardNumberValidator extends CardNumberValidator {
    private String invalidNo;

    public void setInvalidNo(String invalidNo) {
        this.invalidNo = invalidNo;
    }

    @Override
    public CardValidity validate(String cardNumber) {
        if (invalidNo != null && invalidNo.equals(cardNumber)) {
            return CardValidity.INVALID;
        }
        return CardValidity.VALID;
    }
}
```

StubCardNumberValidator는 실제 카드번호 검증 기능을 구현하지 않는다. 대신 단순한 구현으로 실제 구현을 대체한다. validate() 메소드는 invalidNo

필드와 동일한 카드번호면 결과로 INVALID를 리턴하고 그렇지 않으면 VALID를 리턴한다. invalidNo 필드는 setInvalidNo() 메소드로 지정한다. 

즉 StubCardNumberValidator는 setInvalidNo()로 지정한 카드번호에 대해서는 INVALID를 리턴하고 그 외 나머지는 VALID를 리턴한다.

```java
public class AutoDebitRegister_Stub_Test {
    private AutoDebitRegister register;
    private StubCardNumberValidator stubValidator;
    private StubAutoDebitInfoRepository stubRepository;

    @BeforeEach
    void setUp() {
        stubValidator = new StubCardNumberValidator();
        stubRepository = new StubAutoDebitInfoRepository();
        register = new AutoDebitRegister(stubValidator, stubRepository);
    }

    @Test
    void invalidCard() {
        stubValidator.setInvalidNo("111122223333");

        AutoDebitReq req = new AutoDebitReq("user1", "111122223333");
        RegisterResult result = this.register.register(req);

        assertEquals(INVALID, result.getValidity());
    }
```

![image](https://user-images.githubusercontent.com/40031858/135606660-16dddf64-3fcb-455a-9d9b-2f2ca44327c7.png)

추가로 도난 카드에 대한 테스트를 진행하고 싶다고하자. 유효하지 않은 카드번호를 테스트할 때와 동일하게 대역 클래스에 도난 카드번호를 지정하고

이를 비교하는 코드만 추가하면 된다.

```java
public class StubCardNumberValidator extends CardNumberValidator {
    private String invalidNo;
    private String theftNo;

    public void setInvalidNo(String invalidNo) {
        this.invalidNo = invalidNo;
    }

    public void setTheftNo(String theftNo) {
        this.theftNo = theftNo;
    }

    @Override
    public CardValidity validate(String cardNumber) {
        if (invalidNo != null && invalidNo.equals(cardNumber)) {
            return CardValidity.INVALID;
        }
        if (theftNo != null && theftNo.equals(cardNumber)) {
            return CardValidity.THEFT;
        }
        return CardValidity.VALID;
    }
} 
```

도난 카드번호를 처리할 수 있게 대역을 구현했으니 이제 다음과 같이 대역을 이용해서 도난 카드번호에 대한 자동이체 기능을 테스트할 수 잇다.

```java
public class AutoDebitRegister_Stub_Test {
    private AutoDebitRegister register;
    private StubCardNumberValidator stubValidator;
    private StubAutoDebitInfoRepository stubRepository;

 		...

    @Test
    void theftCard() {
        stubValidator.setTheftNo("1234567890123456");

        AutoDebitReq req = new AutoDebitReq("user1", "1234567890123456");
        RegisterResult result = this.register.register(req);

        assertEquals(CardValidity.THEFT, result.getValidity());
    }
```

DB연동 코드도 대역을 사용하기가 적합하다. 예를 들어 자동이체 정보의 DB연동을 처리하는 리포지토리 인터페이스가 다음과 같다고 하자

```java
public interface AutoDebitInfoRepository{
  void save(AutoDebitInfo info);
  AutoDebitInfo findOne(String userId);
}
```

아래의 코드를 보면 AutoDebitRegister는 AutoDebitInfoRepository를 사용해서 자동이체 정보를 저장했다.

```java
public class AutoDebitRegister {
    private CardNumberValidator validator;
    private AutoDebitInfoRepository repository;

   ...

    public RegisterResult register(AutoDebitReq req) {
        CardValidity validity = validator.validate(req.getCardNumber());
        if (validity != CardValidity.VALID) {
            return RegisterResult.error(validity);
        }
        AutoDebitInfo info = repository.findOne(req.getUserId());
        if (info != null) {
            info.changeCardNumber(req.getCardNumber());
        } else {
            AutoDebitInfo newInfo = new AutoDebitInfo(req.getUserId(), req.getCardNumber(), LocalDateTime.now());
            repository.save(newInfo);
        }
        return RegisterResult.success();
    }
}
```

대역을 사용하면 DB없이 AutoDebitRegister를 테스트할 수 있다. AutoDebitInfoReposi-tory의 대역 구현을 아래와 같이 구현해보자.

```java
public class MemoryAutoDebitInfoRepository implements AutoDebitInfoRepository {
    private Map<String, AutoDebitInfo> infos = new HashMap<>();
    @Override
    public void save(AutoDebitInfo info) {
        infos.put(info.getUserId(), info);
    }
    @Override
    public AutoDebitInfo findOne(String userId) {
        return infos.get(userId);
    }
}
```

위처럼 DB대신 맵을 이용해서 자동이체 정보를 저장한다. 메모리에만 데이터가 저장되므로 DB와 같은 영속성을 제공하지는 않지만, 테스트에 사용할 수 

있을 만큼의 기능은 제공한다. 이제 이를 이용해서 테스트코드를 작성해보자

```java
public class AutoDebitRegister_Fake_Test {
    private AutoDebitRegister register;
    private StubCardNumberValidator cardNumberValidator;
    private MemoryAutoDebitInfoRepository repository;

    @BeforeEach
    void setUp() {
        cardNumberValidator = new StubCardNumberValidator();
        repository = new MemoryAutoDebitInfoRepository();
        register = new AutoDebitRegister(cardNumberValidator, repository);
    }

    @Test
    void alreadyRegistered_InfoUpdated() {
        repository.save(
                new AutoDebitInfo("user1", "111222333444", LocalDateTime.now()));

        AutoDebitReq req = new AutoDebitReq("user1", "123456789012");
        RegisterResult result = this.register.register(req);

        AutoDebitInfo saved = repository.findOne("user1");
        assertEquals("123456789012", saved.getCardNumber());
    }

    @Test
    void notYetRegistered_newInfoRegistered() {
        AutoDebitReq req = new AutoDebitReq("user1", "1234123412341234");
        RegisterResult result = this.register.register(req);

        AutoDebitInfo saved = repository.findOne("user1");
        assertEquals("1234123412341234", saved.getCardNumber());
    }
}
```



## 대역의 종류

구현에 따라 다음 표와 같이 대역을 구분할 수 있다

| 대역 종류   | 설명                                                         |
| ----------- | ------------------------------------------------------------ |
| 스텁(Stub)  | 구현을 단순한 것으로 대체한다. 테스트에 맞게 단순히 원하는 동작을<br />수행한다. StubCardNumberValidator가 스텁 대역에 해당한다 |
| 가짜(Fake)  | 제품에는 적합하지 않지만, 실제 동작하는 구현을 제공한다. DB대신에<br />메모리를 이용해서 구현한 MemoryAutoDebitInfoRepository가 가짜<br />대역에 해당한다 |
| 스파이(Spy) | 호출된 내역을 기록한다. 기록한 내용은 테스트 결과를 검증할 때 사용한다. <br />스텁이기도 하다. |
| 모의(Mock)  | 기대한 대로 상호작용하는지 행위를 검증한다. 기대한 대로 동작하지 않으면<br />익셉션을 발생할 수 있다. 모의 객체는 스텁이자 스파이도 된다 |

예를 이용해서 대역을 살펴보자. 사용할 예는 회원 가입 기능이다. 회원 가입 기능을 구현할 UserRegister 및 관련타입이 아래와 같다고하자.

![image](https://user-images.githubusercontent.com/40031858/135608395-b30397fa-69ac-4e41-b0f3-2034384bf353.png)

각 타입은 다음 역할을 수행한다

- UserRegister: 회원 가입에 대한 핵심로직을 수행한다
- WeakPasswordChecker: 암호가 약한지 검사한다
- UserRepository: 회원 정보를 저장하고 조회하는 기능을 제공한다
- EmailNotifier: 이메일 발송 기능을 제공한다

UserRegister를 위한 테스트를 만들어 나가는 과정에서 나머지 타입을 위한 대역으로 스텁, 가짜 , 스파이, 모의 객체를 차례대로 사용해보자.

### 약한 암호 확인 기능에 스텁 사용

암호가 약한 경우 회원 가입에 실패하는 테스트부터 시작하자. 암호가 약한지를 UserRegister가 직접 구현하지 않고 WeakPasswordChecker를

사용하게 하자. 이렇게 하면 각 타입의 역할을 적절하게 분리할 수 있다.

테스트 대상이 UserRegister이므로 WeakPasswordChecker는 대역을 사용할 것이다. 실제 동작하는 구현은 필요하지 않다. 약한 암호인지 

여부를 알려주기만 하면 되므로 스텁 대역이면 충분하다.

```java
public class UserRegisterTest {
    private UserRegister userRegister;
    private StubWeakPasswordChecker stubPasswordChecker = new StubWeakPasswordChecker();

    @BeforeEach
    void setUp() {
        userRegister = new UserRegister(stubPasswordChecker);
    }

    @DisplayName("약한 암호면 가입 실패")
    @Test
    void weakPassword() {
        stubPasswordChecker.setWeak(true);

        assertThrows(WeakPasswordException.class, () -> {
            userRegister.register("id", "pw", "email");
        });
    }
```

암호 확인 요청이 오면 암호가 약하다고 응답하라고 설정한다. 아직 WeaKPasswordChecker 인터페이스가 제공할 메소드를 정하지 않았지만 이를

대신할 스텁은 setWeak() 메소드로 true를 전달받으면 암호가 약하다고 응답하게 구현할 것이다.

아직 어떤 코드도 존재하지 않으므로 테스트 코드는 컴파일에 실패한다. 먼저 WeakPasswordException 타입을 구현한다

```java
public class WeakPasswordException extends RuntimeException {}
```

다음은 StubWeakPasswordChecker를 구현하자. 이 타입은 스텁이므로 상위 타입인 WeakPasswordChecker인터페이스를 만들고 이 인터페이스를

상속해서 스텁을 추가한다. WeakPasswordChecker 타입에 어떤 메소드를 추가할지는 아직 미정이므로 다음과 같이 메소드 없이 인터페이스만

작성한다.

```java
public interface WeakPasswordChecker{}
```

StubWeakPasswordChecker 클래스는 setWeak()메소드가 필요하므로 이 메소드까지 구현한 코드를 작성한다

```java
public class StubWeakPasswordChecker implements WeakPasswordChecker {
  private boolean weak;
  
  public void setWeak(boolean weak){
    this.weak=weak;
  }
}
```

또한 UserRegister클래스를 만들어서 컴파일 에러를 마저 없애자.

```java
public class UserRegister{
  private WeakPasswordChecker passwordChecker;
  
  public UserReigster(WeakPasswordChecker passwordChecker){
    this.passwordChecker = passwordChecker;
  }
  
  public void register(String id, String pw, String email){
    //구현 전
  }
}
```

현재 테스트를 실행하면 실패할 것인데 이 테스트를 통과시키는 방법은 UserRegister#register() 메소드가 바로 익셉션을 발생시키도록 구현하는것이다.

```java
public class UserRegister{
  private WeakPasswordChecker passwordChecker;
  
  public UserRegister(WeakPasswordChecker passwordChecker){
    this.passwordChecker = passwordChecker;
  }
  
  public void register(String id, String pw, String email){
    if(passwordChecker.checkPasswordWeak(pw)){
      throw new WeakPasswordException();
    }
  }
}
```

여기서 WeakPasswordChecker#checkPasswordWeak() 메소드를 이용해서 암호가 약한지 검사하고 있다. 여기서 checkPasswordWeak() 메소드는

UserRegister 입장에서 필요한 코드를 작성한 것이다. 아직 WeakPasswordChecker에는 이 메소드가 존재하지 않는다. 존재하지 않는 메소드를 

사용했기에 컴파일 에러가 발생하니 컴파일 에러를 없애보자.

```java
public interface WeakPasswordChecker{
  boolean checkPasswordWeak(String pw);
}
```

WeakPasswordChecker 인터페이스에 메소드를 추가했으므로 이를 구현한 대역 클래스인 StubWeakPasswordChecker 소스 파일에서 컴파일

에러가 나니 이 또한 구현을 알맞게 추가핮.

```java
public class StubWeakPasswordChecker implements WeakPasswordChecker{
  private boolean weak;
  
  public void setWeak(boolean weak){
    this.weak=weak;
  }
  
  @Override
  public boolean checkPsswordWeak(String pw){
    return weak;
  }
}
```

StubWeakpasswordChecker의 checkPasswordWeak() 메소드는 단순히 weak 필드 값을 리턴한다. 이 정도만 구현해도 UserRegister가 

약한 암호인 경우와 그렇지 않은 경우에 대해 올바르게 동작하는지 확인 할 수 있다.

단순히 익셉션을 발생하는 코드에서 암호 검사 뒤에 익셉션을 발생하도록 구현을 변경햇으니 다시 테스트를 실행해서 통과되는 것을 확인한다.

### 리포지토리를 가짜 구현으로 사용

다음 테스트로 동일 ID를 가진 회원이 존재할 경우 익셉션을 발생하는 테스트를 작성하자.

```java
public class AutoDebitRegister_Fake_Test {
    private AutoDebitRegister register;
    private StubCardNumberValidator cardNumberValidator;
    private MemoryAutoDebitInfoRepository repository;

    @BeforeEach
    void setUp() {
        cardNumberValidator = new StubCardNumberValidator();
        repository = new MemoryAutoDebitInfoRepository();
        register = new AutoDebitRegister(cardNumberValidator, repository);
    }

    @Test
    void alreadyRegistered_InfoUpdated() {
        repository.save(
                new AutoDebitInfo("user1", "111222333444", LocalDateTime.now()));

        AutoDebitReq req = new AutoDebitReq("user1", "123456789012");
        RegisterResult result = this.register.register(req);

        AutoDebitInfo saved = repository.findOne("user1");
        assertEquals("123456789012", saved.getCardNumber());
    }

    @Test
    void notYetRegistered_newInfoRegistered() {
        AutoDebitReq req = new AutoDebitReq("user1", "1234123412341234");
        RegisterResult result = this.register.register(req);

        AutoDebitInfo saved = repository.findOne("user1");
        assertEquals("1234123412341234", saved.getCardNumber());
    }
}
```

UserRepository도 실제와 동일하게 동작하는 가짜 대역을 이용해서 이미 같은 ID를 가진 사용자가 존재하는 상황을 만들면 된다. 이를 염두에 두고

테스트 코드를 추가하자. 추가한 코드는 다음과 같다.

```java
public class UserRegisterTest {
    private UserRegister userRegister;
    private StubWeakPasswordChecker stubPasswordChecker = new StubWeakPasswordChecker();
    private MemoryUserRepository fakeRepository = new MemoryUserRepository();

    @BeforeEach
    void setUp() {
        userRegister = new UserRegister(stubPasswordChecker,
                fakeRepository);
    }
  
   @DisplayName("이미 같은 ID가 존재하면 가입 실패")
    @Test
    void dupIdExists() {
        // 이미 같은 ID 존재하는 상황 만들기
        fakeRepository.save(new User("id", "pw1", "email@email.com"));

        assertThrows(DupIdException.class, () -> {
            userRegister.register("id", "pw2", "email");
        });
    }
                                        
                                                       
```

이제 순서대로 컴파일 에러를 없애보자. 먼저 UserRepository인터페이스와 MemoryUserRepository 클래스를 각각 정의하자

```java
public interface UserRepository{}

public class MemoryUserRepository implements UserRepository{}
```

다음으로 UserRegister 생성자의 인자로 fakeRepository를 전달하므로 생성자에 파라미터를 추가한다. 추가한 코드는 다음과 같다.

```java
public class UserRegister{
  private WeakPasswordChecker passwordChecker;
  private UserRepository userRepository;
  
  public UserRegister(WeakPasswordChecker passwordChecker,
                     UserRepository userRepository){
    this.passwordChecker = passwordChecker;
    this.userRepository = userRepository;
  }
}
```

```java
public class User {
    private String id;
    private String password;
    private String email;

    public User(String id, String password, String email) {
        this.id = id;
        this.password = password;
        this.email = email;
    }

    public String getId() {
        return id;
    }

    public String getEmail() {
        return email;
    }
}
```

레포지토리에 save(User) 메소드를 추가하자. 

```java
public interface UserRepository {
    void save(User user);
}
```

```java
public class MemoryUserRepository implements UserRepository {
    private Map<String, User> users = new HashMap<>();
    @Override
    public void save(User user) {
        users.put(user.getId(), user);
    }
}
```

이제 이러면 테스트가 통과될 것이다. 테스트를 통과시킨 뒤에는 구현을 일반화하자. 이미 존재하는 ID를 가진 사용자가 존재하는지 확인하는 것이다.

```java
public class UserRegister {
   ...

    public void register(String id, String pw, String email) {
        if (passwordChecker.checkPasswordWeak(pw)) {
            throw new WeakPasswordException();
        }
        User user = userRepository.findById(id);
        if (user != null) {
            throw new DupIdException();
        }
    }
}
```

아직 UserRepository 인터페이스는 findById() 메소드가 없으므로 이 메소드를 UserRepository 인터페이스에 추가하고 대역클래스에서 구현하자.

```java
public interface UserRepository {
    void save(User user);

    User findById(String id);
}
```

```java
public class MemoryUserRepository implements UserRepository {
    private Map<String, User> users = new HashMap<>();

    @Override
    public void save(User user) {
        users.put(user.getId(), user);
    }

    @Override
    public User findById(String id) {
        return users.get(id);
    }
}
```

구현을 일반화했으니 다시 테스트를 실행하자. 테스트가 깨지지 않고 통과할 것이다. 중복 아이디가 존재하지 않을 경우 회원가입에 성공하는 경우도

테스트하자. 이 테스트도 가짜 대역을 사용한다. 

```java
    @DisplayName("같은 ID가 없으면 가입 성공함")
    @Test
    void noDupId_RegisterSuccess() {
        userRegister.register("id", "pw", "email");

        User savedUser = fakeRepository.findById("id");
        assertEquals("id", savedUser.getId());
        assertEquals("email", savedUser.getEmail());
    }
```

회원 가입에 성공했는지 여부를 확인하려면 리포지토리에 새로운 사용자 정보가 올바르게 저장되었는지 확인하면 된다. 이 테스트를 실행하면 

NullPointerException이 발생하면서 실패한다. 이는 대역을 이용해서 구한 User가 null임을 의미한다. 이 테스트를 통과시키는 가장 간단한 방법은

당므 코드처럼 상수를 이용해서 생성한 User객체를 리포지토리에 저장하는 것이다.

```java
public class UserRegister {
    private WeakPasswordChecker passwordChecker;
    private UserRepository userRepository;
    private EmailNotifier emailNotifier;

    public UserRegister(WeakPasswordChecker passwordChecker,
                        UserRepository userRepository,
                        EmailNotifier emailNotifier) {
        this.passwordChecker = passwordChecker;
        this.userRepository = userRepository;
        this.emailNotifier = emailNotifier;
    }

    public void register(String id, String pw, String email) {
        if (passwordChecker.checkPasswordWeak(pw)) {
            throw new WeakPasswordException();
        }
        User user = userRepository.findById(id);
        if (user != null) {
            throw new DupIdException();
        }
        userRepository.save(new User("id", "pw", "email"));
    }
}
```

다시 테스트를 실행하면 통과한다. 이제 구현을 일반화하자 상수 대신 파라미터로 전달 받은 값을 사용해서 User객체를 생성하면 된다.

```java
public class UserRegister {
    private WeakPasswordChecker passwordChecker;
    private UserRepository userRepository;
    private EmailNotifier emailNotifier;

    public UserRegister(WeakPasswordChecker passwordChecker,
                        UserRepository userRepository,
                        EmailNotifier emailNotifier) {
        this.passwordChecker = passwordChecker;
        this.userRepository = userRepository;
        this.emailNotifier = emailNotifier;
    }

    public void register(String id, String pw, String email) {
        if (passwordChecker.checkPasswordWeak(pw)) {
            throw new WeakPasswordException();
        }
        User user = userRepository.findById(id);
        if (user != null) {
            throw new DupIdException();
        }
        userRepository.save(new User(id, pw, email));
    }
}
```

### 이메일 발송 여부를 확인하기 위해 스파이를 사용

회원 가입에 성공하면 이메일로 회원 가입 안내 메일을 발송한다고 하자. 이를 검증하기 위한 골격은 다음과 같다

```java
//실행
userRegister.register("id","pw","eamil@somedomian.com");

//결과
eamil@somedomain.com으로 이메일 발송을 요청했는지 확인
```

이메일 발송여부를 어떻게 확인할 수 있을까? 방법 중 하나는 UserRegister가 EamilNotifier의 메일 발송 기능을 실행할 때 이메일 주소로 

"eamil@somedomain.com" 을 사용했는지 확인하는 것이다. 이런 용도로 사용할 수 있는 것이 스파이 대역이다. EmailNotifier의 스파이 대역을

이용한 테스트 코드를 만들어보자. 회원 가입 시 이메일을 올바르게 발송했는지 확인할 수 있으려면 EamilNotifier의 스파이 대역이 이메일 발송 여부와 

발송을 요청할 때 사용한 이메일 주소를 제공할 수 있어야 한다.

```java
public class SpyEmailNotifier  {
    private boolean called;
    private String email;

    public boolean isCalled() {
        return called;
    }

    public String getEmail() {
        return email;
    }   
}
```

EmailNotifier 인터페이스는 다음과 같이 작성한다. 아직 메소드를 정의핮 ㅣ않았다

```java
public interface EmailNotifier {}
```

이제 스파이 대역을 이용해서 메일 발송 여부를 확인하는 테스트를 작성하자. 작성한 코드는 다음과 같다.

```java

public class UserRegisterTest {
    private UserRegister userRegister;
    private StubWeakPasswordChecker stubPasswordChecker = new StubWeakPasswordChecker();
    private MemoryUserRepository fakeRepository = new MemoryUserRepository();
    private SpyEmailNotifier spyEmailNotifier = new SpyEmailNotifier();

    @BeforeEach
    void setUp() {
        userRegister = new UserRegister(stubPasswordChecker,
                fakeRepository,
                spyEmailNotifier);
    }
...

  @DisplayName("가입하면 메일을 전송함")
    @Test
    void whenRegisterThenSendMail() {
        userRegister.register("id", "pw", "email@email.com");

        assertTrue(spyEmailNotifier.isCalled());
        assertEquals(
                "email@email.com",
                spyEmailNotifier.getEmail());
    }
```

이대로 테스트를 실행하면 다음 라인에서 테스트에 실패한다

```java
assertTrue(spyEmailNotifier.isCalled());
```

이 단언을 통과하려면 다음 두 가지를 해야 한다.

- UserRegister가 EmailNotifier의 이메일 발송 기능을 호출
- 스파이의 이메일 발송 기능 구현에서 호출 여부 기록

먼저 UserRegister가 EmailNotifier의 이메일 발송 기능을 호출하는 코드를 추가하자.

```java
public class UserRegister {
    private WeakPasswordChecker passwordChecker;
    private UserRepository userRepository;
    private EmailNotifier emailNotifier;

    ...

    public void register(String id, String pw, String email) {
        if (passwordChecker.checkPasswordWeak(pw)) {
            throw new WeakPasswordException();
        }
        User user = userRepository.findById(id);
        if (user != null) {
            throw new DupIdException();
        }
        userRepository.save(new User(id, pw, email));

        emailNotifier.sendRegisterEmail(email);
    }
}

```

아직 EmailNotifier 인터페이스에 sendRegisterEmail() 메소드가 존재하지 않으므로 컴파일 에러가 발생한다. EmailNotifier 인터페이스 메소드를 추가

해서 컴파일 에러를 없앤다.

```java
public interface EmailNotifier {
    void sendRegisterEmail(String email);
}
```

그리고 SpyEmailNotifier에서 이를 구현하자.

```java
public class SpyEmailNotifier implements EmailNotifier {
    private boolean called;
    private String email;

    public boolean isCalled() {
        return called;
    }

    public String getEmail() {
        return email;
    }

    @Override
    public void sendRegisterEmail(String email) {
        this.called = true;
    }
}
```

필요한 코드를 작성했으니 테스트를 실행하자. 이번에도 테스트를 실패하는데 에러가 발생하는 위치는 아래와 같다.

```java
 assertEquals(
                    "email@email.com",
                    spyEmailNotifier.getEmail());
```

따라서 이 테스트 통과를 위해 다음 코드를 추가하자

```java
public class SpyEmailNotifier implements EmailNotifier {
    private boolean called;
    private String email;

    public boolean isCalled() {
        return called;
    }

    public String getEmail() {
        return email;
    }

    @Override
    public void sendRegisterEmail(String email) {
        this.called = true;
      	this.email = email;
    }
}
```

### 모의 객체로 스텁과 스파이 대체

모의 객체를 위한 몇가지 도구가 존재하는데 이 레포지토리에서는 `Mockito` 를 사용한다. 

먼저 약한 암호인 경우 가입에 실패하는 테스트를 모의 객체를 이용해서 작성해보자. 테스트 코드는 아래와 같다.

```java
public class UserRegisterMockTest {
    private UserRegister userRegister;
    private WeakPasswordChecker mockPasswordChecker = Mockito.mock(WeakPasswordChecker.class);
    private MemoryUserRepository fakeRepository = new MemoryUserRepository();
    private EmailNotifier mockEmailNotifier = Mockito.mock(EmailNotifier.class);

    @BeforeEach
    void setUp() {
        userRegister = new UserRegister(mockPasswordChecker,
                fakeRepository,
                mockEmailNotifier);
    }

    @DisplayName("약한 암호면 가입 실패")
    @Test
    void weakPassword() {
        BDDMockito.given(mockPasswordChecker.checkPasswordWeak("pw")).willReturn(true);

        assertThrows(WeakPasswordException.class, () -> {
            userRegister.register("id", "pw", "email");
        });
    }
```



```markdown
BDDMockito.given(mockPasswordChecker.checkPasswordWeak("pw")).willReturn(true);

이 코드는 모의객체가 다음과 같이 행동하도록 설정한다

BDDMockito

// "pw" 인자를 사용해서 모의 객체의 checkPasswordWeak 메소드를 호출하면

.given(mockPasswordChecker.checkPasswordWeak("pw"))

//결과로 true를 리턴하라

.willReturn(true);
```

대역 객체가 기대하는 대로 상호작용 했는지 확인하는 것이 모의 객체의 주요 기능이다. Mockito를 사용하면 아래와 같이 모의 객체가 기대한 대로

불렸는지 검증할 수 있다.

```java
@DisplayName("회원 가입시 암호 검사 수행함")
@Test
void checkPassword(){
  userRegister.register("id","pw","email");
  
  BDDMockito.then(mockPasswordChecker)
    				.should()
    				.checkPasswordWeak(BDDMockito.anyString());
}
```

모의 객체를 사용하면 스파이도 가능하다. 아래의 코드는 모의 객체의 sendRegisterEmail() 메소드를 호출할 때 사용한 인자 값을 구하는 코드 예를 보여준다.

```java
@DisplayName("가입하면 메일을 전송함")
    @Test
    void whenRegisterThenSendMail() {
        userRegister.register("id", "pw", "email@email.com");

        ArgumentCaptor<String> captor = ArgumentCaptor.forClass(String.class);
        BDDMockito.then(mockEmailNotifier).should().sendRegisterEmail(captor.capture());

        String realEmail = captor.getValue();
        assertEquals("email@email.com", realEmail);
    }
```

Mockito의 ArgumentCaptor는 모의 객체를 메소드를 호출할 때 전달한 객체를 담는 기능을 제공한다. BDDMockito.then().should()로 모의 객체의

메소드가 호출됐는지 확인할 때 ArgumentCaptor#capture() 메소드를 사용하면 메소드를 호출할 때 전달한 인자가 ArgumentCaptor에 담긴다.

ArgumentCaptor#getValue() 메소드를 사용해서 보관한 인자를 구할 수 있다.

# 대역과 개발 속도

TDD과정에서 대역을 사용하지 않고 실제 구현을 사용한다면 다음과 같은 일이 벌어지게 된다

- 카드 정보 제공 업체에서 도난 카드번호를 받을 때까지 테스트를 기다린다
- 카드 정보 제공 API가 비정상 응답을 주는 상황을 테스트하기 위해 업체의 변경 대응을 기다린다
- 회원 가입 테스트를 한 뒤에 편지가 도착할 때까지 메일함을 확인한다
- 약한 암호 검사 기능을 개발할 때까지 회원 가입 테스트를 대기한다

네 경우 모두 대기 시간이 발생한다. 도난 카드에 대한 테스트를 진행하기 위해 업체로부터 도난 카드번호를 받아야 한다. 바로 카드번호를

받을 수 있으면 좋지만 1~2일 이상 소요되는 경우도 있다.

회원 가입할 때 메일이 발송되는지 확인하려면 실제 이메일 주소를 이용해서 테스트를 진행해야 한다. 또한, 메일이 도착할 때까지 메일함을

확인해야 한다. 이메일 특성상 테스트를 실행하고 몇 분 뒤에 메일이 도착하기도 한다.

약한 암호 검사 기능을 다른 개발자가 구현하고 있다면 그 개발자가 구현을 완료할 때까지 약한 암호에 대한 회원 가입 테스트를 진행할 수

없다. 대역을 사용하면 실제 구현이 없어도 다양한 상황에 대해 테스트할 수 있다. 외부의 카드 정보 제공 API와 연동할 수 없는 경우에도

유효한 카드번호나 도난 카드번호에 대한 테스트를 진행할 수 있다. DB가 없어도 동일 ID가 이미 존재하는 상황을 테스트할 수 있다.

또한, 대역을 사용하면 실제 구현이 없어도 실행 결과를 확인할 수 있다. DB가 없어도 회원 데이터가 올바르게 저장되는지 확인할 수 있고

메일 서버가 없어도 이메일 발송 요청을 하는지 확인할 수 있다.

즉, 대역은 의존하는 대상을 구현하지 않아도 테스트 대상을 완성할 수 있게 만들어주며 이는 대기시간을 줄여주어 개발 속도를 올리는데

도움이 된다.

# 모의 객체를 과하게 사용하지 않기

모의 객체는 스텁과 스파이를 지원하므로 대역으로 모의객체를 많이 사용한다. 하지만 모의 객체를 과하게 사용하면 오히려 테스트 코드가 

복잡해지는 경우도 발생한다. 회원 가입 성공 테스트를 모의 객체를 이용해 작성해보자

```java
public class UserRegisterMockOvercaseTest {
  private UserRegister userRegister;
  private UserRepository mockRepository = Mockito.mock(UserRepository.class);
  ... 생략
    
  @BeforeEach
  void setUp(){
    userRegister = new UserRegister(mockPasswordChecker , mockRepository , mockEmailNotifier);
  }
  
  @Test
  void noDupId_RegisterSuccess(){
    userRegister.register("id","pw","email");
    ArgumentCaptor<User> captor = ArgumentCaptor.forClass(User.class);
    BDDMockito.then(mockRepository).should().save(captor.capture());
    
    User savedUser = captor.getValue();
    assertEquals("id",savedUser.getId());
    assertEquals("eamil",savedUser.getEmail());
  }
}
```

성공 여부를 확인하기 위해 mockRepository의 save() 메소드가 호출되었는지 여부를 검증하고 ArgumentCaptor를 이용해서 save() 메소드를 호출

할 때 전달한 인자를 저장했다. 이렇게 저장한 객체를 이용해서 값이 유효한지 검증한다. 

이 코드는 겨로가적으로 회원 데이터가 올바르게 저장되었는지 검증하므로 원하는 결과를 확인한다. 하지만 결과를 확인하기 위해 테스트 코드가 복잡

해졌다. save() 메소드가 호출되었는지 확인해야 하고 ArgumentCaptor를 이용해서 호출할 때 전달한 인자를 저장해야 한다.

메모리를 이용한 가짜 구현을 사용하면 다음과 같이 결과 확인 코드가 단순해진다.

```java
@Test
void 같은_ID가_없으면_가입(){
  userRegister.register("id","pw","email");
  
  User savedUser = fakeRepository.findById("id");
  assertEquals("id",savedUser.getId());
  assertEquals("email",savedUser.getEmail());
}
```

결과를 확인하는 코드가 단순해질 뿐만 아니라 테스트 코드의 의미도 더 명확하다. 모의 객체를 사용했을 때는 "리포지토리의 save() 메소드를 호출해야 하고

이때 전달한 객체의 값이 어때야 한다" 는 식으로 결과를 검증했다면 가짜 구현을 사용하면 "리포지토리에 저장된 객체의 값이 어때야 한다" 는 식으로

검증할 내용에 더 가까워진다. 모의 객체를 이용하면 대역 클래스를 만들지 않아도 되니까 처음에는 편할 수 있다. 하지만 결과값을 확인하는 수단으로 모의 

객체를 사용하기 시작하면 결과 검증 코드가 길어지고 복잡해진다. 특히 하나의 테스트를 위해 여러 모의 객체를 사용하기 시작하면 결과 검증 코드의 

복잡도는 배로 증가한다. 게다가 모의 객체는 기본적으로 메소드 호출 여부를 검증하는 수단이기 때문에 테스트 대상과 모의 객체 간의 상호 작용이

조금만 바뀌어도 테스트가 깨지기 쉽다. 

이런 이유로 모의 객체의 메소드 호출 여부를 결과 검증 수단으로 사용하는 것은 주의해야 한다. 특히 DAO나 리포지토리와 같이 저장소에 대한 

대역은 모의 객체를 사용하는 것보다 메모리를 이용한 가짜 구현을 사용하는 것이 테스트 코드 관리에 유리하다. 물론 처음에는 가짜 대역을 구현

해야하니 귀찮을 수 있는데 일단 가짜 대역을 구현하면 모의 객체를 사용할 때 보다 테스트 코드가 간결해지고 관리하기 쉬워진다.
