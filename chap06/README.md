# 테스트 코드의 구성

## 기능에서의 상황

기능은 주어진 상황에 따라 다르게 동작한다. 예를 들어 다음 기능을 보자

- 파일에서 숫자를 읽어와 숫자의 합을 구한다
- 한 줄마다 한 개의 숫자를 포함한다

이 기능을 MathUtils.sum() 메소드로 구현한다고 가정하자. 다음처럼 sum() 메소드에 파일을 인자로 전달할 것이다. sum() 메소드는 인자로

전달받은 파일에서 한 줄씩 읽어와 숫자로 변환한 뒤에 합한 값을 결과로 제공하면 된다

```java
File datafile = new File("data.txt");
long sum = MathUtils.sum(dataFile);
```

하지만 이 기능을 구현하려면 고려할 것이 있다. 먼저 파일이 없는 상황을 처리해야 한다. 데이터를 읽을 파일이 없다면 인자로 잘못되었다는 익셉션을

발생하거나 문제 상황을 알려줄 수 있는 값을 리턴해야 한다. 비슷하게 데이터 중에 숫자가 아닌 잘못된 데이터가 존재하는 경우에도 알맞은 결과를 생성

해야 한다.

다음 예를 보자. 숫자 야구 게임은 0~9 사이의 서로 다른 숫자 세 개를  고르면 상대방이 그 숫자를 맞추는 게임이다. 예측한 숫자가 포함되어 있지만, 

위치가 다르면 볼이고 위치가 같으면 스트라이크이다. 어떤 숫자가 볼이고 스트라이크인지는 알려주지 않는다. 이 게임에서의 상황은 정답 숫자이다.

같은 예측이라도 정답 숫자에 따라 결과가 달라진다. 예를 들어 예측한 숫자가 456일 때 정답 숫자가 123인 경우와 456인 결과가 다르다.

```java
BaseballGame game1 = new BaseballGame("123");
Score score1 = game1.guess("456");
assertEquals(0,score1.ball());
assertEquals(0,score1.strikes());

BaseballGame game2 = new BaseballGame("456");
Score score2 = game2.guess("456");
assertEquals(3,score2.strikes());
```

## 테스트 코드의 구성 요소: 상황, 실행, 결과 확인

기능은 상황에 따라 결과가 달라진다. 테스트 코드는 기능을 실행하고 그 결과를 확인하므로 상황, 실행, 결과 확인의 세 가지 요소로 테스트를 구성할 수 

있다. 어떤 상황이 주어지고, 그 상황에서 기능을 실행하고, 실행한 결과를 확인하는 세 가지가 테스트 코드의 기본 골격을 이루게 된다.

JUnit에서 상황을 설정하는 방법은 테스트할 대상에 따라 달라진다. 숫자 야구 게임을 예로 들어보자. 숫자 야구 게임을 구현한 BaseballGame 

클래스는 객체를 생성할 때 정답 숫자를 지정한다. 이 경우 각 테스트 메소드마다 객체를 생성해서 상황을 설정할 수 있다.

```java
@Test
void exactMatch(){
  // 정답이 456인 상황
  BaseballGame game = new BaseballGame("456");
  //실행
  Score score = game.guess("456");
  //결과 확인
  assertEquals(3, score.strikes());
  assertEquals(0, score.balls());
}

@Test
void noMatch(){
  //정답이 123인 상황
  BaseballGame game = new BaseballGame("123");
  
  //실행
  Score score = game.guess("456");
  
  //결과 확인
  assertEquals(0, score.strikes());
  assertEquals(0, score.balls());
}
```

또 다른 방법은 @BeforeEach 를 적용한 메소드에서 상황을 설정하는 것이다. 이때는 주로 상황 설정과 꽌련된 대상을 필드로 보관한다

```java
private BaseballGame game;

@BeforeEach
void givenGame(){
  game = new BaseballGame("456");
}

@Test
void exactMatch(){
  // 실행
  Score score = game.guess("456");
  
  //결과 확인
  assertEquals(3, score.strikes());
  assertEquals(0, score.balls());
}
```

상황이 없는 경우도 존재한다. 2장의 암호 강도 측정예가 이에 해당하는데 암호 강도 측정의 경우 결과에 영향을 주는 상황이 존재하지 않으므로 

테스트는 다음처럼 기능을 실행하고 결과를 확인하는 코드만 포함하고 있다.

```java
@Test
void meetsAllCriteria_Then_Strong(){
  //실행
  PasswordStrengthMeter meter = new PasswordStrengthMeter();
  PasswordStrength result = meter.meter("ab12!@AB");
  
  //결과 확인
  assertEquals(PasswordStrength.STRONG, result);
}
```

실행 결과를 확인하는 쉬운 방법은 리턴 값을 사용하는 것이다. 위 코드에서도 암호 강도를 측정한 결과를 리턴 값으로 받아 이 값을 확인했다.

실행 결과가 항상 리턴 값으로 존재하는 것은 아니다. 실행 결과로 익셉션을 발생하는 것이 정상인 경우도 있다. 예를 들어 숫자 야구 게임 생성 기능의

테스트 코드는 정답 숫자에 동일한 숫자가 존재하면 게임 생성에 실패해야 한다. 이 경우 게임 생성 실패 결과를 표시하기 위해 BaseBallGame

생성자가 IllegalArgumentException을 발생시키도록 구현할 수 있다.

```java
@Test
void genGame_With_DupNumber_Then_Fail(){
  assertThrows(IllegalArgumentException.class,
              () -> new BaseballGame("110"));
}
```

## 외부 상황과 외부 결과

상황 설정이 테스트 대상으로 국한된 것은 아니다. 상황에는 외부 요인도 있다. 

```java
File datafile = new File("data.txt");
long sum = MathUtils.sum(dataFile);
```

MathUtils.sum() 메소드를 테스트하려면 파일이 존재하지 않는 상황에서의 결과도 확인해야 한다. 그렇다면 파일이 존재하지 않는 상황을 어떻게 만들

수 있을까? 가장 쉬운 방법은 존재 하지 않는 파일을 경로로 사용하는 것이다.

```java
@Test
void noDataFile_Then_Exception(){
  File dataFile = new File("badpath.txt");
  assertThrows(IllegalArgumentException.class,
              () -> MathUtils.sum(dataFile));
}
```

이 방법이 쉽긴 하지만 항상 테스트에 성공할 것이라는 보장은 없다. 우연이라도 해당 파일이 존재할 수 있기 때문이다. 테스트는 실행할 때마다 동일한

결과를 보장해야 하는데 우연에 의해 테스트 결과가 달라지면 동일한 결과를 보장할 수 없다. 이는 테스트를 신뢰할 수 없게 만들어 테스트 결과를 

무시하게 만드는 요인이 될 수 있다.

더욱 확실한 방법은 명시적으로 파일이 없는 상황을 만드는 것이다. 다음은 명시적으로 파일이 없는 상황을 만드는 예를 보여준다.

```java
@Test
void noDataFile_Then_Exception(){
  givenNoFile("badpath.txt");
  
  File dataFile = new File("badpath.txt");
  assertThrows(IllegalArgumentException.class , 
              () -> MathUtils.sum(datafile));
}

private void givenNoFile(String path){
  File file = new File(path);
  if(file.exists()){
    boolean deleted = file.delete();
    if(!deleted)
      throw new RuntimeException("fail givenNoFile: " + path);
  }
}
```

이 테스트에서 givenNoFile() 메소드는 해당 경로에 파일이 존재하는지 검사해서 존재할 경우 해당 파일을 삭제한다. 이렇게 함으로써 테스트가 항상

올바른 상황에서 동작한다는 것을 보장할 수 있다. 

다음으로 파일이 존재하는 상황은 어떻게 만들 수 있을까? 쉬운 방법은 상황에 알맞은 파일을 미리 만들어 두는 것이다. 예를 들면 다음 데이터를 갖는

"datafile.txt" 파일을 src/test/resource폴더에 미리 만드는 것이다.

```markdown
1
2
3
4
```

테스트 메소드는 다음 코드처럼 이 파일을 사용해서 상황에 맞는 테스트를 실행한다

```java
@Test
void dataFileSumTest(){
  File dataFile = new File("src/test/resources/datafile.txt");
  long sum = MathUtils.sum(dataFile);
  assertEquals(10L, sum);
}
```

다른 개발자도 테스트를 실행할 수 있어야 하므로 테스트에 맞게 준비한 파일은 버전관리 대상에 추가한다.

파일을 미리 만들지 않고 테스트 코드에서 상황에 맞는 파일을 생성하는 방법도 있다. 다음은 이방법을 사용한 예이다.

```java
@Test
void dataFileSumTest2(){
  givenDataFile("target/datafile.txt" , "1","2","3","4");
  File dataFile = new File("target/datafile.txt");
  long sum = MathUtils.sum(dataFile);
  assertEquals(10L , sum);
}

private void givenDataFile(String path, String...lines){
  try{
    Path dataPath = Paths.get(path);
    if(Files.exists(dataPath)){
      Files.delete(dataPath);
    }
    Files.write(dataPath,Arrays.asList(lines));
  }catch(IOException e){
    throw new RuntimeException(e);
  }
}
```

이 방법의 장점은 테스트 코드안에 필요한 것이 다 있다는 것이다. 테스트 코드에서 상황을 명시적으로 구성하기 때문에 테스트 내용을 이해하기 위해

많은 파일을 볼 필요가 없다.

테스트 대상이 아닌 외부에서 결과를 확인해야 할 때도 있다. 예를 들어 처리 결과를 지정한 경로의 파일에 저장하는 기능을 생각해보자. 이 기능을

실행한 결과를 검증하려면 해당 경로 파일에 파일이 원하는 내용으로 만들어 졌는지 확인해야 한다.

## 외부 상태가 테스트 결과에 영향을 주지 않게 하기

테스트 코드는 한 번만 실행하고 끝나지 않는다. TDD를 진행하는 동안에도 계속 실행하고 개발이 끝난 이후에도 반복적으로 테스트를 실행해서

문제가 없는지 검증한다. 그렇기 때문에 테스트는 언제 실행해도 항상 정상적으로 동작하는 것이 중요하다. 간헐적으로 실패하거나 다른 테스트

다음에 실행해야 성공하면 테스트 결과를 믿을 수 없게 된다. 이렇게 되면 테스트가 실패해도 무감각해지고 더 나아가 테스트를 만들지 않게 된다

회원 가입 기능을 예로 들어보자. 회원 가입 기능 테스트에는 다음을 포함한다

- 중복된 ID가 이미 존재하면 가입 실패
- 모든 조건을 충족하면 가입 성공

```java
@Test
void dupIdTest(){
  RegistReq req= new RegistReq("saechimdaeki","김준성중복");
  assertThrows(DuplicateIdException.class, () -> registerService.register(req));
}

@Test
void registerSuccessfully(){
  RegistReq req = new RegistReq("saechim","김준성");
  registerService.register(req);
  Member mem=memberRepo.findById("saechim");
  assertEquals("김준성",mem.getName());
}
```

dupIdTest() 테스트를 검증하려면 DB의 회원 테이블에 아이디가 "saechimdaeki" 인 데이터를 미리 추가해야 한다. 

아이디가 "saechim" 인 데이터가 없는 상태에서 registerSuccessfully( )테스트를 실행했더니 통과했다고 하자. 이 테스트에 성공하면 DB 회원

테이블에 아이디가 "saechim" 인 데이터가 생성된다. 이 상태에서 다시 registerSuccessfully() 테스트를 실행하면 아이디 중복으로 테스트에 실패한다.

이렇게 외부 상태에 따라 테스트의 성공 여부가 바뀌지 않으려면 테스트 실행 전에 외부를 원하는 상태로 만들거나 텍스트 실행 후에 외부 상태를

원래대로 되돌려 놓아야 한다.

## 외부 상태와 테스트 어려움

외부환경을 테스트에 맞게 구성하는 것은 항상 가능한 것은 아니다. 자동이체 등록 기능을 생각해보자. 이 기능은 입력받은 계좌번호가 올바른지 확인해야

한다. 이를 위해 금융회사에서 제공하는 REST API를 사용한다면 자동이체 등록 기능에 대한 테스트는 다음 상황에서의 결과를 확인 할수 있어야한다

- REST API 응답 결과가 유효한 계좌 번호인 상황
- REST API 응답 결과가 유효하지 않은 계좌번호인 상황
- REST API 서버에 연결할 수 없는 상황
- REST API 서버에서 응답을 5초 이내에 받지 못하는 상황

이렇게 테스트 대상의 상황과 결과에 외부 요인이 관여할 경우 대역을 사용하면 테스트 작성이 쉬워진다. 대역은 테스트 대상이 의존하는 대상의 실제 

구현을 대신하는 구현인데 이 대역을 통해서 외부 상황이나 결과를 대처할 수 있다.

대역에는 다양한 종류가 존재하는데 이는 다음장인 7장에서 알아보자.
