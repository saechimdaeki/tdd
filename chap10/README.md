# 10 테스트 코드와 유지보수

## 테스트 코드와 유지보수

빠른 서비스 출시를 위해 CI와 CD를 도입하는 곳이 증가하고 잇다. 지속적으로 코드를 통합하고 출시 가능한 상태로 만들고 배포하려면 새로 추가 한

코드가 기존 기능을 망가뜨리지 않는지 확인할 수 있어야 하며 이런 이유로 자동화 테스트는 CI/CD의 필수 요건 중 하나이다. TDD를 하는 과정에서 작성한

테스트 코드는 CI/CD에서 자동화 테스트로 사용되어 버그가 배포되는 것을 막아주고 이는 소프트웨어 품질이 저하되는 것을 방지한다.

테스트 코드는 그 자체로 코드이기 때문에 제품 코드와 동일하게 유지보수 대상이 된다. 깨지는 테스트를 방치하는 상황이 길어지면 다음과 같은 문제가 발생

할 수 있다.

- 실패한 테스트가 새로 발생해도 무감각해진다. 테스트 실패 여부에 상관없이 빌드하고 배포하기 시작한다
- 빌드를 통과시키기 위해 실패한 테스트를 주석 처리하고 실패한 테스트는 고치지 않는다.

테스트 코드는 코드를 변경했을때 기존 기능이 올바르게 동작하는지 확인하는 회귀 테스트를 자동화하는 수단으로 사용되는데 깨진 테스트를 방치하기

시작하면 회귀 테스트가 검증하는 범위가 줄어든다. 실패한 테스트를 통과시키기 위해 많은 노력이 필요하면 점점 테스트코드에서 멀어지고

TDD에서도 멀어진다. 이런 악순환이 발생하지 않으려면 테스트코드 자체의 유지보수성이 좋아야 한다. 테스트 코드를 유지보수하기 좋아야 

지속적으로 테스트를 작성하게 되고 결과적으로 소프트웨어의 품질이 떨어지는 것도 막을 수 있다.

유지보수하기 좋은 코드를 만들기 위해 필요한 좋은 패턴과 원칙이 존재하는 것처럼 좋은 테스트 코드를 만들려면 몇가지 주의해야할 사항이 있다.

## 변수나 필드를 사용해서 기댓값 표현하지 않기

```java
@Test
void dateFormat(){
  LocalDate date=LocalDate.of(1945,8,15);
  String dateStr=formatDate(date);
  assertEquals(date.getYear()+"년 "+
              date.getMonthValue()+"월 "+
              date.getDayOfMonth()+"일",dateStr);
}
```

이 단언은 논리적으로 맞지만 문자열 연결이 있어서 코드가 복잡하고 실수로 date.getMonthValue() 대신 date.getMonth()를 사용하면 테스트를 실행해서 테스트가 깨져야

비로소 실수를 알아채기도 한다. 

```java
@Test
void dateFormat(){
  LocalDate date=LocalDate.of(1945,8,15);
  String dateStr= formatDate(date);
  assertEquals("1945년 8월 15일",dateStr);
}
```

이코드는 복잡하지 않고 기대하는 값도 명확하게 표현하고 있다. 테스트가 깨진다면 formatDate()메소드만 확인하면 된다.

다음코드를 보자 이코드는 기대하는 값을 기술할 때 로컬변수와 필드를 사용하고 있다.

```java
private List<Integer> answers = Arrays.asList(1,2,3,4);
private Long respondentId = 100L;

@DisplayName("답변에 성공하면 결과 저장함")
@Test
puvlic void saveAnswerSuccessfully(){
  //답변할 설문이 존재
  Survey survey= SurveyFactory.createApprovedSurvey(1L);
  surveyRepository.save(survey);
  
  //설문 답변
  SurveyAnswerRequest surveyAnswer = SurveyAnswerRequest.builder()
    				.surveyId(survey.getId())
    				.respondentId(respondentId)
    				.answers(answers)
    				.build();
  
  svc.answerSurvey(surveyAnswer);
  
  //저장 결과 확인
  SurveyAnswer savedAnswer=
    	memoryRepository.findBySurveyAndRespondent(
  			survey.getId(),respondentId);
  assertAll(
  	()-> assertEquals(respondentId,savedAnswer.getRespondentId()),
    ()-> assertEquals(answers.size(),savedAnswer.getAnswers().size()),
    ()-> assertEquals(answers.get(0),savedAnswer.getAnswers().get(0)),
    ()-> assertEquals(answers.get(1),savedAnswer.gerAnswers().get(1))
    );
}
```

<code>  	()-> assertEquals(respondentId,savedAnswer.getRespondentId())</code> 만약 이부분에서 NPE가 발생하면 어떻게 할까? survey변수와 respondentId필드의 

값을 확인해야 한다. 테스트에 성공하더라도 테스트 코드를 처음보는 사람은 변수와 필드를 오가며 테스트코드를 이해해야 한다. 이제 이를 다음과 같이 수정해보자.

객체를 생성할 때 변수와 필드 대신 값 자체를 사용했다. 

```java
private List<Integer> answers = Arrays.asList(1,2,3,4);
private Long respondentId = 100L;

@DisplayName("답변에 성공하면 결과 저장함")
@Test
puvlic void saveAnswerSuccessfully(){
  //답변할 설문이 존재
  Survey survey= SurveyFactory.createApprovedSurvey(1L);
  surveyRepository.save(survey);
  
  //설문 답변
  SurveyAnswerRequest surveyAnswer = SurveyAnswerRequest.builder()
    				.surveyId(1L)
    				.respondentId(100L)
    				.answers(Arrays.asList(1,2,3,4))
    				.build();
  
  svc.answerSurvey(surveyAnswer);
  
  //저장 결과 확인
  SurveyAnswer savedAnswer=
    	memoryRepository.findBySurveyAndRespondent(1L,100L);
  assertAll(
  	()-> assertEquals(100L,savedAnswer.getRespondentId()),
    ()-> assertEquals(4,savedAnswer.getAnswers().size()),
    ()-> assertEquals(1,savedAnswer.getAnswers().get(0)),
    ()-> assertEquals(2,savedAnswer.gerAnswers().get(1))
    );
}
```

이렇게 수정하면 객체를 생성할 때 사용한 값이 무엇인지 알아보기 위해 필드와 변수를 참조하지 않아도 된다. 단언할 때 사용한 값이 무엇인지 알기 위해 필드와 변수를 오갈

필요도 없다.

## 두 개 이상을 검증하지 않기 

처음 테스트코드를 작성하면 한 테스트 메소드에 가능한 많은 단언을 하려고 시도한다. 그 과정에서 서로 다른 검증을 섞는 경우가 있다

```java
@DisplayName("같은 ID가 없으면 가입에 성공하고 메일을 전송함")
@Test
void registerAndSendMail(){
  userRegister.register("id","pw","email");
  
  //검증1: 회원 데이터가 올바르게 저장되엇는지 검증
  User savedUser = fakeRepository.findById("id");
  assertEquals("id",savedUser.getId());
  assertEquals("email",savedUser.getEmail());
  
  //검증2: 이메일 발송을 요청했는지 검증
  ArgumentCaptor<String> captor = ArgumentCaptor.forClass(String.class);
  BDDMockito.then(mockEmailNotifier)
    	.should().snedRegisterEmail(captor.capture());
  
  String realEmail = captor.getValue();
  assertEquals("email@email.com",realEmail);
}
```

이 테스트는 두 가지를 검증한다. 첫 번째는 회원 가입 이후에 데이터가 올바르게 저장되는지 검증하고, 두 번째는 이메일 발송을 올바르게 요청하는지 검증한다. 이 테스트가 

잘못된 것은 아니지만 한 테스트에서 검증하는 내용이 두 개 이상이면 테스트 결과를 확인할 때 집중도가 떨어진다. 만약 첫번째 검증이 실패하면 테스트는 거기서 멈춘다.

첫번째 검증이 통과되어야 그때서야 두번째 검증이 성공했는지 여부를 확인할 수 있다.

한 테스트 메소드에서 서로 다른 내용을 검증한다면 각 검증 대상을 별도로 분리해서 테스트의 집중도를 높일 수 있다. 

```java
@DisplayName("같은 ID가 없으면 가입 성공함")
@Test
void noDupId_RegisterSuccess(){
  userRegister.register("id","pw","email");
  
  User savedUser = fakeRepository.findById("id");
  assertEquals("id",savedUser.getId());
  assertEquals("email",savedUser.getEmail());
}

@DisplayName("가입하면 메일을 전송함")
@Test
void whenRegisterThenSendMail(){
  userRegister.register("id","pw","email");

  ArgumentCaptor<String> captor = ArgumentCaptor.forClass(String.class);
  BDDMockito.then(mockEmailNotifier)
    	.should().snedRegisterEmail(captor.capture());
  
  String realEmail = captor.getValue();
  assertEquals("email@email.com",realEmail);
}

```

만약 첫 번째 메소드에서 테스트에 실패하면 구체적인 실패 위치를 몰라도 저장한 데이터를 저장하는 것이 잘못되었다는 것을 알 수 있다. 마찬가지로 두 번째 메소드에서 테스트에 실패

하면 메일 전송 과정이 잘못되었다는 것을 알 수 있다. 검증 대상이 명확하게 구분된다면 테스트 메소드도 구분하는 것이 유지보수에 유리하다.

## 정확하게 일치하는 값으로 모의 객체 설정하지 않기

다음 코드를 보자 모의 객체를 이용해서 "pw" 문자열은 약한 암호로 처리하도록 지정하고 있다.

```java
@DisplayName("약한 암호면 실패")
@Test
void weakPassword(){
  BDDMockito.given(mockPasswordChecker.checkPasswordWeak("pw"))
    				.willReturn(true);
  
  assertThrows(WeakPasswordException.class, () -> {
    userRegister.register("id","pw","email");
  });
}
```

이 테스트는 작은 변화에도 실패한다 예를들어 다음과 같이 코드를 변경했다고 하자

​	    userRegister.register("id","pwa","email");

모의 객체는 "pw"인 경우에만 true를 리턴하도록 지정했기 때문에 "pwa"를 지정하면 false를 리턴해서 테스트에 실패하게 된다.

이 코드는 약한 암호인 경우 UserRegister가 원하는대로 동작하는지 확인하기 위한 테스트이지 "pw"나 "pwa" 가 약한 암호인지 확인하는 테스트 코드가 아니다.

따라서 다음과 같이 변경해도 원하는 테스트를 수행할 수 있다.

```java
@DisplayName("약한 암호면 가입 실패")
@Test
void weakPassword(){
  BDDMockito
    .given(mockPasswordChecker.checkPasswordWeak(Mockito.anyString()))
    .willReturn(true);
  
  assertThrows(WeakPasswordException.class, () -> {
    userRegister.register("id","pw","email");
  });
}
```

Mockito.anyString() 을 사용하였는데 이 코드는 임의의 String값에 일치한다는 것을 의미한다. 이제 "pw"가 아닌 다른 문자열을 인자로 전달해도 테스트는 깨지지 않는다.

모의 객체를 호출했는지 여부를 확인하는 경우도 동일하다. 아래 코드처럼 특정값을 사용해서 호출 여부를 검증할 경우 register() 메소드를 호출할 때 파라미터 값이 조금만 바뀌어도

테스트가 깨지게 된다.

```java
@DisplayName("회원 가입시 암호 검사 수행함")
@Test
void checkPassword(){
  userRegister.register("id","pw","email");
  
  BDDMockito.then(mockPasswordChecker)
    				.should()
    				.checkPasswordWeak("pw");
}
```

모의 객체는 가능한 범용적인 값을 사용해서 기술해야 한다. 한정된 값에 일치하도록 모의객체를 사용하면 약간의 코드 수정만으로도 테스트는 실패하게 된다. 이경우 테스트 코드의

일부 값을 수정하면 모의 객체 관련 코드도 함께 수정하는 번거로움이 있다.

테스트의 의도를 해치지 않는 범위에서 "pw"와 같은 특정한 값보다는 Mockito.anyString() 과 같은 범용적인 값을 사용해야한다. 이렇게 해야 약간의 코드 수정때문에 테스트가 실패

하는 것을 방지할 수 있다.

## 과도하게 구현 검증하지 않기

테스트 코드를 작성할 때 주의할점은 테스트 대상의 내부 구현을 검증하는 것이다. 모의 객체를 처음 사용할 때 특히 이런 유혹에 빠지기 쉽다. 

```java
@DisplayName("회원 가입시 암호 검사 수행함")
@Test
void checkPassword(){
  userRegister.register("id","pw","email");
  
  // PasswordChecker#checkPasswordWeak() 메소드 호출 여부 검사
  BDDMockito.then(mockPasswordChecker)
    			.should()
    			.checkPasswordWeak(Mockito.anyString());
  
  // UserRepository#findById() 메소드를 호출하지 않는 것을 검사
  BDDMockito.then(mockRepository)
    			.should(Mockito.never())
    			.findById(Mockito.anyString());
}
```

내부 구현을 검증하는 것이 나쁜 것은 아니지만 한가지 단점이 있다. 그것은 바로 구현을 조금만 변경해도 테스트가 깨질 가능성이 커진다는 것이다. 예를들어 중복 ID가 존재하는지

먼저 확인하고 그 다음에 아이디가 약한지 검사하도록 register() 메소드의 구현을 변경한다고 하자. 이러면 위 테스트는 깨진다. 왜냐면 중복 아이디를 검사하는 과정에서 

UserRepository의 findById() 메소드를 호출하기 때문이다.

내부 구현은 언제든지 바뀔 수 있기 때문에 테스트 코드는 내부 구현보다 실행 결과를 검증해야 한다. 예제 코드의 경우 register() 메소드가 PasswordChecker#checkPasswordWeak()

메소드를 호출하는지 검증하는 것보다 약한 암호일 때 register() 의 결과가 올바른지 검증해야한다. 그렇게 함으로써 내부 구현을 일부 바꿔도 테스트가 깨지지 않게 유지할 수 있다.

예를들어 다음의 레거시 코드가 있다고하자

```java
public void changeEmail(String id, String email){
  int cnt=userDao.countById(id);
  if(cnt==0) throw new NoUserException();
  userDao.updateEmail(id,email);
}
```

레거시 코드에서 DAO는 다양한 update, select 메소드를 정의하고 있는 경우가 많기 때문에 메모리를 이용한 가짜 구현으로 대체하기 쉽지 않다. 그래서 레거시 코드에 대한 테스트

코드를 작성할 때는 모의 객체를 많이 활용한다. 위 코드에 대한 테스트코드도 다음과 같이 모의 객체를 이용해서 작성할 수 있다.

```java
@Test
void changeEmailSuccessfully(){
  given(mockDao.countById(Mockito.anyString())).willReturn(1);
  
  emailService.changeEmail("id","new@somehost.com");
  
  then(mockDao).should()
    	.updateEmail(Mockito.anyString(),Mockito.matches("new@somehost.com"));
}
```

이 코드는 이메일을 수정했는지 확인하기 위해 모의 객체의 updateEmail() 메소드가 호출됐는지 확인한다. 모의 객체를 호출하는지 여부를 확인하는 것은 구현을 검증하는 것이지만

이메일이 변경되는지 확인할 수 있는 수단은 이것뿐이다.

## 셋업을 이용해서 중복된 상황을 설정하지 않기

테스트 코드를 작성하다 보면 각 테스트 코드에서 동일한 상황이 필요할 때가 있다. 이경우 중복 코드를 제거하기 위해 @BeforeEach 메소드를 이용해서 상황을 구성할 수 있다.

```java
@BeforeEach
void setUp(){
  changeService = new ChangeUserService(memoryRepository);
  memoryRepository.save(new User("id","pw",new Address("서울","북부")));
}

@Test
void noUser(){
  assertThrows(
  UserNotFoundException.class , 
  ()-> changeService.changeAddress("id2",new Address("서울","남부")));
}

@Test
void changeAddress(){
  changeService.changeAddress("id",new Address("서울","남부"));
  
  User user=memoryRepository.findById("id");
  assertEquals("서울",user.getAddress().getCity());
}

@Test
void changePw(){
  changeService.changePw("id","pw","newpw");
  
  User user=memoryRepository.findById("id");
  assertTrue(user.matchPassword("newpw"));
}

@Test
void pwNotMatch(){
  assertThrows(
  IdPwNotMatchException.class, 
  () -> changeService.changePw("id","pw2","newpw"));w
}
```

이렇게 중복을 제거하고 코드 길이도 짧아져서 코드 품질이 좋아졌다고 생각할 수 있지만, 테스트 코드에서는 상황이 달라진다. 예를들어 pwNotMatch()  테스트 메소드가 실패해서

면달만에 테스트 코드를 다시 볼 일이 생겼다고 하자. 테스트에 실패한 이유를 알려면 어떤 상황인지 확인해야 한다. 처음 테스트 코드를 작성할 때는 셋업 메소드로 상황을 구성해도

내용 분석에 어려움이 없지만 몇달만에 다시 코드를 보면 기억이 잘 나지 않기 때문에 셋업 메소드를 확인해야한다. 즉 코드를 위아래로 이동하면서 실패한 원인을 분석해야 한다.

셋업 메소드를 이용한 상황 설정으로 인해 발생할 수 있는 또 다른 문제는 테스트가 깨지기 쉬운 구조가 된다는 것이다. 모든 테스트 메소드가 동일한 상황 코드를 공유하기 때문이다.

셋업 메소드에서 User객체를 생성할때 사용한 "pw"를 "pw2" 로 바꿨다고 하자

```java
@BeforeEach
void setUp(){
  changeService = new ChangeUserService(memoryRepository);
  memoryRepository.save(new User("id","pw2",new Address("서울","북부")));
}
```

이 변경으로 인해 changePw( )테스트와 pwNotMatch() 테스트가 실패한다. 이렇게 테스트가 깨지는 것을 방지하려면 셋업 메소드의 상황 설정 코드를 변경하기 전에 영향을 받는

테스트 메소드가 있는지 확인해야 한다.

테스트 메소드는 검증을 목표로 하는 하나의 완전한 프로그램이어야 한다. 메소드는 별도 프로그램으로서 검증 내용을 스스로 잘 설명할 수 있어야 한다. 그러기 위해서는 상황 구성

코드가 테스트 메소드 안에 위치해야 한다.

상황설정 코드를 테스트 메소드에서 직접 하도록 변경해보자.

```java
@BeforeEach
void setUp(){
  changeService = new ChangeUserService(memoryRepository);
}

@Test
void noUser(){
  assertThrows(
  UserNotFoundException.class , 
  ()-> changeService.changeAddress("id",new Address("서울","남부")));
}

@Test
void changeAddress(){
  memoryRepository.save(new User("id","pw",new Address("서울","북부")));
  
  changeService.changeAddress("id",new Address("서울","남부"));
  
  User user=memoryRepository.findById("id");
  assertEquals("서울",user.getAddress().getCity());
}

@Test
void changePw(){
  memoryRepository.save(new User("id","pw",new Address("서울","북부")));
  
  changeService.changePw("id","pw","newpw");
  
  User user=memoryRepository.findById("id");
  assertTrue(user.matchPassword("newpw"));
}

@Test
void pwNotMatch(){
  memoryRepository.save(new User("id","pw",new Address("서울","북부")));
  
  assertThrows(
  IdPwNotMatchException.class, 
  () -> changeService.changePw("id","pw2","newpw"));w
}
```

코드는 다소 길어졌지만 테스트메소드 자체는 스스로를 더 잘 설명하고 있다. 테스트에 실패해도 코드를 이리저리 왔다 갔다 하면서 보지 않아도 된다. 실패한 테스트 메소드 위주로 코드

를 보면 된다. 각 테스트에 맞게 상황을 설정하는 것도 쉽다. 한 테스트 메소드의 상황을 변경해도 다른 테스트에 영향을 주지 않기 때문이다.

### 통합 테스트에서 데이터 공유 주의하기

셋업을 이용한 상황 설정과 비슷한 것으로 통합 테스트의 DB데이터 초기화가 있다. DB연동을 포함한 통합 테스트를 실행하려면 DB데이터를 알맞게 구성해야 한다. 이를 위한 방법은

테스트를 실행할 때마다 DB데이터를 초기화하는 쿼리를 실행하는 것이다. 스프링 프레임워크를 사용하면 @Sql 애노테이션을 사용해서 테스트를 실행하기 전에 특정 쿼리를 

실행할 수 있다. 

```java
@SpringBootTest
@Sql("classpath:init-data.sql")
public class UserRegisterIntTestUsingSql{
  @Autowired
  private UserRegister register;
  
  @Autowired
  private JdbcTemplate jdbcTemplate;
  
  @Test
  void 동일ID가_이미_존재하면_익셉션(){
    //실행, 결과 확인
    assertThrows(DupIdException.class,
                ()->register.register("cbk","strongpw","email@email.com"));
  }
  
  @Test
  void 존재하지_않으면_저장함(){
    //실행
    register.register("cbk2","strongpw","email@email.com");
    ..생략
  }
}
```

@Sql 애노테이션으로 지정한 sql파일은 다음과 같이 테스트에 필요한 데이터를 초기화한다. 통합 테스트 메소드는 데이터 초기화를 위한 코드를 작성하지 않아도 된다. 

이 방식은 편리하지만 셋업 메소드를 이용한 상황 설정과 마찬가지로 초기화를 위한 쿼리 파일을 조금만 변경해도 많은 테스트가 깨질 수 있고 테스트가 깨지면 관련된 쿼리 파일을

봐야 한다. 이는 테스트 코드의 유지보수를 귀찮고 어렵게 만든다.

통합 테스트 코드를 만들 때는 다음의 두가지로 초기화 데이터를 나눠서 생각해야 한다

- 모든 테스트가 같은 값을 사용하는 데이터: 예)코드값 데이터
- 테스트 메소드에서만 필요한 데이터: 예) 중복 ID 검사를 위한 회원 데이터

코드값 데이터는 (거의) 바뀌지 않는다. 모든 테스트가 동일한 코드값 데이터를 사용해도 문제가 없으며 오히려 서로 다른 코드값 데이터를 사용하면 문제가 발생할 수 있다.

이렇게 모든 테스트가 다른 값을 사용하면 안되는 데이터는 동일한 데이터를 공유해도 된다.

반면에 특정 테스트 메소드에서만 의미 이쓴ㄴ 데이터는 모든 테스트가 공유할 필요가 없다. 이런 데이터는 아래처럼 특정 테스트케이스에서만 생성해서

테스트 코드가 완전히 하나가 되도록 해야 한다.

```java
@Test
void dupId(){
  //상황
  jdbcTemplate.update(
  		"insert into user values(?,?,?) "+
  		"on duplicate key update password = ?, email = ?",
  		"cbk","pw","cbk@cbk.com","pw","cbk@cbk.com");
  
  //실행,결과확인
  assertThrows(DupIdException.class,
              () -> register.register("cbk","strongpw","email@email.com"));
}
```

### 통합 테스트의 상황 설정을 위한 보조 클래스 사용하기

위의 코드는 테스트 메소드를 분석하기는 좋아졌는데 반대로 상황을 만들기 위한 코드가 여러 테스트코드에 중복된다. 테이블 이름이나 칼럼 이름이 바뀌면 여러 테스트 메소드를

수정해야 하므로 유지보수에 좋지 않다. 테스트 메소드에서 직접 상황을 구성하면서 코드 중복을 없애는 방법이 있는데 그것은 바로 상황 설정을 위한 보조 클래스를 사용하는 것이다.

```java
public class UserGivenHelper{
  private JdbcTemplate jdbcTemplate;
  
  public UserGivenHelper(JdbcTemplate jdbcTemplate){
    this.jdbcTemplate=jdbcTemplate;
  }
  
  public void givenUser(String id, String pw, String email){
    jdbcTemplate.update(
    	"insert into user values (?,?,?) "+
    	"on duplicate key update password = ?,email=?",
    	id,pw,email,pw,email);
  }
}
```

상황을 구성하기 위한 보조클래스를 사용하게 변경한 테스트 코드는 다음과 같다.

```java
@Autowired JdbcTemplate jdbcTemplate;
private UserGivenHelper given;

@BeforeEach
void setUp(){
  given = new UserGivenHelper(jdbcTemplate);
}

@Test
void dupId(){
  given.givenUser("cbk","pw","cbk@cbk.com");
  
  //실행, 결과 확인
  assertThrows(DupIdException.class,
              () -> register.register("cbk","strongpw","email@email.com"));
}

```

상황 설정을 위한 보조 도구를 사용하면 위와 같이 givenUser() 라는 메소드 이름을 사용하므로 어떤 상황을 구성하는지 이해할 수 있고 각 테스트 메소드에서 상황을 구성하기 위해

코드가 중복되는 것도 방지할 수 있다.

## 실행 환경이 다르다고 실패하지 않기

같은 테스트 메소드가 실행 환경에 따라 성공하거나 실패하면 안된다. 로컬 개발 환경에서는 성공하는데 빌드 서버에서는 실패한다거나 윈도우에서는 성공하는데 맥 OS에서는 실패하는

식으로 테스트를 실행하는 환경에 따라 테스트를 다르게 동작하면 안된다. 이 전형적인 예가 바로 파일경로이다.

```java
public class BulkLoaderTest{
  private String bulkFilePath = "d:\\mywork\\temp\\bulk.txt";
  
  @Test
  void load(){
    BulkLoader loader = new BulkLoader();
    loader.load(bulkFilePath);
    
    ...생략
  }
}
```

이 파일경로는 D드라이브를 포함한다. D드라이브가 없는 맥 OS에서는 이 테스트를 실행하면 항상 실패한다. 이렇기 때문에  테스트에서 사용하는 파일은 프로젝트 폴더를 기준으로

상대 경로를 사용해야 한다. 예를 들어 "src/test/resources"와 같은 폴더에 "bulk.txt" 파일을 생성하고 테스트 코드는 상대경로로 사용한다

```java
public class BulkLoaderTest{
  private String bulkFilePath = "src/test/resources/bulk.txt";
  
  @Test
  void load(){
    BulkLoader loader = new BulkLoader();
    loader.load(bulkFilePath);
    
    ...생략
  }
}
```

테스트 코드에서 파일을 생성하는 경우에도 특정 OS나 본인의 개발 환경에서만 올바르게 동작하지 않도록 주의해야 한다. 메이븐 프로젝트라면 target폴더에 파일 생성 결과를 저장

하거나 OS가 제공하는 임시 폴더에 파일을 생성하면 실행 환경에 따라 테스트가 다르게 동작하는 것을 방지할 수 있다.

```java
public class ExporterTest{
  @Test
  void export(){
    String folder = System.getProperty("java.io.tmpdir");
    Exporter exporter = new Exporter(folder);
    exporter.export("file.txt");
    
    Path path=Paths.get(folder,"file.txt");
    assertTrue(Files.exists(path));
  }
}
```

이 코드는 실행 환경에 알맞은 임시 폴더 경로를 구해서 동작하기 때문에 환경이 달라 테스트가 실패하는 상황은 벌어지지 않는다.

간혹 특정 OS환경에서만 실행해야 하는 테스트도 있다. 이런경우에는 Junit5가 제공하는 @EnabledOnOs 애노테이션과 @DisabledOnOn애노테이션을 사용해 OS에 따른 테스트

실행 여부를 지정하면 된다.

```java
@Test
@EnabledOnOs({OS.LINUX,OS.MAC})
void callBash(){
  //...
}

@Test
@DisabledOnOs(OS.WINDOWS)
void changeMode(){
  //...
}
```

## 실행 시점이 다르다고 실패하지 않기

다음 코드를 보자. 이 코드는 회원의 만료 여부를 확인하는 기능을 제공한다

```java
public class Member{
  private LocalDateTime expiryDate;
  
  public boolean isExpired(){
    return expiryDate.isBefore(LocalDateTime.now());
  }
}
```

이 기능을 검사하기 위한 테스트 코드는 다음과 같이 작성할 수 있을것이다.

```java
@Test
void notExpired(){
  //테스트 코드를 작성한 시점이 2019년 1월 1일
  LocalDateTime expiry = LocalDateTime.of(2019,12,31,0,0,0);
  Member m =Member.builder().expiryDate(expiry).build();
  assertFalse(m.isExpired());
}
```

이 코드는 만료일을 2019년 12월 31일 0시 0분 0초로 설정하고 isExpired() 메소드의 결과가 false인지 확인한다. 이 코드를 작성한 시점인 2019년 1월 1일에는 테스트에 문제가

발생하지 않지만 2019년 12월 31일에 테스트를 실행하면 깨진다. 깨진 테스트를 복구하기 위해 누군가는 만료일을 2099년으로 변경할 수도 있다.

```java
@Test
void notExpired(){
  //테스트 코드를 작성한 시점이 2019년 1월 1일
  LocalDateTime expiry = LocalDateTime.of(2099,12,31,0,0,0);
  Member m =Member.builder().expiryDate(expiry).build();
  assertFalse(m.isExpired());
}
```

이렇게 변경하면 2019년 기준으로 80년은 테스트가 끼지지 않겠지만 이보다는 테스트 코드에서 시간을 명시적으로 제어할 수 있는 방법을 선택하는 것이 좋다. 

Member#isExpired의 경우 시간 파라미터로 전달받아 비교하는 방법을 사용할 수 있다.

```java
public class Member{
  private LocalDateTime expiryDate;
  
  public boolean passedExpiryDate(LocalDateTime time){
    return expiryDate.isBefore(time);
  }
}
```

이제 테스트 코드는 다음과 같이 바뀐다. 

```java
@Test
void notExpired(){
  //테스트 코드를 작성한 시점이 2019년 1월 1일
  LocalDateTime expiry = LocalDateTime.of(2019,12,31,0,0,0);
  Member m =Member.builder().expiryDate(expiry).build();
  assertFalse(m.passedExpiryDate(LocalDateTime.of(2019,12,30,0,0,0)));
}
```

이 테스트 코드는 실행 시점에 상관없이 항상 통과한다. 시간을 전달하면 경계 조건도 쉽게 테스트할 수 있다. 예를들어 0.001초만 지나도 만료로 처리하는지 확인하는 코드를 작성할 수 

있다.

시점을 제어하는 또 다른 방법은 별도의 시간 클래스를 작성하는 것이다.

```java
public class BizClock{
  private static BizClock DEFAULT = new BizClock();
  private static BizClock instance = DEFAULT;
  
  public static void reset(){
    instance = DEFAULT;
  }
  
  public static LocalDateTime now(){
    return instance.timeNow();
  }
  
  protected void setInstance(BizClock bizClock){
    BizClock.instance = bizClock;
  }
  
  public LocalDateTime timeNow(){
    return LocalDateTime.now();
  }
}
```

Member#isExpired()는 다음과 같이 BizClock 클래스를 이용해서 현재 시간을 구할 수 있다.

```java
public class Member{
  private LocalDateTime expiryDate;
  
  public boolean isExpired(){
    return expiryDate.isBefore(BizClock.now());
  }
}
```

BizClock 클래스의 setInstance() 메소드를 사용하면 instance정적 필드를 교체할 수 있으므로 BizClock을 상속받은 하위 클래스를 이용하면 BizClock#now() 가 원하는 시간을

제공하도록 만들 수 있다. 

```java
class TestBizClock extends BizClock{
  private LocalDateTime now;
  
  public TestBizClock(){
    setInstance(this);
  }
  public void setNow(LocalDateTime now){
    this.now = now;
  }
  @Override
  public LocalDateTime timeNow(){
    return now!=null ? now: super.now();
  }
}
```

TestBizClock 클래스를 이용하면 테스트 코드의 시간을 원하는 시점으로 제어할 수 있다.

```java
public class MemberTest{
  TestBizClock testClock = new TestBizClock();
  
  @AfterEach
  void resetClock(){
    BizClock.reset();
  }
  
  @Test
  void notExpired(){
    testClock.setNow(LocalDateTiome.of(2019,1,1,13,0,0));
    LocalDateTime expiry = LocalDateTime.of(2019,12,31,0,0,0);
    Member m =Member.builder().expiryDate(expiry).build();
    assertFalse(m.isExpired());
  }
}
```

### 랜덤하게 실패하지 않기

실행 시점에 따라 테스트가 실패하는 또 다른 예는 랜덤 값을 사용하는 것이다. 랜덤 값에 따라 달라지는 결과를 검증할 때 주로 이런 문제가 발생한다. 예를들어 숫자 야구 게임을

위한 Game클래스가 생성자에서 Random을 이용해서 숫자를 생성한다고 하자

```java
public class Game{
  private int[] nums;
  
  public Game(){
    Random random=new Random();
    int firstNo = random.nextInt(10);
    ...
    this.nums = new int[] {firstNo,secondNo, thridNo};
  }
  public Score guess(int ... answer){
    ...생략
  }
}
```

Game을 테스트하는 코드를 작성해보자. 아무것도 일치하지 않는 경우를 테스트하고 싶어도 테스트 코드를 작성할 수가 없다. 정답이 랜덤하게 만들어져서 어떤 숫자를 넣어야 일치

하지 않는지 미리 알 수 없기 때문이다.

```java
@Test
void noMatch(){
  Game g =new Game();
  Score s =g.guess(?,?,?); //테스트를 통과시킬 수 있는 값이 매번 바뀜
  assertEquals(0,s.strikes());
  assertEquals(0,s.balls());
}
```

랜덤하게 생성한 값이 결과 검증에 영향을 준다면 구조를 변경해야 테스트가 가능하다. Game같은 경우는 직접 랜덤 값을 생성하지 말고 생성자를 통해 값을 받도록 수정하면 테스트가 

간으해진다

```java
public class Game{
  private int[] nums;
  public Game(int[] nums){
    ...값 확인 코드
    this.nums=nums;
  }
}
```

또는 랜덤 값 생성을 다른 객체에 위임하게 바꿔도 된다. 예를들어 게임 숫자 생성을 위한 클래슬르 별도로 만든다

```java
public class GameNumGen{
  public int[] generate(){
    ...랜덤하게 값 생성
  }
}
```

이제 Game클래스는 GameNumGen을 이용해서 랜덤 값을 구하도록 변경한다

```java
public class Game{
  private int[] nums;
  public Game(gameNumGen gen){
    nums=gen.generate();
  }
}
```

테스트 코드는 GameNumGen의 대역을 사용해서 원하는 값을 갖고 Game클래스를 테스트할 수 있다.

```java
@Test
void noMatch(){
  //랜덤 값 생성을 별도 타입으로 분리하고
  //이를 대역으로 대체해서 대체
  GameNumGen gen=mock(GameNumGen.class);
  given(gen.generate()).willReturn(new int[]{1,2,3});
  
  Game g =new Game(gen);
  Score s =g.guess(4,5,6);
  assertEquals(0,s.strikes());
  assertEquals(0,s.balls());
}
```

## 필요하지 않은 값은 설정하지 않기

중복아이디를 가진 회원은 가입할 수 없다는 것을 검증하기 위한 테스트코드이다.

```java
@Test
void dupIdExists_Then_Exception(){
  //동일 ID가 존재하는 상황
  memoryRepository.save(
  	User.builder().id("dupid").name("이름")
  							.email("abc@abc.com")
  							.password("abcd")
  							.regDate(LocalDateTime.now())
  							.build());
  
  RegisterReq req=RegisterReq.builder()
    	.id("dupid").name("다른이름")
    	.email("dupid@abc.com")
    	.password("abcde")
    	.build()
    
   assertThrows(DupIdException.class,
       () -> userRegisterSvc.register(req));
}
```

이 테스트 코드를 잘못 만든 것은 아니짐나 검증할 내용에 비하면 필요하지 않은 값까지 설정하고 있다. 이 테스트는 동일 ID가 존재하는 경우에 가입할 수 없는지를 검증하는 것이

목적이기 때문에 동일 ID가 존재하는 상황을 만들 때 이메일, 이름, 가입일과 같은 값은 필요하지 않다. RegisterReq객체를 생성할 때에도 검증에 필요한 값만 지정하면 된다.

테스트할 범위에 필요한 값만 설정하ㅣ도록 변경한 코드는 다음과 같다

```java
@Test
void dupIdExists_Then_Exception(){
  //동일 ID가 존재하는 상황
  memoryRepository.save(User.builder().id("dupid").build());
  
  RegisterReq req= RegisterReq.builder()
    	.id("dupid")
    	.build();
  assertThrows(DupIdException.class,
        () -> userRegisterSvc.register(req));
}
```

테스트에 필요한 값만 설정하면 필요하지 않은 값을 설정하느라 고민할 필요가 없다. 또한, 테스트 코드가 짧아져서 한눈에 내용을 파악할 수 있다.

### 단위 테스트를 위한 객체 생성 보조 클래스

단위 테스트 코드를 작성하다 보면 상황 구성을 위해 필요한 데이터가 다소 복잡할 때가 있다. 예를 들어 설문에 답변하는 기능을 구현하기 위해 다음에 해당하는 설문이 존재하는

상황이 필요하다고 가정하자.

- 설문이 공개 상태임
- 설문 조사 기간이 끝나지 않음
- 설문 객관식 문항이 두 개임
- 각 객관식 문항의 보기가 두 개임

테스트 코드는 상황을 구성하기 위해 다음과 같은 코드를 사용해야 한다. null이면 안되는 필수 속성이 많다면 상황 구성 코드는 더 복잡해질 것이다.

```java
@Test
void answer(){
  memorySurveyRepository.save(
  	Survey.builder().id(1L)
  				.satatus(SurveyStatus.OPEN)
  				.endOfPeriod(LocalDateTime.now().plusDays(5))
    			.questions(asList(new Question(1,"질문1",asList(Item.of(1,"보기1"),Item.of(2,"보기2"))),
                        		new Question(1,"질문2", asList(Item.of(1,"답1"),Item.of(2,"답2")))
                           )
                    ).build();
  );
  
  ansewerService.answer(...생략);
}
```

테스트를 위한 생성 클래스를 따로 만들면 이런 복잡함을 줄일 수 있다. 다음은 테스트 코드에서 필요한 객체를 생성할 때 사용할 수 있는 팩토리 클래스의 예를 보여준다.

```java
public class TestSurveyFactory{
  public static Survey createAnswerableSurvey(Long id){
    Survey.builder().id(id)
  				.satatus(SurveyStatus.OPEN)
  				.endOfPeriod(LocalDateTime.now().plusDays(5))
    			.questions(asList(new Question(1,"질문1",asList(Item.of(1,"보기1"),Item.of(2,"보기2"))),
                        		new Question(1,"질문2", asList(Item.of(1,"답1"),Item.of(2,"답2")))
                           )
                    ).build();
  }
}
```

이제 답변 가능한 설문이 필요한 테스트 코드는 이 팩토리 클래슬르 이용해 간결하게 상황을 구성할 수 있다.

```java
@Test
void answer(){
  memorySurveyRepository.save(TestSurveyFactory.createAnswerableSurvey(1L));
    ansewerService.answer(...생략);
}
```

## 조건부로 검증하지 않기

테스트는 성공하거나 실패해야 한다. 테스트가 성공하거나 실패하렴녀 반드시 단언을 실행해야 한다. 만약 조건에 따라서 단언을 하지 않으면 그 테스트는 성공하지도 실패하지도 않은 

테스트가 된다. 이런 문제가 발생하는 예를 보자.

```java
@Test
void canTranslateBasicWord(){
  Translator tr=new Translator();
  if(tr.contains("cat"))
    assertEquals("고양이",tr.translate("cat"));
}
```

이 코드의 tr.contains("cat") 이 true가 아니면 단언을 실행하지 않는다. 만약 이 코드의 목적이 "cat"정도의 기본 단어는 번역을 할 수 있어야 한다는 것을 테스트하는 것이 목적이라면

이는 문제가 된다. 왜냐면 tr.contains("cat") 이 false를 리턴하면 테스트가 실패하지 않기 때문이다. 

이런 문제가 발생하지 않으려면 조건에 대한 단언도 추가해야한다. 

```java
@Test
void canTranslateBasicWorld(){
  Translator tr = new Translator();
  assertTranslationOfBasicWord(tr,"cat");
}

private void assertTranslationOfBasicWord(Translator tr, String word){
  assertTrue(tr.contains("cat"));
  assertEquals("고양이",tr.translate("cat"));
}
```

tr.translate("cat") 을 단언하기에 앞서 tr.contains("cat")이 true 인지 검사한다. 이렇게 함으로써 실패한 테스트를 놓치는 것을 방지할 수 있다.

## 통합 테스트는 필요하지 않은 범위까지 연동하지 않기

다음 코드를 보자

```java
@Component
public class MemberDao {
  private JdbcTemplate jdbcTemplate;
  
  public MemberDao(JdbcTemplate jdbcTemplate){
    this.jdbcTemplate= jdbcTemplate;
  }
  
  public List<Member> selectAll(){
    ...생략
  }
}
```

이 코드는 스프링의 JdbcTemplate을 이용해서 데이터를 연동하고 있다. 스프링 부트프로젝트를 사용한다면 다음과 같은 코드를 이용해서 DB연동 테스트를 진행할 수 있다.

```java
@SpringBootTest
public class MemberDaoIntTest{
  @Autowired
  MemberDao dao;
  
  @Test
  void findAll(){
    List<Member> members=dao.selectAll();
    assertTrue(members.size()>0);
  }
}
```

이 테스트코드는 잘못 만들지는 않았지만 한 가지 단점이 있다. 테스트하는 대상은 DB와 연동을 처리하는 MemberDao인데 @SpringBootTest 애노테이션을 사용하면 서비스,

컨트롤러 등 모든 스프링 빈을 초기화한다는 것이다. DB 관련된 설정 외에 나머지 설정도 처리하므로 스프링을 초기화하는 시간이 길어질 수 있다.

스프링 부트가 제공하는 @JdbcTest 애노테이션을 사용하면 DataSoruce, JdbcTemplate등 DB연동과 관련된 설정만 초기화 한다. 다른 빈을 생성하지 않으므로 스프링을 

초기화하는 시간이 짧아진다. 다음은 @JdbcTest 애노테이션을 이용예를 보여준다

```java
@JdbcTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
public class MemberDaoJdbcTest{
  @Autowired
  JdbcTemplate jdbcTemplate;
  
  private MemberDao dao;
  
  @BeforeEach
  void setUp(){
    dao = new MemberDao(jdbcTemplate);
  }
  @Test
  void findAll(){
    ...생략
  }
}
```

이 코드는 MemberDao객체를 직접 생성하고 있지만 대신 확인에 필요한 스프링 설정만 초기화하고 테스트할 수 있는 장점이 있다.

DataSource와 JdbcTemplate을 테스트 코드에서 직접 생성하면 스프링 초기화 과정이 빠지므로 테스트 시간은 더 짧아질 것이다.

## 더 이상 쓸모 없는 테스트 코드

LocalDateTime을 문자열로 변환하는 코드가 필요한데 LocalDateTime클래스를 사용한 경험이 없다고 하자.  이경우 테스트 코드를 사용해 LocalDateTime의 포멧팅

방법을 익힐 수 있다.

```java
@Test
void format(){
  LocalDateTime dt = LocalDateTime.of(2019,8,15,12,0,0);
  assertEquals("2019-08-15 12:00:00", dt.format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")));
}
```

비슷하게LocalDate를 사용할 때 2020년 1월 31일에서 한 달을 더함녀 2020년 2월 29일이 나오는지 확인하고 싶다면 다음 테스트 코드를 작성해서 확인할 수 있다.

```java
@Test
void diff(){
  LocalDate date = LocalDate.of(2020,1,31);
  assertEquals(LocalDate.of(2020,2,29), date.plusMonths(1));
}
```

이런 테스트 코드는 사용법을 익히고 나면 더이상 필요가 없다. 이렇게 특정 클래스의 사용법을 익히기 위해 작성한 테스트 코드는 오래 유지할 필요가 없으므로 삭제한다.

단지 테스트 커버리지를 높이기 위한 목적으로 작성한 테스트 코드도 유지할 필요가 없다. 코드 품질을 측정하는 수단으로 테스트 커버리지를 많이 사용하는데 이 수치를 높이기 위해 

다음과 같은 테스트코드를 작성할 때가 있다.

```java
@Test
void testGetter(){
  User user = new User(1L, "이름");
  assertEquals(1L,user.getId());
  assertEquals("이름",user.getName());
}
```

이 코드에서 User 클래스의 getId() 메소드나 getName() 메소드는 매우 단순해서 메소드를 검증할 목적으로 테스트 코드를 작성할 필요가 없다. 이 테스트코드는 단지 테스트 커버리지

를 높이기 위해서 존재할 뿐이다. 이런 테스트 코드 역시 실제 코드 유지보수에는 아무 도움이 되지 않으므로 삭제한다. 테스트 커버리지를 높여야 한다면 실제로 테스트 코드가 다루지

않는 if-else나 하위 타입등을 찾아 테스트를 추가해야한다. 그래야 의미있는 테스트 커버리지 측정값을 얻을 수 있다.

