# Junit 5 기초

## Junit 5 모듈 구성

JUnit 5는 크게 세 개의 요소로 구성되어 있다.

- Junit 플랫폼: 테스팅 프레임워크를 구동하기 위한 런처와 테스트 엔진을 위한 API를 제공한다
- Junit 주피터 (Jupiter) : Junit 5를 위한 테스트 API와 실행 엔진을 제공한다
- Junit 빈티지 (Vintage): Junit 3과 4로 작성된 테스트를 Junit 5 플랫폼에서 실행하기 위한 모듈을 제공한다

Junit5는 테스트를 위한 API로 주피터 API를 제공한다. 주피터 API를 사용해서 테스트를 작성하고 실행하려면 주피터 관련 모듈을 의존에 추가하면 된다.

여기서 junit-jupiter 모듈은 unit-jupiter-api 모듈, junit-jupiter-params 모듈, junit-jupiter-engine 모듈을 포함한다. 이 세 모듈은 JUnit으로 테스트 코드를 작성하고

실행하기 위한 모듈이다.

```groovy
plugins {
    id 'java'
}

group 'org.example'
version '1.0-SNAPSHOT'

repositories {
    mavenCentral()
}

dependencies {
    testImplementation 'org.junit.jupiter:junit-jupiter-api:5.7.0'
    testRuntimeOnly 'org.junit.jupiter:junit-jupiter-engine:5.7.0'
}

test {
    useJUnitPlatform()
}
```

testImplementation을 사용해서 junit-jupiter 의존을 추가한다. junit-jupiter 모듈은 Unit 5.4 버전부터 제공한다. Junit 5.3버전이나 그 이하 버전을 사용한다면 필요한 모듈을

각각 설정해야한다.

## @Test 애노테이션과 테스트 메소드

JUnit 코드의 기본 구조는 간단하다. 테스트로 사용할 클래스를 만들고 @Test 애노테이션을 메소드에 붙이기만 하면 된다.

```java
public class SumTest {
    @Test
    void sum(){
        int result=2+3;
        assertEquals(5,result);
    }
}
```

테스트 클래스의 이름을 작성하는 특별한 규칙은 없지만 보통 다른 클래스와 구분을 쉽게 하기 위해 `Test` 를 접미사로 붙인다. 테스트를 실행할 메소드에는 @Test 애노테이션을

붙인다. 이때 @Test 애노테이션을 붙인 메소드는 private이면 안된다.

## 주요 단언 메소드

Assertions 클래스는 assertEquals() 를 포함해 아래의 표처럼 단언 메소드를 제공한다.

| 메소드                                          | 설명                                                         |
| ----------------------------------------------- | ------------------------------------------------------------ |
| assertEquals(expected, actual)                  | 실제 값(actual)이 기대하는 값(expected)과<br />같은지 검사한다 |
| assertNotEquals(unexpected, actual)             | 실제 값(actual)이 특정 값(unexpected) 과 <br />같지 않은지 검사한다 |
| assertSame(Object expected, Object actual)      | 두 객체가 동일한 객체인지 검사한다                           |
| assertNotSame(Object unexpected, Object actual) | 두 객체가 동일하지 않은 객체인지 검사한다                    |
| assertTrue(boolean condition)                   | 값이 true인지 검사한다                                       |
| assertFalse(boolean condition)                  | 값이 false인지 검사한다                                      |
| assertNull(Object actual)                       | 값이 null인지 검사한다                                       |
| assertNotNull(Object actual)                    | 값이 null이 아닌지 검사한다                                  |
| fail()                                          | 테스트를 실패 처리한다                                       |

주요 타입별로 assertEquals() 메소드가 존재한다. 예를 들어 int 타입을 위한 assertEquals() 메소드, Long 타입을 위한 assertEquals() 메소드, Object를 위한 assertEquals() 

메소드 등이 존재한다. assertEquals() 메소드를 사용할 때 주의할 점은 첫 번째 인자가 기대하는 값이고 두 번째 인자가 검사하려는 값이라는 점이다.

assertEquals(Object expected, Object actual) 메소드는 equals() 메소드를 이용해서 두 객체가 같은지 비교한다. LocalDate 클래스는 알맞게 equals() 메소드를 구현했으므로

다음과 같이 assertEquals() 메소드를 이용해서 두 객체가 같은 값을 갖는지 비교할 수 있다

```java
LocalDate dateTime1 = LocalDate.now();
LocalDate dateTime2 = LocalDate.now();
assertEquals(dateTime1, dateTime2);
```

fail() 메소드는 테스트에 실패했음을 알리고 싶을때 사용한다. 예를 들어 ID와 암호로 전달 받은 파라미터 값이 null이면 IllegalArgumentException이 발생하도록 인증 기능을 

구현 했다고 가정하자. 테스트 코드는 ID와 암호로 null을 전달했는데 익셉션이 발생하지 않으면 테스트에 실패했다고 볼 수 있다. 이럴때 fail()을 사용할 수 있다.

```java
try{
  AuthService authService = new AuthService();
  authService.authenticate(null,null);
  fail(); // 이 지점에 다다르면 fail() 메소드는 테스트 실패 에러를 발생
}catch(IllegalArgumentException e){
  
}
```

익셉션 발생 유무가 검증 대상이라면 fail() 메소드를 사용하는 것보다 아래 표의 두 메소드를 사용하는 것이 더 명시적이다.

| 메소드                                                       | 설명                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `assertThrows(Class<T> expectedType, Executable executable)` | executable을 실행한 결과로 지정한 타입의<br />익셉션이 발생하는지 검사한다 |
| `assertDoesNotThrow(Executable executable)`                  | executable을 실행한 결과로 익셉션이 발생하지<br />않는지 검사한다 |

다음은 assertThrows()를 이용해서 지정한 익셉션이 발생하는지 검사하는 코드 예이다.

```java
assertThrows(IllegalArgumentException.class,
            () -> {
              AuthService authService = new AuthService();
              authService.authenticate(null,null);
            });
```

assertThrows() 메소드는 발생한 익셉션 객체를 리턴한다. 발생한 익셉션을 이용해서 추가로 검증이 필요하면 assertThrows() 메소드가 리턴한 익셉션 객체를 사용하면 된다

```java
IllegalArgumentException thrown = assertThrows(IllegalArgumentException.class,
            () -> {
              AuthService authService = new AuthService();
              authService.authenticate(null,null);
            });
assertTrue(thrown.getMessage().contains("id"));
```

참고로 assertThrows()와 assertDoesNotThrow() 메소드에서 사용하는 Executable 인터페이스는 다음과 같이 execute() 메소드를 가진 함수형 인터페이스이다.

```java
public interface Executable{
  void execute() throws Throwable;
}
```

assert 메소드는 실패하면 다음 코드를 실행하지 않고 바로 익셉션을 발생한다. 예를 들어 다음 코드는 첫 번째 assertEquals() 메소드에서 검증에 실패하기 때문에 그 시점에 

AssertionFailedError를 발생한다. 따라서 두 번째 assertEquals() 메소드는 실행되지 않는다.

```java
assertEquals(3, 5/2); //검증 실패로 에러 발생
assertEquals(4, 2*2); // 이 코드는 실행되지 않음
```

그런데 경우에 따라 일단 모든 검증을 실행하고 그중에 실패한 것이 있는지 확인하고 싶을 때가 있다. 이럴 때 사용할 수 있는 것이 assertAll() 메소드이다. 다음은 assertAll() 메소드의

사용 예이다.

```java
assertAll(
	() -> assertEquals(3, 5/2);
  () -> assertEquals(4, 2*2);
  () -> assertEquals(6, 11/2);
)
```

assertAll() 메소드는 Executable 목록을 가변 인자로 전달받아 각 Executable을 실행한다. 실행 결과로 검증에 실패한 코드가 있으면 그 목록을 모아서 에러 메시지로 보여준다.

# 테스트 라이프 사이클

## @BeforeEach 애노테이션과 @AfterEach 애노테이션

JUnit은 각 테스트 메소드마다 다음 순서대로 코드를 실행한다.

1. 테스트 메소드를 포함한 객체 생성
2. (존재하면) @BeforeEach 애노테이션이 붙은 메소드 실행
3. @Test 애노테이션이 붙은 메소드 실행
4. (존재하면) @AfterEach 애노테이션이 붙은 메소드 실행

다음 코드는 동작 방식을 이해하기 위해 작성한 코드다.

```java
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

public class LifecycleTest {
    LifecycleTest(){
        System.out.println("new LifeCycleTest");
    }
    
    @BeforeEach
    void setUp(){
        System.out.println("setUp");
    }
    
    @Test
    void a(){
        System.out.println("A");
    }
    
    @Test
    void b(){
        System.out.println("B");
    }
    @AfterEach
    void tearDown(){
        System.out.println("tearDown");
    }
} 
```

위 테스트 클래스를 실행하면 콘솔에 다음 결과가 출력된다

```markdown
new LifecycleTest
setUp
A

tearDown
new LifecycleTest
setUp
B
tearDown
```

이 결과를 보면 @Test 메소드를 실행할 때마다 객체를 새로 생성하고 테스트 메소드를 실행 하기 전과 후에 @BeforeEach 애노테이션과 @AfterEach 애노테이션을 

붙인 메소드를 실행 한다는 것을 알 수 있다. @BeforeEach 애노테이션은 테스트를 실행하는데 필요한 준비 작업을 할 때 사용한다. @BeforeEach 애노테이션을 이용해서

테스트에서 사용할 임시 파일을 생성한다거나 테스트 메소드에서 사용할 객체를 생성한다

반대로 @AfterEach 애노테이션은 테스트를 실행한 후에 정리할 것이 있을때 사용한다. 테스트에서 사용한 임시 파일을 삭제해야 할 때 @AfterEach 애노테이션을 사용하면 된다

@BeforeEach 애노테이션과 @AfterEach 애노테이션을 붙인 메소드는 @Test 애노테이션과 마찬가지로 private면 안된다.

## @BeforeAll 애노테이션과 @AfterAll 애노테이션

한 클래스의 모든 테스트 메소드가 실행되기 전에 특정 작업을 수행해야 한다면 @BeforeAll 애노테이션을 사용한다. @BeforeAll 애노테이션은 정적 메소드에 붙이는데 이 메소드는

클래스의 모든 테스트 메소드를 실행하기 전에 한 번 실행된다

@AfterAll 애노테이션은 반대로 클래스의 모든 테스트 메소드를 실행한 뒤에 실행된다. 이 메소드 역시 정적 메소드에 적용한다.

# 테스트 메소드 간 실행 순서 의존과 필드 공유하지 않기

```java
public class BadTest{
  private FileOperator op = new FileOperator();
  private static File file; // 두 테스트가 데이터를 공유할 목적으로 필드 사용
  
  @Test
  void fileCreationTest(){
    File createdFile = op.createFile();
    assertTrue(createdFile.length()>0);
    this.file=createdFile;
  }
  
  @Test
  void readFileTest(){
    long data = op.readData(file);
    assertTrue(data > 0);
  }
}
```

이 코드는 file필드를 이용해서 fileCreationTest() 메소드에서 생성한 File을 보관하고 그 file필드를 readFileTest() 메소드에서 사용한다. 테스트 메소드를 실행할 때마다 객체를 

새로 생성하므로 file을 정적 필드로 정의했다. 이 테스트는 fileCreationTest() 메소드가 readFileTest() 메소드보다 먼저 실행된다는 것을 가정한다.

실제로 원하는 순서대로 테스트 메소드가 실행될 수도 있다. 하지만 테스트 메소드가 특정 순서대로 실행된다는 가정하에 테스트 메소드를 작성하면 안 된다.

Junit이 테스트 순서를 결정하긴 하지만 그 순서는 버전에 따라 달라질 수 있다. 순서가 달라지면 테스트도 실패한다. 예를 들어 readFileTest() 메소드가 먼저 실행되면

file필드가 null이므로 테스트에 실패하게 된다.

각 테스트 메소드는 서로 독립적으로 동작해야 한다. 한 테스트 메소드의 결과에 따라 다른 테스트 메소드의 실행 결과가 달라지면 안 된다. 그런 의미에서 테스트 메소드가 서로 필드를

공유한다거나 실행 순서를 가정하고 테스트를 작성하지 말아야 한다.

## 추가 애노테이션 : @DisplayName, @Disabled

테스트 실행결과를 보면 테스트 메소드이름을 사용해서 테스트 결과를 표시한다.

자바는 메소드 이름에 공백이나 특수문자를 사용할 수 없기 때문에 메소드 이름만으로 테스트 내용을 설명하기가 부족할 수 있다. 이럴 때는 @DisplayName 애노테이션을 사용해서

테스트에 표시 이름을 붙일 수 있다. 다음은 사용 예이다.

```java
@DisplayName("@DisplayName 테스트")
public class DisplayNameTest{
  @DisplayName("값 같은지 비교")
  @Test
  void assertEqualsMethod(){
    ... 생략
  }
  
  @Test
  void failMethod(){
    ...생략
  }
  
  @DisplayName("익셉션 발생 여부 테스트")
  @Test
  vodi assertThrowsTest(){
    ...생략
  }
  @Test
  void assertAllTest(){
    ...생략
  }
}
```

특정 테스트를 실행하지 않고 싶을 때는 @Disabled 애노테이션을 사용한다. Junit은 @Disabled 애노테이션이 붙은 클래스나 메소드는 테스트 실행 대상에서 제외한다.

아직 테스트코드가 완성되지 않았거나 잠시 동안 테스트를 실행하지 말아야 할 때 이 애노테이션을 사용한다. 다음은 사용 예이다

```java
public class AssertionsTest{
  @Disabled
  @Test
  void failMethod(){
    try{
      AuthService authService = new AuthService();
      authService.authenticate(null,null);
      fail();
    }catch(IllegalArgumentException e){
      
    }
  }
  ...생략
}
```

