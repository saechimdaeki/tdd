# Chapter 02- TDD시작

## TDD란?

TDD는 테스트부터 시작한다. 구현을 먼저하고 나중에 테스트하는 것이 아니라 먼저 테스트를 하고 그다음에 구현한다. 

구현 코드가 없는데 어떻게 테스트할 수 있을까? 여기서 테스트를 먼저 한다는 것은 기능이 올바르게 동작하는지 검증하는 테스트 코드를 먼저 작성한다는

것을 의미한다. 기능을 검증하는 테스트 코드를 먼저 작성하고 테스트를 통과시키기 위해 개발을 진행한다.



TDD로 개발할 때 먼저 해야할 것은 기능을 검증하는 테스트 코드를 작성하는것이다. 현실적인 기능을 TDD로 구현해보자. 구현할 기능은 암호검사기이다.

암호 검사기는 문자열을 검사해서 규칙을 준수하는지에 따라 암호를 '약함', '보통', '강함' 으로 구분한다. 

살펴볼 예제는 다음 규칙을 이용해서 암홀르 검사할 것이다

- 검사할 규칙은 다음 세가지이다
  - 길이가 8글자 이상
  - 0부터 9 사이의 숫자를 포함
  - 대문자 포함
- 세 규칙을 모두 충족하면 암호는 강함이다
- 2개의 규칙을 충족하면 암호는 보통이다
- 1개이하의 규칙을 충족하면 암호는 약함이다



이제 첫번째 텟그트 코드를 작성할 차례이다.

```java
public class PasswordStrengthMeterTest {
    @Test
    void name(){
        
    }
}
```

### 첫 번째 테스트: 모든 규칙을 충족하는 경우

첫번째 테스트를 선택할 때에는 가장 쉽거나 가장 예외적인 상황을 선택해야한다. 암호 검사 기능에서 가장 쉽거나 가장 예외적인 것은 무엇일까?

- 모든 규칙을 충족하는 경우
- 모든 조건을 충족하지 않는 경우

여기서 모든 규칙을 충족하는 경우를 작성해보자. 

```java
public class PasswordStrengthMeterTest {
    @Test
    void meetsAllCriteria_Then_Strong(){
        PasswordStrengthMeter meter=new PasswordStrengthMeter();
        PasswordStrength result = meter.meter("ab12!@AB");
        Assertions.assertEquals(PasswordStrength.STRONG,result);
    }
}
```

PasswordStrengthMeter타입과 PasswordStrength 타입이 존재하지 않으므로 컴파일에러가 발생한다. 먼저 컴파일 에러를 없애자. 

```java
public enum PasswordStrength {
    STRONG
}
```

```java
public class PasswordStrengthMeter {
    public PasswordStrength meter(String s){
        return null;
    }
}
```

컴파일 에러를 없앴으나 기대한 값은 STRONG인데 실제 값은 null이어서 테스트에 실패했음을 볼 수 있다.  이 테스트를 통과하는 방법은 간단한데

다음과 같이 수정을 하면 된다.

```java
public class PasswordStrengthMeter {
    public PasswordStrength meter(String s){
        return PasswordStrength.STRONG;
    }
}
```

## 두 번째 테스트: 길이만 8글자 미만이고 나머지 조건은 충족하는 경우

이번에 테스트할 대상은 패스워드 문자열의 길이가 8글자 미만이고 나머지 조건은 충족하는 암호이다. 이 암호의 강도는 보통이어야 한다.

```java
    @Test
    void meetsOtherCriteria_except_for_Length_Then_Normal(){
        PasswordStrengthMeter meter= new PasswordStrengthMeter();
        PasswordStrength result=meter.meter("ab12!@A");
        assertEquals(PasswordStrength.NORMAL,result);
    }
```

현재 PasswordStrength 열거 타입에 NORMAL이 없으므로 컴파일 에러를 NORMAL을 추가해서 없애자.

새로 추가한 테스트가 실패했다. 새로 추가한 테스트를 통과시키는 가장 쉬운 방법은 다음과 같이 NORMAL을 리턴하도록 수정하는 것이다.

```java
public PasswordStrength meter(String s){
        return PasswordStrength.NORMAL;
    }
```

그런데 이렇게하면 첫번째 테스트가 통과하지 못한다. 두 테스트를 모두 통과시킬 수 있는 만큼 코드를 작성해보자.

```java
public class PasswordStrengthMeter {
    public PasswordStrength meter(String s){
        if(s.length()<8){
            return PasswordStrength.NORMAL;
        }
        return PasswordStrength.STRONG;
    }
}
```

## 세 번째 테스트: 숫자를 포함하지 않고 나머지 조건은 충족하는 경우 

이번 테스트 대상은 숫자를 포함하지 않고 나머지 조건은 충족하는 암호이다. 이 암호도 보통 강도를 지녀야한다. 이를 위한 테스트 코드를 작성해보자.

```java
  @Test
    void meetsOtherCriteria_except_for_number_Then_Normal(){
        PasswordStrengthMeter meter = new PasswordStrengthMeter();
        PasswordStrength result = meter.meter("ab!@ABqwer");
        assertEquals(PasswordStrength.NORMAL,result);
    }
```

추가한 테스트가 실패하는 것을 돌려보면 알 수 있다. 이 테스트를 통과하는 방법은 어렵지 않다. 암호가 숫자를 포함했는지를 판단해서 

포함하지 않은 경우 NORMaL을 리턴하게 구현하면 된다. 구현코드는 다음과 같다.

```java
public class PasswordStrengthMeter {
    public PasswordStrength meter(String s){
        if(s.length()<8){
            return PasswordStrength.NORMAL;
        }
        
        boolean containsNum = meetsContainingNumberCriteria(s);
        if(!containsNum) return PasswordStrength.NORMAL;
        return PasswordStrength.STRONG;
    }

    private boolean meetsContainingNumberCriteria(String s) {
        for (char c : s.toCharArray()) {
            if(c>='0' && c <='9')
                return true;
        }
        return false;
    }
}
```

이제 테스트 코드 또한 리팩토링을 거쳐 다음과 같이 만들자.

```java
public class PasswordStrengthMeterTest {

    private PasswordStrengthMeter meter=new PasswordStrengthMeter();

    public void assertStrength(String password, PasswordStrength expStr){
        PasswordStrength result = this.meter.meter(password);
        assertEquals(expStr,result);
    }

    @Test
    void meetsAllCriteria_Then_Strong(){
        assertStrength("ab12!@AB",PasswordStrength.STRONG);
    }

    @Test
    void meetsOtherCriteria_except_for_Length_Then_Normal(){
        assertStrength("ab12!@A",PasswordStrength.NORMAL);
    }

    @Test
    void meetsOtherCriteria_except_for_number_Then_Normal(){
        assertStrength("ab!@ABqwer",PasswordStrength.NORMAL);
    }
}
```

## 네 번째 테스트: 값이 없는 경우

이런 예외 사항을 고려하지 않으면 우리가 만드는 소프트웨어는 비정상적으로 동작하게 된다. 예를 들어 meter() 메소드에 null을 전달하면 NPE이

발생하게 된다. 테스트를 추가해보자. null을 입력할 경우 암호 강도 측정기는 어떻게 반응해야 할까? 

- IllegalArgumentException을 발생한다
- 유효하지 않은 암호를 의미하는 PasswordStrength.INVALID를 리턴한다.

여기서는 두번째 방법을 선택하고 새로운 테스트를 작성해보자.

```java
 @Test
    void nullInput_Then_Invalid(){
        assertStrength(null,PasswordStrength.INVALID);
    }
```

마찬가지로 INVALID 열거형을 더하고 PasswordStrengthMeter에 다음과 같이 null처리 구현을 더하자.

```java
public class PasswordStrengthMeter {
    public PasswordStrength meter(String s){
        if(s==null) return PasswordStrength.INVALID;
        if(s.length()<8){
            return PasswordStrength.NORMAL;
        }

        boolean containsNum = meetsContainingNumberCriteria(s);
        if(!containsNum) return PasswordStrength.NORMAL;
        return PasswordStrength.STRONG;
    }

    private boolean meetsContainingNumberCriteria(String s) {
        for (char c : s.toCharArray()) {
            if(c>='0' && c <='9')
                return true;
        }
        return false;
    }
}
```

이제 테스트에 성공할 것이다. 하지만 예외사항이 null만 있는 것은 아니다. 빈 문자열도 예외사항이다. 빈 문자열에 대한 테스트도 추가한다.

```java
@Test
    void emptyInput_Then_Invalid(){
        assertStrength("",PasswordStrength.INVALID);
    }
```

이 테스트를 실행하면 기대한 값은 INVALID인데 실젯값은 NORMAL임을 알 수 있다. 따라서 이 테스트 통과를 위해 PasswordStrengthMeter를 변경하자

```java
public class PasswordStrengthMeter {
    public PasswordStrength meter(String s){
        if(s==null || s.isEmpty()) return PasswordStrength.INVALID;
        if(s.length()<8){
            return PasswordStrength.NORMAL;
        }

        boolean containsNum = meetsContainingNumberCriteria(s);
        if(!containsNum) return PasswordStrength.NORMAL;
        return PasswordStrength.STRONG;
    }

    private boolean meetsContainingNumberCriteria(String s) {
        for (char c : s.toCharArray()) {
            if(c>='0' && c <='9')
                return true;
        }
        return false;
    }
}
```

## 다섯 번째 테스트: 대문자를 포함하지 않고 나머지 조건을 충족하는 경우

다음 추가할 테스트는 대문자를 포함하지 않고 나머지 조건은 충족하는 경우이다. 먼저 테스트 코드를 다음과 같이 추가한다.

```java
		@Test
    void meetsOtherCriteria_except_for_Uppercase_Then_Normal(){
        assertStrength("ab12!@df",PasswordStrength.NORMAL);
    }
```

이제 이 테스트 코드를 통과시키기 위해 코드를 더해보자.

```java
public class PasswordStrengthMeter {
    public PasswordStrength meter(String s){
        if(s==null || s.isEmpty()) return PasswordStrength.INVALID;
        if(s.length()<8){
            return PasswordStrength.NORMAL;
        }

        boolean containsNum = meetsContainingNumberCriteria(s);
        if(!containsNum) return PasswordStrength.NORMAL;
        boolean containsUpp= false;
        for (char ch : s.toCharArray()) {
            if(Character.isUpperCase(ch)){
                containsUpp=true;
                break;
            }
        }
        if(!containsUpp) return PasswordStrength.NORMAL;
        return PasswordStrength.STRONG;
    }
  ...
```

meter메소드가 복잡해졌으니 메소드 추출을하여 다음과 같이 코드를 정리하자

```java
public class PasswordStrengthMeter {
    public PasswordStrength meter(String s){
        if(s==null || s.isEmpty()) return PasswordStrength.INVALID;
        if(s.length()<8){
            return PasswordStrength.NORMAL;
        }

        boolean containsNum = meetsContainingNumberCriteria(s);
        if(!containsNum) return PasswordStrength.NORMAL;
        boolean containsUpp = meetsContainingUppercaseCriteria(s);
        if(!containsUpp) return PasswordStrength.NORMAL;
        return PasswordStrength.STRONG;
    }

    private boolean meetsContainingUppercaseCriteria(String s) {
        for (char ch : s.toCharArray()) {
            if(Character.isUpperCase(ch)){
                return true;
            }
        }
        return false;
    }
  ...
```

## 여섯 번째 테스트: 길이가 8글자 이상인 조건만 충족하는 경우

길이가 8글자 이상인 조건만 충족하는 경우 암호 강도는 약함이다. 이를 위한 테스트코드를 추가하자.

```java
  @Test
    void meetsOnlyLengthCriteria_Then_Weak(){
        assertStrength("abdefghi",PasswordStrength.WEAK);
    }
```

열거형 타입에 WEAK를 더해도 이 테스트는 실패한다. 실제결과는 NORMAL이기 때문이다. 이 테스트를 통과시키려면 세 조건 중에서 길이 조건은

충족하고 나머지 두 조건은 충족하지 않았을때 WEAK를 리턴하도록 구현해야 한다. 이를 구현하려면 먼저 길이가 8 이상인지 여부를 뒤에서

확인할 수 있어야 한다. 이를 위해 길이가 8이상인지 여부를 담는 로컬 변수를 추가하자.

```java
public class PasswordStrengthMeter {
    public PasswordStrength meter(String s){
        if(s==null || s.isEmpty()) return PasswordStrength.INVALID;
        boolean lengthEnough = s.length()>=8;
        boolean containsNum = meetsContainingNumberCriteria(s);
        boolean containsUpp = meetsContainingUppercaseCriteria(s);

        if(!lengthEnough){
            return PasswordStrength.NORMAL;
        }
        if(!containsNum) return PasswordStrength.NORMAL;
        if(!containsUpp) return PasswordStrength.NORMAL;
        return PasswordStrength.STRONG;
    }
  ...
```

추가로 if절의 위치를 이동한 이유는 다음 두 로직을 구분해서 모으기 위함이다

- 개별 규칙을 검사하는 로직
- 규칙을 검사한 결과에 따라 암호 강도를 계산하는 로직

다시 테스트를 실행해보자. 새로추가한 테스트는 여전히 실패하나 기존에 작성한 테스트는 깨지지 않고 통과한다. 이제 새로 추가한 테스트도

통과시켜보자. 테스트를 통과시키기 위해 추가한 코드는 다음과 같다.

```java
public class PasswordStrengthMeter {
    public PasswordStrength meter(String s){
       ...
         // ---------------- 이부분 -------------
        if(lengthEnough && !containsNum && !containsUpp)
            return PasswordStrength.WEAK;
        
        if(!lengthEnough){
            return PasswordStrength.NORMAL;
        }
      ....
```

## 일곱 번째 테스트: 숫자 포함 조건만 충족하는 경우

다음 테스트는 숫자 포함 조건만 충족하는 경우이다. 이를 검증하기 위한 테스트 코드를 추가하자.

```java
 @Test
    void meetsOnlyNumCriteria_Then_Weak(){
        assertStrength("12345",PasswordStrength.WEAK);
    }
```

이 테스트는 실패하며 이 테스트를 통과시키는 방법은 간단하다.

```java
public class PasswordStrengthMeter {
    public PasswordStrength meter(String s){
      ...
        if(lengthEnough && !containsNum && !containsUpp)
            return PasswordStrength.WEAK;
        if(!lengthEnough && containsNum && !containsUpp)
            return PasswordStrength.WEAK;
        if(!lengthEnough){
            return PasswordStrength.NORMAL;
        }
      ...
```

## 여덟 번째 테스트: 대문자 포함 조건만 충족하는 경우

이번에는 대문자 포함조건만 충족하는 경우이다. 추가한 테스트 코드는 다음과 같다.

```java
 @Test
    void meetsOnlyUpperCriteria_Then_Weak(){
        assertStrength("ABZEF",PasswordStrength.WEAK);
    }
```

테스트를 실행하면 실패하는데 통과시키기위해 코드를 추가하자. 

```java
public class PasswordStrengthMeter {
    public PasswordStrength meter(String s){
   ...
        if(lengthEnough && !containsNum && !containsUpp)
            return PasswordStrength.WEAK;
        if(!lengthEnough && containsNum && !containsUpp)
            return PasswordStrength.WEAK;
        if(!lengthEnough&& !containsNum && containsUpp)
            return PasswordStrength.WEAK;
      ...  
```

## 코드 정리: meter() 메소드 리팩토링

지금까지 작성한 구현코드를 먼저 보자. 

```java
public class PasswordStrengthMeter {
    public PasswordStrength meter(String s){
        if(s==null || s.isEmpty()) return PasswordStrength.INVALID;
        boolean lengthEnough = s.length()>=8;
        boolean containsNum = meetsContainingNumberCriteria(s);
        boolean containsUpp = meetsContainingUppercaseCriteria(s);

        if(lengthEnough && !containsNum && !containsUpp)
            return PasswordStrength.WEAK;
        if(!lengthEnough && containsNum && !containsUpp)
            return PasswordStrength.WEAK;
        if(!lengthEnough&& !containsNum && containsUpp)
            return PasswordStrength.WEAK;

        if(!lengthEnough){
            return PasswordStrength.NORMAL;
        }
        if(!containsNum) return PasswordStrength.NORMAL;
        if(!containsUpp) return PasswordStrength.NORMAL;
        return PasswordStrength.STRONG;
    }

    private boolean meetsContainingUppercaseCriteria(String s) {
        for (char ch : s.toCharArray()) {
            if(Character.isUpperCase(ch)){
                return true;
            }
        }
        return false;
    }

    private boolean meetsContainingNumberCriteria(String s) {
        for (char c : s.toCharArray()) {
            if(c>='0' && c <='9')
                return true;
        }
        return false;
    }
}

```



```java
				if(lengthEnough && !containsNum && !containsUpp)
            return PasswordStrength.WEAK;
        if(!lengthEnough && containsNum && !containsUpp)
            return PasswordStrength.WEAK;
        if(!lengthEnough&& !containsNum && containsUpp)
            return PasswordStrength.WEAK;
```

이 코드는 세 조건 중에서 한 조건만 충족하는 경우 암호강도가 약하다는 것을 구현한 것이다. 

그렇다면 충족하는 조건 개수를 사용하면 어떨까?

```java
public class PasswordStrengthMeter {
    public PasswordStrength meter(String s){
        if(s==null || s.isEmpty()) return PasswordStrength.INVALID;
        
        int metCounts=0;
        boolean lengthEnough = s.length()>=8;
        if(lengthEnough) metCounts++;
        boolean containsNum = meetsContainingNumberCriteria(s);
        if(containsNum) metCounts++;
        boolean containsUpp = meetsContainingUppercaseCriteria(s);
        if(containsUpp) metCounts++;
        if(metCounts==1) return PasswordStrength.WEAK;
        if(!lengthEnough){
            return PasswordStrength.NORMAL;
        }
        if(!containsNum) return PasswordStrength.NORMAL;
        if(!containsUpp) return PasswordStrength.NORMAL;
        return PasswordStrength.STRONG;
    }
  ...
```

이제 다음 코드를 리팩토링 해보자. 아래 코드는 어떤의미일까?

```java
 				if(!lengthEnough){
            return PasswordStrength.NORMAL;
        }
        if(!containsNum) return PasswordStrength.NORMAL;
        if(!containsUpp) return PasswordStrength.NORMAL;
```

이 코드를 작성한 시점에 이 코드의 의도는 충족하는 조건이 두 개인 경우 암호강도가 보통이라는 규칙을 표현한 것이다. 즉 다음과같이 변경할 수 있다.

```java
  public PasswordStrength meter(String s){
        if(s==null || s.isEmpty()) return PasswordStrength.INVALID;

        int metCounts=0;
        boolean lengthEnough = s.length()>=8;
        if(lengthEnough) metCounts++;
        boolean containsNum = meetsContainingNumberCriteria(s);
        if(containsNum) metCounts++;
        boolean containsUpp = meetsContainingUppercaseCriteria(s);
        if(containsUpp) metCounts++;
        if(metCounts==1) return PasswordStrength.WEAK;
        if(metCounts==2) return PasswordStrength.NORMAL;
        return PasswordStrength.STRONG;
    }
...
```

암호 강도는 이제 metCounts 값을 이용해서 계산한다. 이제 lengthEnough, containsNum, containsUpp 변수는 metCounts 값을 증가시킬 때만

사용된다. 따라서 다음과 같이 코드를 바꾸면 이 세 변수도 제거할 수 있다.

```java
public class PasswordStrengthMeter {
    public PasswordStrength meter(String s){
        if(s==null || s.isEmpty()) return PasswordStrength.INVALID;

        int metCounts=0;
        if(s.length()>=8) metCounts++;
        if( meetsContainingNumberCriteria(s)) metCounts++;
        if(meetsContainingUppercaseCriteria(s)) metCounts++;
        if(metCounts==1) return PasswordStrength.WEAK;
        if(metCounts==2) return PasswordStrength.NORMAL;
        return PasswordStrength.STRONG;
    }
  ...
```

## 아홉 번째 테스트: 아무 조건도 충족하지 않은 경우

아무 조건도 충족하지 않는경우 이를 검증하기 위한 테스트를 다음과 같이 추가하자.

```java
		@Test
    void meetsNoCriteria_Then_Weak(){
        assertStrength("abc",PasswordStrength.WEAK);
    }
```

테스트를 실행하면 원하는 결과는  WEAK인데 실제 결과는 STRONG이다. 

이 테스트를 통과시키려면 다음 중 한가지 방법을 사용하면 된다.

- 충족 개수가 1개 이하인 경우  WEAK를 리턴하도록 수정
- 충족 개수가 0개인 경우 WEAK를 리턴하는 코드 수정
- 충족 개수가 3개인 경우 STRONG을 리턴하는 코드를 추가하고 마지막에  WEAK를 리턴하도록 코드 수정

여기서는 첫번째 방법을 사용하자. 



````java
public class PasswordStrengthMeter {
    public PasswordStrength meter(String s){
        if(s==null || s.isEmpty()) return PasswordStrength.INVALID;

        int metCounts=0;
        if(s.length()>=8) metCounts++;
        if( meetsContainingNumberCriteria(s)) metCounts++;
        if(meetsContainingUppercaseCriteria(s)) metCounts++;
        if(metCounts<=1) return PasswordStrength.WEAK;
        if(metCounts<=2) return PasswordStrength.NORMAL;
        return PasswordStrength.STRONG;
    }
  ...
````

## 코드 정리: 코드 가독성 개선

코드정리를 좀더하자. 이번 변경 대상은 metCounts 변수다. 이부분을 메소드로 빼면 meter()의 가독성을 조금더 높일 수 있을 것이다. 리팩토링 결과는

다음과 같다.

```java
public class PasswordStrengthMeter {
    public PasswordStrength meter(String s){
        if(s==null || s.isEmpty()) return PasswordStrength.INVALID;

        int metCounts=getMetCriteriaCounts(s);
        if(metCounts<=1) return PasswordStrength.WEAK;
        if(metCounts<=2) return PasswordStrength.NORMAL;
        return PasswordStrength.STRONG;
    }

    private int getMetCriteriaCounts(String s) {
        int metCounts=0;
        if(s.length()>=8) metCounts++;
        if( meetsContainingNumberCriteria(s)) metCounts++;
        if(meetsContainingUppercaseCriteria(s)) metCounts++;
        return metCounts;
    }
  
  ...
```

meter() 메소드의 가독성이 좋아졌다. meter() 메소드를 처음보는 개발자도 다음과 같이 코드를 읽을 수 있다.

-  암호가 null이거나 빈 문자열이면 암호 강도는  INVALID다.
- 충족하는 규칙 개수를 구한다
- 충족하는 규칙 개수가 1개 이하면 암호강도는  WEAK다
- 충족하는 규칙 개수가 2개 이하면 암호 강도는 NORMAL이다
- 이 외 경우 (충족하는 규칙 개수가 2개보다 크면) 암호 강도는 STRONG이다.

## TDD흐름

![image](https://user-images.githubusercontent.com/40031858/134805737-00c1d982-43cb-4db9-8c7f-285d642b2ea8.png)

TDD는 기능을 검증하는 테스트를 먼저 작성한다. 작성한 테스트를 통과하지 못하면 테스트를 통과할 만큼만 코드를 작성한다. 테스트를 통과한 뒤에는

개선할 코드가 있으면 리팩토링 한다. 리팩토링을 수행한 뒤에는 다시 테스트를 실행해서 기존 기능이 망가지지 않았는지 확인한다. 

이과정을 반복하면서 점진적으로 기능을 완성해 나가는것, 이것이 전형적인 TDD의 흐름이다.

```markdown
# 레드- 그린 - 리팩터

### TDD 사이클을 레드-그린-리팩터로 부르기도 한다. 여기서 레드는 실패하는 테스트를 의미한다. 레드는 테스트 코드가 실패하면
### 빨간색을 이용해서 실패한 테스트를 보여주는 데서 비롯했다. 비슷하게 그린은 성공한 테스트를 의미한다
### 즉 코드를 구현해서 실패하는 테스트를 통과시키는 것을 뜻한다. 마지막으로 리팩터는 이름 그대로 리팩토링 과정을 의미한다.
```

### 테스트가 개발을 주도

테스트 코드를 먼저 작성하면 테스트가 개발을 주도하게 된다. 앞서 경험을 보면 가장 먼저 통과해야 할 테스트를 작성했다. 테스트를 작성하는 과정에서 

구현을 생각하지 않았다. 단지 해당 기능이 올바르게 검증할 수 있는 테스트코드를 만들었을 뿐이다.



테스트를 추가한 뒤에는 테스트를 통과시킬 만큼 기능을 구현했다. 지금까지 작성한 테스트를 통과할 만큼만 구현을 진행했다. 아직 추가하지 않은 테스트를

고려해서 구현하지 않았다. 예를 들어 값이 없는 경우에 대한 테스트를 추가하고 개발을 진행할 때에는 그때까지 추가한 테스트를 통과할 만큼의 코드만 구현했다.

미리 앞서 한 조건만 충족하는 암호를 측정하는 기능을 구현하지 않았다. 테스트 코드를 만들면 다음 개발 범위가 정해진다. 테스트 코드가 추가되면서

검증하는 범위가 넓어질수록 구현도 점점 완성되어간다. 이렇게 테스트가 개발을 주도해 간다

### 지속적인 코드 정리

구현을 완료한 뒤에는 리팩토링을 진행했다. 리팩토링할 대상이 눈에 들어오면 리팩토링을 진행해서 코드를 정리했다. 당장 리팩토링할 대상이나 어떻게

리팩토링해야 할지 생각나지 않으면 다음 테스트를 진행했다. 테스트 코드 자체도 리팩토링 대상에 넣었다.

TDD는 개발 과정에서 지속적으로 코드 정리를 하므로 코드 품질이 급격히 나빠지지 않게 막아 주는 효과가 있다. 이는 향후 유지보수 비용을 낮추는데

기여한다.

### 빠른 피드백

TDD가 주는 이점은 코드 수정에 대한 피드백이 빠르다는 점이다. 새로운 코드를 추가하거나 기존 코드를 수정하면 테스트를 돌려서 해당 코드가 

올바른지 바로 확인할 수 있다. 이는 잘못된 코드가 배포되는 것을 방지한다.
