# 테스트 가능한 설계

## 테스트가 어려운 코드 

모든 테스트를 테스트할 수 있는 것은 아니다. 개발을 진행하다 보면 테스트하기 어려운 코드를 만나게된다.

### 하드 코딩된 경로

결제 대행업체가 결제 내역이 유효한지 확인할 수 있도록 익일 오전에 결제 결과를 파일로 제공한다고하자. 이 파일을 읽어와 DB에 결제 내역을 반영

하는 코드는 다음과 같이 작성할 수 있다.

```java
public class PaySync {
    private PayInfoDao payInfoDao = new PayInfoDao();

    public void sync() throws IOException {
        Path path = Paths.get("/data/pay/cp0001.csv");
        List<PayInfo> payInfos = Files.lines(path)
                .map(line -> {
                    String[] data = line.split(",");
                    PayInfo payInfo = new PayInfo(
                            data[0], data[1], Integer.parseInt(data[1])
                    );
                    return payInfo;
                })
                .collect(Collectors.toList());

        payInfos.forEach(pi -> payInfoDao.insert(pi));
    }
}
```

이 코드를 보면 파일 경로가 하드 코딩되어 있다. 이 코드를 테스트하려면 해당 경로에 파일이 반드시 위치해야 한다. 하드 코딩된 경로 뿐만 아니라 

하드 코딩된 IP주소, 포트 번호도 테스트를 어렵게 만든다

### 의존 객체를 직접 생성

앞의 코드에서 테스트를 어렵게 만드는 또 다른 요인은 의존 대상을 직접 생성하고 있다는 점이다. 

```java
public class PaySync{
  //의존 대상을 직접 생성
  private PayInfoDao payInfoDao = new PayInfoDao();
  
  public void sync() throws IoException{
    ...생략
      payInfos.forEach(pi -> payInfoDao.insert(pi));
  }
}
```

이 코드를 테스트하려면 PayInfoDao가 올바르게 동작하는데 필요한 모든 환경을 구성해야 한다. DB를 준비해야 하고 필요한 테이블도 만들어야한다

테스트를 실행하면 데이터가 DB에 추가되므로 같은 테스트를 다시 실행하기 전에 기존에 들어간 데이터를 삭제해야 한다. 그렇지 않으면 중복 데이터로

인해 데이터 삽입에 실패하게 된다.

### 정적 메소드 사용

정적 메소드를 사용해도 테스트가 어려워질 수 있다.

```java
public class LoginService {
    private String authKey = "somekey";
    private CustomerRepository customerRepo;

    public LoginService(CustomerRepository customerRepo) {
        this.customerRepo = customerRepo;
    }
    public LoginResult login(String id, String pw) {
        int resp = 0;
        boolean authorized = AuthUtil.authorize(authKey);
        if (authorized) {
            resp = AuthUtil.authenticate(id, pw);
        } else {
            resp = -1;
        }
        if (resp == -1) return LoginResult.badAuthKey();

        if (resp == 1) {
            Customer c = customerRepo.findOne(id);
            return LoginResult.authenticated(c);
        } else {
            return LoginResult.fail(resp);
        }
    }
}
```

이코드는 AuthUtil 클래스의 정적 메소드를 사용하고 있다. AuthUtil 클래스가 인증 서버와 통신하는 경우 이 코드를 테스트하려면 동작하고 있는

인증 서버가 필요하다. AuthUtil 클래스가 통신할 인증 서버 정보를 시스템 프로퍼티에서 가져온다면 시스템 프로퍼티도 테스트 환경에 맞게 설정해야

한다. 게다가 다양한 상황을 테스트하려면 인증 서버에 저장되어 있는 유효한 아이디와 암호를 사용해야 한다. 인증 서버 담당자가 휴가라도 가면 

돌아와야 할 때까지 기다려야 할 수도 있다.

### 실행 시점에 따라 달라지는 결과

다음 코드를 보자. 이 코드는 특정 사용자의 포인트를 계산하는 로직을 담고 있다.

```java
public class UserPointCalculator {
    private SubscriptionDao subscriptionDao;
    private ProductDao productDao;

    public UserPointCalculator(SubscriptionDao subscriptionDao,
                               ProductDao productDao) {
        this.subscriptionDao = subscriptionDao;
        this.productDao = productDao;
    }

    public int calculatePoint(User u) {
        Subscription s = subscriptionDao.selectByUser(u.getId());
        if (s == null) throw new NoSubscriptionException();
        Product p = productDao.selectById(s.getProductId());
        LocalDate now = LocalDate.now();
        int point = 0;
        if (s.isFinished(now)) {
            point += p.getDefaultPoint();
        } else {
            point += p.getDefaultPoint() + 10;
        }
        if (s.getGrade() == GOLD) {
            point += 100;
        }
        return point;
    }
}
```

calculatePoint() 메소드는 사용자의 구독 상태나 제품에 따라 계산한 겨로가 값을 리턴한다. 여기서 테스트를 어렵게 만드는 코드는 Localdate.now() 

이다. 같은 테스트 코드라도 이에 따라 실행 결과가 달라진다. 어제까지는 문제 없이 성공하던 테스트가 오늘은 깨질 수 있는 것이다.

Random을 이용해서 임의 값을 사용하는 코드도 비슷하다. Random이 생성한 값에 따라 실행 결과가. 달리잘 수 있다. 이렇게 테스트를 실행하는 시점에

따라 테스트 결과가 달라진다면 그 테스트는 믿을 수 없게 된다.

### 역할이 섞여 있는 코드

위 코드의 또 다른 문제는 포인트 계산 로직만 테스트하기 어렵다는 점이다. 포인트 계산 결과를 테스트하려면 SubscriptionDao와 ProductDao에 

대한 대역을 구성해야 한다. 사실 포인트 계산 자체는 이 두 DAO와 상관이 없다. 포인트 계산에 필요한 것은 Subscription과 시간 그리고 Product이다.

하지만 포인트 계산만 테스트할 수 없다. SubscrptionDao와 ProductDao의 대역을 알맞게 설정해야만 포인트 계산도 가능해진다.

### 그 외 테스트가 어려운 코드

이외에 테스트 대상이 다음과 같다면 테스트가 어려울 수 있다.

- 메소드 중간에 소켓 통신 코드가 포함되어 있다
- 콘솔에서 입력을 받거나 결과를 콘솔에 출력한다
- 테스트 대상이 사용하는 의존 대상 클래스나 메소드가 final이다. 이 경우 대역으로 대체가 어려울 수 있다
- 테스트 대상의 소스를 소유하고 있지 않아 수정이 어렵다

## 테스트 가능한 설계

앞에서 살펴본 코드의 테스트가 어려운 주된 이유는 의존하는 코드를 교체할 수 있는 수단이 없기 때문이다. 상황에 따라 알맞은 방법을 적용하면 의존

코드를 교체할 수 있게 만들 수 있다. 몇가지 주요방법을 보자

### 하드 코딩된 상수를 생성자나 메소드 파라미터로 받기

하드 코딩된 경로가 테스트가 어려운 이유는 테스트 환경에 따라 경로를 다르게 줄 수 있는 수단이 없기 때문이다. 하드 코딩된 상수 때문에 테스트가 

힘들다면 해당 상수를 교체할 수 있는 기능을 추가하면 된다. 쉬운 방법은 생성자나 세터를 이용해서 경로를 전달받는 것이다. 

아래의 코드는 세터를 사용한 예를 보여준다. 파일 경로를 필드에 보관하도록 수정했다

```java
public class PaySync{
  private String filePath = "D:\\data\\pay\\cp0001.csv";
  
  public void setFilePath(String filePath){
    this.filePath = filePath;
  }
  
  public void sync() throws IOException{
    Path path = Paths.get(filePath);
    ...
  }
}
```

파일 경로를 변경할 수 있게 세터 메소드를 추가했다면 테스트 코드는 알맞게 파일 경로를 변경해서 테스트할 수 있다. 다음은 그 예이다.

```java
@Test
void someTest() throws IOException {
  PaySync paySync = new PaySync();
  paySync.setFilePath("src/test/resources/c0111.csv");
  
  paySync.sync();
  ...결과 검증
}
```

파일 경로를 변경하는 또 다른 방법은 메소드를 실행할 때 인자로 전달 받는 것이다

```java
public class PaySync{
  public void sync(String filePath) throws IOException{
    Path path = Paths.get(filePath);
    ...생략
  }
}
```

상수 대신 메소드 파라미터로 값을 전달받도록 수정하면 다음과 같이 테스트 코드에서 파일 경로를 제어할 수 있게 된다

```java
@Test
void someTest() throws IOException{
  PaySync paySync = new PaySync();
  paySync.sync("src/test/resources/c0111.csv");
  
  ...결과 검증
}
```

### 의존 대상을 주입 받기

의존 대상은 주입 받을 수 있는 수단을 제공해서 교체할 수 있도록 한다. 생성자나 세터를 주입수단으로 이용하면 된다. 생성자나 세터를 통해 의존 대상을

교체할 수 있게 되면 실제 구현 대신에 대역을 사용할 수 있어 테스트를 보다 원활하게 작성할 수 있다. 아래의 코드는 생성자를 이용해서 의존대상을

주입하게 수정한 코드이다.

```java
public class PaySync{
  private PayInfoDao payInfoDao;
  private String filePath = "D:\\data\\pay\\cp0001.csv";
  
  public PaySync(PayInfoDao payInfoDao){
    this.payInfoDao = payInfoDao;
  }
  public void setFilePath(String filePath){
    this.filePath = filePath;
  }
  
  public void sync() throws IOException{
    Path path = Paths.get(filePath);
    List<PayInfo> payInfos = Files.lines(path)
      .map(...생략)
      .collect(Collectors.toList());
    
    payInfos.forEach(pi -> payInfoDao.insert(pi));
  }
}
```

만약 많은 레거시 코드에서 생성자 없느 버전을 사용하고 있다면 아래와 같이 기존 코드는 그대로 유지하고 세터를 이용해서 의존대상을 

교체할 수 있도록 하면 된다.

```java
public class PaySync{
  private PayInfoDao payInfoDao = new PayInfoDao();
  private String filePath = "D:\\data\\pay\\cp0001.csv";
  
  public void setPayInfoDao(PayInfoDao payInfoDao){
    this.payInfoDao = payInfoDao;
  }
  public void sync() throws IOException{
    Path path=Paths.get(filePath);
    List<PayInfo> payInfos = Files.lines(path)
      .map(...생략)
      .collect(Collectors.toList());
    
    payInfos.forEach(pi -> payInfoDao.insert(pi));
  }
  
}
```

의존 대상을 교체할 수 있도록 코드를 수정했다면 이제 대역을 사용해서 테스트를 진행할 수 있다

```java
public class PaySyncTest{
  //대역 생성
  private MemoryPayInfoDao memoryDao = new MemoryPayInfoDao();
  
  @Test
  void allDataSaved() throws IOException{
    PaySync paySync = new PaySync();
    paySync.setPayInfoDao(memoryDao); //대역으로 교체
    paySync.setFilePath("src/test/resources/c0111.csv");
    
    paySync.sync();
    
    //대역을 이용한 결과 검증
    List<PayInfo> savedInfos = memoryDao.getAll();
    assertEquals(2,savedInfos.size());
  }
}
```

### 테스트하고 싶은 코드를 분리하기

```java
public int calculatePoint(User u) {
        Subscription s = subscriptionDao.selectByUser(u.getId());
        if (s == null) throw new NoSubscriptionException();
        Product p = productDao.selectById(s.getProductId());
        LocalDate now = LocalDate.now();
        int point = 0;
        if (s.isFinished(now)) {
            point += p.getDefaultPoint();
        } else {
            point += p.getDefaultPoint() + 10;
        }
        if (s.getGrade() == GOLD) {
            point += 100;
        }
        return point;
    }
```

위 코드에서 SubscriptionDao와 ProductDao에 대한 대혁 또는 실제 구현이 필요하고 LocalDate에 대한 값이 필요하다. 테스트하고 싶은

코드는 포인트 계산인데 나머지 코드가 올바르게 동작해야 비로소 포인트 계산에 대한 테스트가 가능하다.

이렇게 기능의 일부만 테스트하고 싶다면 해당 코드를 별도 기능으로 분리해서 테스트를 진행할 수 있다. 예를 들어 포인트를 계산하는 코드를 아래와 같이

별도 클래스로 분리할 수 있다.

```java
public class PointRule {

    public int calculate(Subscription s, Product p, LocalDate now) {
        int point = 0;
        if (s.isFinished(now)) {
            point += p.getDefaultPoint();
        } else {
            point += p.getDefaultPoint() + 10;
        }
        if (s.getGrade() == GOLD) {
            point += 100;
        }
        return point;
    }
}
```

이제 다음과 같이 포인트 계산 기능만 테스트할 수 있다

```java
public class PointRuleTest{
  @Test
  void 만료전_GOLD등급은_130포인트{
    PointRule rule = new PointRule();
    Subscription s = new Subscription(
    			LocalDate.of(2019,5,5),
    			Grade.GOLD);
    Product p =new Product();
    p.setDefaultPoint(20);
    
    int point = rule.calculate(s,p,LocalDate.of(2019,5,1));
    
    assertEquals(130,point);
  }
}
```

원래 포인트 계산을 포함하던 코드는 분리한 기능을 사용하도록 수정한다

```java
public int calculatePoint(User u) {
        Subscription s = subscriptionDao.selectByUser(u.getId());
        if (s == null) throw new NoSubscriptionException();
        Product p = productDao.selectById(s.getProductId());
        LocalDate now = times.today();
        return pointRule.calculate(s, p, now);
    }
```

포인트 계산 기능 자체를 대역으로 변경하고 싶다면 '의존 대상을 주입 받기' 에서 설명한 것처럼 세터를 이용해서 의존 대상을 주입할 수 있게 하면된다

```java
public class UserPointCalculator{
  private PointRule pointRule = new PointRule(); //기본 구현을 사용
  private SubscriptionDao subscriptionDao;
  private ProductDao productDao;
  
  ...생략
  
  // 별도로 분리한 계산 기능을 주입할 수 있는 세터 추가
  // 테스트 코드에서 대역으로 계산 기능을 대체할 수 있게 함
  public void setPointRule(PointRule pointRule){
    this.pointRule = pointRule;
  }
  
  public int calculatePoint(User u){
    Subscription s = subscriptionDao.selectByUser(u.getId());
    if(s==null) throw new NoSubscriptionException();
    Product p =productDao.selectById(s.getProductId());
    LocalDate now = LocalDate.now();
    return pointRule.calculate(s,p,now);
  }
}
```

### 시간이나 임의 값 생성 기능 분리하기

테스트 대상이 시간이나 임의 값을 사용하면 테스트 시점에 따라 테스트 결과가 달라진다. 이 경우 테스트 대상이 사용하는 시간이나 임의 값을 제공하는

기능을 별도로 분리해서 테스트 가능성을 높일 수 있다. 다음 코드에서 LocalDate.now() 코드는 테스트를 실행하는 일자에 따라 값이 달라지므로

테스트 결과도 함께 달라지게 만든다

```java
public class DailyBatchLoader {
    private String basePath = ".";

    public int load() {
        LocalDate date = LocalDate.now();
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy/MM/dd");
        Path batchPath = Paths.get(basePath, date.format(formatter), "batch.txt");

        // ...batchPath에서 데이터를 읽어와 저장하는 코드

        return 0;
    }
}
```

현재 일자를 구하는 기능을 분리하고 분리한 대상을 주입할 수 있게 변경하면 테스트를 원하는 상황으로 쉽게 제어할 수 있다. 먼저 현재 일자를 구하는

기능을 다음과 같이 분리하자

```java
public class Times{
  public LocalDate today(){
    return LocalDate.now();
  }
}
```

이제 DailyBatchLoader가 분리한 Times를 이용해서 오늘 일자를 구하도록 수정한다. 수정한 결과는 다음과 같다

```java
public class DailyBatchLoader {
    private Times times = new Times();
    private String basePath = ".";


    public void setTimes(Times times) {
        this.times = times;
    }

    public int load() {
        LocalDate date = times.today();
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy/MM/dd");
        Path batchPath = Paths.get(basePath, date.format(formatter), "batch.txt");

       //.. batchPath에서 데이터를 읽어와 저장하는 코드
      return result;
    }
}
```

테스트 코드는 Times 대역을 이용해서 원하는 상황을 쉽게 구성할 수 있다. 예를 들어 테스트에 사용할 파일을 src/test/resources/2019/01/01 폴더에

저장했다면 다음과 같이  Times의 대역을 이용해서 DailyBatchLoader가 사용할 일자를 지정할 수 있다.

```java
public class DailyBatchLoaderTest {
    private Times mockTimes = Mockito.mock(Times.class);
    private final DailyBatchLoader loader = new DailyBatchLoader();

    @BeforeEach
    void setUp() {
        loader.setBasePath("src/test/resources");
        loader.setTimes(mockTimes);
    }

    @Test
    void loadCount() {
        given(mockTimes.today()).willReturn(LocalDate.of(2019, 1, 1));

        int ret = loader.load();

        assertEquals(3, ret);
    }
}
```

임의 값도 비슷하다. 임의 값을 제공하는 라이브러리를 직접 사용하지말고 별도로 분리한 타입을 사용해서 대역으로 처리할 수 있어야 테스트 간으하게

만들 수 있다.

### 외부 라이브러리는 직접 사용하지말고 감싸서 사용하기

테스트 대상이 사용하는 외부 라이브러리를 쉽게 대체할 수 없는 경우도 있다. 외부 라이브러리가 정적 메소드를 제공한다면 대체할 수 없다.

아래의 코드를 보자

```java
public LoginResult login(String id,String pw){
  int resp=0;
  boolean authorized = AuthUtil.authorize(authKey);
  if(authrozied)
    resp = AuthUtil.authenticate(id,pw);
  else
    resp= -1;
  if(resp==-1) return LoginResult.badAuthKey();
  if(resp==1){
    Customer c = customerRepo.findOne(id);
    return LoginResult.authenticated(c);
  }else
    return LoginResult.fail(resp);
}
```

이 코드에서 AuthUtil클래스가 외부에서 제공한 라이브러리에 포함되어 있다고 하자 . AuthUtil.authorize() 메소드와  AuthUtil.authenticate() 메소드는

정적 메소드이기 때문에 대역으로 대체하기 어렵다.

이렇게 대역으로 대체하기 어려운 외부 라이브러리가 있다면 외부 라이브러리를 직접 사용하지 말고 외부 라이브러리와 연동하기 위한 타입을 따로 만든다.

그리고 테스트 대상은 이렇게 분리한 타입을 사용하게 바꾼다. 테스트 대상 코드는 새로 분리한 타입을 사용함으로써 외부 연동이 필요한 기능을 쉽게

대역으로 대체할 수 있게 된다.

예를들어  AuthUtil을 사용하는 아래와 같은 클래스로 분리할 수 있다.

```java
public class AuthService {
    private String authKey = "somekey";

    public int authenticate(String id, String pw) {
        boolean authorized = AuthUtil.authorize(authKey);
        if (authorized) {
            return AuthUtil.authenticate(id, pw);
        } else {
            return -1;
        }
    }
}
```

```java
public class LoginService {
    private AuthService authService = new AuthService();
    private CustomerRepository customerRepo;

    public LoginService(CustomerRepository customerRepo) {
        this.customerRepo = customerRepo;
    }

    public void setAuthService(AuthService authService) {
        this.authService = authService;
    }

    public LoginResult login(String id, String pw) {
        int resp = authService.authenticate(id, pw);
        if (resp == -1) return LoginResult.badAuthKey();

        if (resp == 1) {
            Customer c = customerRepo.findOne(id);
            return LoginResult.authenticated(c);
        } else {
            return LoginResult.fail(resp);
        }
    }

}
```

AuthService를 대역으로 대체할 수 있게 되었으므로 인증 성공 상황과 실패 상황에 대해 LoginService가 올바르게 동작하는지 검증하는 테스트 코드를

만들 수 있다. 의존하는 대상이  Final클래스이거나 의존 대상의 호출 메소드가 final이어서 대역으로 재정의할 수 없는 경우에도 동일한 기법을

적용해서 테스트 가능하게 만들 수 있다.
