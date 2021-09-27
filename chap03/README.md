# 테스트 코드 작성 순서

2장에서는 암호 강도 측정 기능을 TDD로 구현했다. 기능을 구현할 때 테스트 코드를 작성한 순서는 다음과 같았다

1. 모든 규칙을 충족하는 암호 강도는 '강함'
2. 길이만 8글자 미만이고 나머지 규칙은 충족하는 암호의 강도는 '보통'
3. 숫자를 포함하지 않고 나머지 규칙은 충족하는 암호의 강도는 '보통'
4. 값이 없는 암호의 강도는 '유효하지 않음'
5. 대문자를 포함하지 않고 나머지 규칙은 충족하는 경우
6. 길이가 8글자 이상인 규칙만 충족하는 경우
7. 숫자 포함 규칙만 충족하는 경우
8. 대문자 포함 규칙만 충족하는 경우
9. 아무 규칙도 충족하지 않는 경우

이 순서가 그냥 나온것은 아니다. 실제로 이순서는 다음 규칙을 따랐다

- 쉬운 경우에서 어려운 경우로 진행
- 예외적인 경우에서 정상인 경우로 진행

초반에 복잡한 테스트부터 시작하면 어려운 이유는 보통의 개발자는 한번에 많은 코드를 만들다 보면 나도 모르게 버그를 만들고 나중에 버그를 잡기

위해 많은 시간을 허비하게 된다. 당연히 테스트 통과 시간도 길어진다. 그뿐만 아니라 코드 작성 시간이 길어지면 집중력도 떨어져서 흐름이 자주 끊기게

된다. 가장 구현하기 쉬운 경우부터 시작하면 빠르게 테스트를 통과시킬 수 있다.

또한 예외 상황을 먼저 테스트해야하는 이유가 있는데 다양한 예외상황은 복잡한 if-else 블록을 동반할 때가 많다. 예외 상황을 전혀 고려하지 않은

코드에 예외 상황을 반영하려면 코드의 구조를 뒤집거나 코드 중간에 예외 상황을 처리하기 위해 조건문을 중복해서 추가하는 일이 벌어진다. 

이는 코드를 복잡하게 만들어 버그 발생 가능성을 높인다. 초반에 예외 상황을 테스트하면 이런 가능성이 줄어든다. 예외 상황에 따른 if-else 구조가

미리 만들어지기 때문에 많은 코드를 완성한 뒤에 예외 상황을 반영할 때보다 코드 구조가 덜 바뀐다.

# 테스트 작성 순서 연습

매달 비용을 지불해야 사용할 수 있는 유료 서비스가 있다고 해보자. 이 서비스는 다음 규칙에 따라 서비스 만료일을 결정한다.

- 서비스를 사용하려면 매달 1만원을 선불로 납부한다. 납부일 기준으로 한달뒤가 서비스 만료일이 된다.
- 2개월 이상 요금을 납부할 수 있다
- 10만원을 납부하면 서비스를 1년 제공한다

먼저 만료일을 계산하는 테스트 클래스를 만들자

```java
public class ExpiryDateCalculatorTest {
    
}
```

## 쉬운 것부터 테스트

이제 테스트 메소드를 추가하자. 테스트를 추가할 때는 다음 두가지를 고려한다

- 구현하기 쉬운 것부터 먼저 테스트
- 예외 상황을 먼저 테스트

만료일 계산기에서는 1만 원을 납부하면 한 달 뒤 같은 날을 만료일로 계산하는 것이 가장 쉬울 것같다. 계산에 필요한 값은 납부일과 납부액이고 결과는

계산된 만료일이다. 아래는 이를 테스트코드로 표현한 것이다.

```java
		@Test
    void 만원_납부하면_한달_뒤가_만료일임(){
        LocalDate billingDate= LocalDate.of(2019,3,1);
        int payAmount=10_000;
        
        ExpiryDateCalculator cal=new ExpiryDateCalculator();
        LocalDate expiryDate=cal.calculateExpiryDate(billingDate,payAmount);

        assertEquals(LocalDate.of(2019,4,1),expiryDate);
    }
```

이제 이 테스트를 통과시키려면 ExpiryDateCalculator#calculateExpiryDate() 메소드가 2019-04-01에 해당하는 LocalDate를 리턴하면 된다

```java
public class ExpiryDateCalculator {
    public LocalDate calculateExpiryDate(LocalDate billingDate, int payAmount) {
        return LocalDate.of(2019,4,1);
    }
}
```

## 예를 추가하면서 구현을 일반화

이제 동일 조건의 예를 추가하면서 구현을 일반화해보자. 먼저 1만원을 납부 하는 예를 하나 더 추가한다. 이 예는 2019-03-01 대신 2019-05-05를 

납부일로 사용한다. 따라서 만료일은 2019-06-05이어야 한다. 이를 검증하기 위한 테스트 코드를 추가하자.

```java
@Test
    void 만원_납부하면_한달_뒤가_만료일임(){
        LocalDate billingDate= LocalDate.of(2019,3,1);
        int payAmount=10_000;

        ExpiryDateCalculator cal=new ExpiryDateCalculator();
        LocalDate expiryDate=cal.calculateExpiryDate(billingDate,payAmount);

        assertEquals(LocalDate.of(2019,4,1),expiryDate);

        LocalDate billingDate2 = LocalDate.of(2019, 5, 5);
        int payAmount2=10_000;
        
        ExpiryDateCalculator cal2=new ExpiryDateCalculator();
        LocalDate expiryDate2 = cal2.calculateExpiryDate(billingDate2, payAmount2);
        assertEquals(LocalDate.of(2019,6,5),expiryDate2);
    }
```

이제 이 테스트 통과를 위해 간단한 테스트이니 만큼 바로 구현을 일반화 하자.

```java
public class ExpiryDateCalculator {
    public LocalDate calculateExpiryDate(LocalDate billingDate, int payAmount) {
        return billingDate.plusMonths(1);
    }
}
```

## 코드 정리: 중복 제거

ExpiryDateCalculator 클래스부터 보자. calculateExpiryDate()메소드는 파라미터가 두개이다. 아직 파라미터가 더 추가될지 알 수 없으므로 

발생하지도 않았는데 단정지어 코드를 수정할 필요는 없다.

테스트에 정리할 코드는 없을까? 테스트 메소드에는 다음 형태의 중복이 존재한다.

```java
LocalDate billingDate = 납부일;
int paymentAmount = 납부액;

ExpiryDateCalculator cal = new ExpiryDateCalculator();
LocalDate expiryDate=cal.calculateExpiryDate(billingDate,payAmount);

assertEquals(기댓값,expiryDate);
```

이제 중복을 제거하고 테스트 코드가 여전히 자신을 설명하고 있는지 확인해보자. 

```java
public class ExpiryDateCalculatorTest {

    @Test
    void 만원_납부하면_한달_뒤가_만료일임(){

        assertExpiryDate(LocalDate.of(2019,3,1),10_000,LocalDate.of(2019,4,1));

        assertExpiryDate(LocalDate.of(2019,5,5),10_000,LocalDate.of(2019,6,5));

    }

    private void assertExpiryDate(LocalDate billingDate, int payAmount, LocalDate expectedExpiryDate){

        ExpiryDateCalculator cal=new ExpiryDateCalculator();
        LocalDate realExpiryDate=cal.calculateExpiryDate(billingDate,payAmount);
        assertEquals(expectedExpiryDate,realExpiryDate);
    }
}
```

## 예외 상황 처리

이제 예외 상황을 찾아보자. 단순히 한 달 추가로 끝나지 않는 상황이 존재한다. 예를 들어 다음이 그런 예외상황에 해당한다

- 납부일이 2019-01-31이고 납부액이 1만원이면 만료일은 2019-02-28이다
- 납부일이 2019-05-31이고 납부액이 1만원이면 만료일은 2019-06-30이다.
- 납부일이 2020-01-31이고 납부액이 1만원이면 만료일은 2020-02-29이다.

이 세가지 조건은 납부일 기준으로 다음달의 같은날이 만료일이 아니다. 이를 테스트로 추가해야한다. 이것들을 테스트에 추가하자

```java
    @Test
    void 납부일과_한달_뒤_일자가_같지_않음(){
        assertExpiryDate(LocalDate.of(2019,1,31),10_000,LocalDate.of(2019,2,28));
        assertExpiryDate(LocalDate.of(2019,5,31),10_000,LocalDate.of(2019,6,30));
        assertExpiryDate(LocalDate.of(2020,1,31),10_000,LocalDate.of(2020,2,29));
    }
```

이 테스트들은 모두 통과하는데 LocalDate#plusMonths() 메소드가 알아서 한달 추가 처리를 해준다

## 다음 테스트 선택: 다시 예외 상황

다음 테스트를 선택하자. 그다음으로 쉽거나 예외적인 것을 선택하면 된다.

- 2만원을 지불하면 만료일이 두 달 뒤가 된다
- 3만원을 지불하면 만료일이 세 달 뒤가 된다

다음은 생각할 수 있는 예외 상황이다

- 첫 납부일이 2019-01-31이고 만료되는 2019-02-28에 1만원을 납부하면 다음 만료일은 2019-03-31이다
- 첫 납부일이 2019-01-30이고 만료되는 2019-02-28에 1만원을 납부하면 다음 만료일은 2019-03-30이다
- 첫 납부일이 2019-05-31이고 만료되는 2019-06-30에 1만원을 납부하면 다음 만료일은 2019-07-31이다.

이전 테스트가 1개월 요금 지불을 기준으로 했으므로 1개월 요금 지불에 대한 예외상항을 마무리하는 방향이 좋을 것같다 .

## 다음 테스트를 추가하기 전에 리팩토링

만료일을 계산하는데 필요한 값이 세 개로 늘었다. 다음을 고민할 때가 됐다

- calculateExpiryDate 메소드의 파라미터로 첫 납부일 추가
- 첫 납부일, 납부일, 납부액을 담은 객체를 calculateExpiryDate 메소드에 전달

첫 납부일을 파라미터로 추가하면 파라미터가 세 개로 늘어난다. 파라미터 개수는 적을 수록 코드 가독성과 유지보수에 유리하므로 메소드의 파라미터 

개수가 세 개 이상이면 객체로 바꿔 한 개로 줄이는 것을 고려해야 한다. 리팩토링을 진행하고 나면 ExpiryDateCalculator코드는 다음과 같이 바뀐다.

```java
public class ExpiryDateCalculator {
    public LocalDate calculateExpiryDate(PayData payData) {
        return payData.getBillingDate().plusMonths(1);
    }
}
```

```java
public class PayData {
    private LocalDate billingDate;
    private int payAmount;

    public PayData() {
    }

    public PayData(LocalDate billingDate, int payAmount) {
        this.billingDate = billingDate;
        this.payAmount = payAmount;
    }

    public LocalDate getBillingDate() {
        return billingDate;
    }

    public int getPayAmount() {
        return payAmount;
    }
    public static Builder builder(){
        return new Builder();
    }

    public static class Builder{
        private PayData data = new PayData();

        public Builder billingDate(LocalDate billingDate){
            data.billingDate=billingDate;
            return this;
        }

        public Builder payAmount(int payAmount){
            data.payAmount=payAmount;
            return this;
        }
        public PayData build(){
            return data;
        }
    }
}
```

이 공부의 주된 목적은 리팩토링이 아니므로 과정을 생략하고 진행한다. 

```java
public class ExpiryDateCalculatorTest {

    @Test
    void 만원_납부하면_한달_뒤가_만료일임(){

        assertExpiryDate(PayData.builder()
                .billingDate(LocalDate.of(2019,3,1))
                .payAmount(10_000)
                .build(), LocalDate.of(2019,4,1));

        assertExpiryDate(PayData.builder()
                .billingDate(LocalDate.of(2019,5,5))
                .payAmount(10_000)
                .build(), LocalDate.of(2019,6,5));

    }

    @Test
    void 납부일과_한달_뒤_일자가_같지_않음(){
        assertExpiryDate(PayData.builder()
                .billingDate(LocalDate.of(2019,1,31))
                .payAmount(10_000)
                .build(), LocalDate.of(2019,2,28));
        assertExpiryDate(PayData.builder()
                .billingDate(LocalDate.of(2019,5,31))
                .payAmount(10_000)
                .build(), LocalDate.of(2019,6,30));
        assertExpiryDate(PayData.builder()
                .billingDate(LocalDate.of(2020,1,31))
                .payAmount(10_000)
                .build(), LocalDate.of(2020,2,29));
    }

    private void assertExpiryDate(PayData payData, LocalDate expectedExpiryDate){

        ExpiryDateCalculator cal=new ExpiryDateCalculator();
        LocalDate realExpiryDate=cal.calculateExpiryDate(payData);
        assertEquals(expectedExpiryDate,realExpiryDate);
    }
}

```

## 예외 상황 테스트 진행 

앞서 테스트하려던 다음 사례를 테스트로 추가하자

- 첫 납부일이 2019-01-31이고 만료되는 2019-02-28에 1만원을 납부하면 당므 만료일은 2019-03-31이다.

```java
 @Test
    void 첫_납부일과_만료일_일자가_다를때_만원_납부(){
        PayData payData=PayData.builder()
                .firstBillingDate(LocalDate.of(2019,1,31))
                .billingDate(LocalDate.of(2019,2,28))
                .payAmount(10_000)
                .build();
        assertExpiryDate(payData,LocalDate.of(2019,3,31));
    }
```

첫 납부일을 전달해야하므로 PayData.Builder#firstBillingDate() 메소드를 사용하는 코드를 작성했다. PayData 컴파일 에러를 고치기 위해 코드를 추가하자

```java
public class PayData {
    private LocalDate firstBillingDate;
    private LocalDate billingDate;
    private int payAmount;

    public PayData() {
    }

    public PayData(LocalDate firstBillingDate, LocalDate billingDate, int payAmount) {
        this.firstBillingDate=firstBillingDate;
        this.billingDate = billingDate;
        this.payAmount = payAmount;
    }

    public LocalDate getFirstBillingDate() {
        return firstBillingDate;
    }

    public LocalDate getBillingDate() {
        return billingDate;
    }

    public int getPayAmount() {
        return payAmount;
    }
    public static Builder builder(){
        return new Builder();
    }

    public static class Builder{
        private PayData data = new PayData();

        public Builder firstBillingDate(LocalDate firstBillingDate){
            data.firstBillingDate=firstBillingDate;
            return this;
        }

       ...
}
```

이제 첫 납부일을 전달할 수 있게 되었다. 테스트를 실행하면 띠용?! 테스트가 실패한다. 일단 상수를 이용해서 테스트를 통과시키자.

```java
public class ExpiryDateCalculator {
    public LocalDate calculateExpiryDate(PayData payData) {
        if(payData.getFirstBillingDate().equals(LocalDate.of(2019,1,31)))
            return LocalDate.of(2019,3,31);
        return payData.getBillingDate().plusMonths(1);
    }
}
```

이제 새로 추가한 테스트는 통과했으나 앞서 작성한 두  테스트가 실패한다. NullPointException이 발생한다. 

```java
public class ExpiryDateCalculator {
    public LocalDate calculateExpiryDate(PayData payData) {
        if(payData.getFirstBillingDate()!=null)
            if(payData.getFirstBillingDate().equals(LocalDate.of(2019,1,31)))
                return LocalDate.of(2019,3,31);
        return payData.getBillingDate().plusMonths(1);
    }
}
```

상수를 이용해서 테스트를 통과시켰으니 새로운 테스트 사례를 추가해서 구현을 일반화할 차례다. 다음 추가할 사례는 다음과 같다

- 첫 납부일이 2019-01-30이고 만료되는 2019-02-28에 1만원을 납부하면 다음 만료일은 2019-03-30이다.

이 사례를 검증하기 위한 테스트코드를 다음과 같이 추가하자.

```java
 @Test
    void 첫_납부일과_만료일_일자가_다를때_만원_납부(){
       ...
        PayData payData2 = PayData.builder()
                .firstBillingDate(LocalDate.of(2019, 1, 30))
                .billingDate(LocalDate.of(2019, 2, 28))
                .payAmount(10_000)
                .build();

        assertExpiryDate(payData2,LocalDate.of(2019,3,30));
    }
```

테스트가 실패가 뜨니 통과할 만큼만 구현을 일반화해보자. 다음 로직을 사용하면 될 것 같다.

```java
public class ExpiryDateCalculator {
    public LocalDate calculateExpiryDate(PayData payData) {
        if(payData.getFirstBillingDate()!=null) {
            LocalDate candidateExp = payData.getBillingDate().plusMonths(1);
            if(payData.getFirstBillingDate().getDayOfMonth()!= candidateExp.getDayOfMonth()){
                return candidateExp.withDayOfMonth(payData.getFirstBillingDate().getDayOfMonth());
            }
        }
        
        return payData.getBillingDate().plusMonths(1);
    }
}
```

## 코드 정리: 상수를 변수로

ExpiryDateCalculator 코드를 보면 payData.getBillingDate().plusMonths(1) 코드에서 상수 1을 사용했다. 이 1은 만료일을 계산할 때 추가할 개월

수를 의미한다. 상수 1을 다음과 같이 변수로 바꾸자.

```java
public class ExpiryDateCalculator {
    public LocalDate calculateExpiryDate(PayData payData) {
        int addedMonths =1;
        if(payData.getFirstBillingDate()!=null) {
            LocalDate candidateExp = payData.getBillingDate().plusMonths(addedMonths);
            if(payData.getFirstBillingDate().getDayOfMonth()!= candidateExp.getDayOfMonth()){
                return candidateExp.withDayOfMonth(payData.getFirstBillingDate().getDayOfMonth());
            }
        }
        return payData.getBillingDate().plusMonths(addedMonths);
    }
}
```

## 다음 테스트 선택: 쉬운 테스트

다음 테스트를 선택하자. 이번에 추가할 사례는 다음이다.

- 2만원을 지불하면 만료일이 두 달 뒤가 된다.
- 3만원을 지불하면 만료일이 석 달 뒤가 된다.

지불할 금액이 곧 추가할 개월 수에 비례하므로 계산하기가 쉽다. 2만원을 지불하면 만료일이 두 달 뒤가 되는 테스트부터 추가하자

```java
  @Test
    void 이만원_이상_납부하면_비례해서_만료일_계산(){
        assertExpiryDate(PayData.builder()
                .billingDate(LocalDate.of(2019,3,1))
                .payAmount(20_000)
                .build(),LocalDate.of(2019,5,1));
    }
```

테스트를 실행하면 테스트를 실패하게 된다. 그렇다면 다시 수정하자.

```java
public class ExpiryDateCalculator {
    public LocalDate calculateExpiryDate(PayData payData) {
        int addedMonths = payData.getPayAmount()/10_000;
      ...
```

모든 테스트가 깨지지 않고 통과한다. 이제 3만원을 납부하는 사례를 추가하자.

```java
    @Test
    void 이만원_이상_납부하면_비례해서_만료일_계산(){
        assertExpiryDate(PayData.builder()
                .billingDate(LocalDate.of(2019,3,1))
                .payAmount(20_000)
                .build(),LocalDate.of(2019,5,1));

        assertExpiryDate(PayData.builder()
                .billingDate(LocalDate.of(2019,3,1))
                .payAmount(30_000)
                .build(),LocalDate.of(2019,6,1));
    }
```

이것도 통과한다.

## 예외 상황 테스트 추가

예외 상황을 추가할 차례다. 이번에 추가할 상황은 첫 납부일과 납부일의 일자가 다를 때 2만원 이상 납부한 경우다. 다음은 그예다.

- 첫 납부일이 2019-01-31이고 만료되는 2019-02-28에 2만원을 납부하면 다음 만료일은 2019-04-30이다.

이 사례를 검증하기 위한 테스트 코드를 추가하자

```java
  @Test
    void 첫_납부일과_만료일_일자가_다를때_이만원_이상_납부(){
        assertExpiryDate(
                PayData.builder()
                        .firstBillingDate(LocalDate.of(2019,1,31))
                        .billingDate(LocalDate.of(2019,2,28))
                        .payAmount(20_000)
                        .build(),
                LocalDate.of(2019,4,30)
                );
    }
```

테스트를 실행하면 테스트가 실패한다. 

```markdown
Invalid date 'APRIL 31'
java.time.DateTimeException: Invalid date 'APRIL 31'
```

익셉션 내용을 보면 4월에는 31일이 없는데 31일로 설정해서 그런 것임을 알 수 있다. 이 테스트를 통과시키려면 다음 조건을 확인해야한다.

- 후보 만료일이 포함된 달의 마지막 날 < 첫 납부일의 일자

이 조건이 참이면 후보 만료일을 그달의 마지막 날로 조정해야한다. 이를 구현한 코드는 다음과 같다.

```java
public class ExpiryDateCalculator {
    public LocalDate calculateExpiryDate(PayData payData) {
        int addedMonths = payData.getPayAmount()/10_000;
        if(payData.getFirstBillingDate()!=null) {
            LocalDate candidateExp = payData.getBillingDate().plusMonths(addedMonths);
            if(payData.getFirstBillingDate().getDayOfMonth()!= candidateExp.getDayOfMonth()){
                if(YearMonth.from(candidateExp).lengthOfMonth()< payData.getFirstBillingDate().getDayOfMonth()){
                    return candidateExp.withDayOfMonth(YearMonth.from(candidateExp).lengthOfMonth());
                }
                return candidateExp.withDayOfMonth(payData.getFirstBillingDate().getDayOfMonth());
            }
        }
        return payData.getBillingDate().plusMonths(addedMonths);
    }
}
```

구현을 추가했다. 다시 테스트를 돌려보면 테스트가 통과한다.

## 코드정리

코드를 정리하는 과정은 레포지토리의 주인이 생략하며 결과창을 나타나겠다.

```java
public class ExpiryDateCalculator {
    public LocalDate calculateExpiryDate(PayData payData) {
        int addedMonths = payData.getPayAmount()/10_000;
        if(payData.getFirstBillingDate()!=null)
            return expiryDateUsingFirstBillingDate(payData,addedMonths);
        else
            return payData.getBillingDate().plusMonths(addedMonths);
    }
    private LocalDate expiryDateUsingFirstBillingDate(PayData payData, int addedMonths){
        
        LocalDate candidateExp = payData.getBillingDate().plusMonths(addedMonths);
        final int dayOfFirstBilling = payData.getFirstBillingDate().getDayOfMonth();
        
        if(dayOfFirstBilling!= candidateExp.getDayOfMonth()){
            final int dayLenOfCandiMon = YearMonth.from(candidateExp).lengthOfMonth();
            if(dayLenOfCandiMon<dayOfFirstBilling)
                return candidateExp.withDayOfMonth(dayLenOfCandiMon);
            return candidateExp.withDayOfMonth(dayOfFirstBilling);
        }else
            return candidateExp;
    }
}
```

## 다음 테스트: 10개월 요금을 납부하면 1년 제공

이제 10만원을 납부하면 서비슬르 1년 제공한다는 규칙을 구현할 차례다. 지금까지 했던 것처럼 먼저할일은 테스트 코드를 추가하는 일이다.

```java
		@Test
    void 십만원을_납부하면_1년_제공(){
        assertExpiryDate(PayData.builder()
                .billingDate(LocalDate.of(2019,1,28))
                .payAmount(100_000)
                .build(), LocalDate.of(2020,1,28));
    }
```

테스트를 실행하면 새로 추가한 메소드에서 실패한다. 

```markdown
expected: <2020-01-28> but was: <2019-11-28>
Expected :2020-01-28
Actual   :2019-11-28
```

이 테스트를 통과시켜보자. 쉬운 방법은 지불한 금액이 10만원인지 여부를 비교하는 것이다. 테스트를 통과시키기 위해 수정하는 코드는 다음과같다.

```java
public class ExpiryDateCalculator {
    public LocalDate calculateExpiryDate(PayData payData) {
        int addedMonths = payData.getPayAmount() ==100_000 ? 12 : payData.getPayAmount()/10_000;
      ...
```

더 진행해야 하는 테스트는 2020년 2월 29일과 같은 윤달 마지막날에 10만원을 납부하는 상황 등인데 이는 스스로 사례를 검증하고 테스트를 추가하자.

---



## 시작이 안 될 때는 단언부터 고민

테스트 코드를 작성하다 보면 시작이 잘 안될 때가 있다. 이럴 땐 검증하는 코드부터 작성하기 시작함녀 도움이 된다.

예를 들어 만료일 계산 기능의 경우 만료일을 검증하는 코드부터 작성해 보는 것이다

```java
@Test
void 만원_납부하면_한달_뒤가_만료일이_됨(){
	//처음 작성하는 코드
	assertEquals(기대하는 만료일, 실제 만료일);
}
```

먼저 만료일을 어떻게 표현할지 결정해야 한다. 만료일이므로 날짜를 표현하는 타입을 선택하면 좋을 것 같다. 자바8의 LocalDate를 이용하자

```java
	assertEquals(LocalDate.of(2019,8,9), 실제 만료일);
```

다음은 실제 만료일을 바꿀 차례다. 이 값은 만료일을 실제로 계산한 결과값을 갖는 변수로 바꿀 수 있다.

```java
LocalDate realExpiryDate = 계산
assertEquals(LocalDate.of(2019,8,9), realExpiryDate);
```

이제 realExpiryDate 변수를 구하는 코드를 작성할 차례다. 만료일을 계산하는 기능이 필요하므로 다음과 같이 코드를 작성할 수 있다.

```java
LocalDate realExpiryDate = cal.calculateExpiryDate(파라미터);
asseryEquals(LocalDate.of(2019,8,9),realExpiryDate);
```

cal의 정확한 타입은 모르지만 어떤 객체의 메소드를 실행해서 계산 기능을 실행하도록 했다. 이제 두 가지를 정해야 한다. cal의 타입과 파라미터 타입이다.

만료일을 계산하는데 납부일과 납부액이 있어야 만료일을 계산할 수 있으므로 파라미터에는 이 두 값을 전달한다. 만원을 납부했을때 한 달 뒤가 만료일이

되는지를 테스트할 것이므로 납부일과 2019-07-09를 전달하고 납부액으로 10,000 원을 전달하게 코드를 수정한다.

```java
LocalDate realExpiryDate= cal.calculateExpiryDate(LocalDate.of(2019,7,9),10_000);
assertEquals(LocalDate.of(2019,8,9),realExpiryDate);
```

cal의 타입은 간단한 만료일 계산을 뜻하는 ExpiryDateCalculator로 정해보자. 이제 위코드는 다음과 같이 바뀐다.

```java
ExpiryDateCalculator cal = new ExpiryDateCalculator();
LocalDate realExpiryDate = cal.calculateExpiryDate(LocalDate.of(2019,7,9),10_000);
asseryEquals(LocalDate.of(2019,8,9), realExpiryDate);
```

이렇게 테스트 코드를 어떻게 작성할지 감을 못잡겠다면 검증 코드부터 시작해보자. 테스트 코드를 작성할 때 많은 도움이 될 것이다.

## 구현이 막히면

TDD를 진행하다 보면 구현이 막힐때가있다. 어떻게 해야 할지 생각이 잘 나지 않거나 무언가 잘못한 것 같은 느낌이 들것이다. 이럴 땐 과감하게 코드를

지우고 미련 없이 다시시작한다. 어떤 순서로 테스트 코드를 작성했는지 돌이켜보고 순서를 바꿔서 다시 진행한다. 다시 진행할 때에는 다음을 상기한다.

- 쉬운 테스트, 예외적인 테스트
- 완급 조절

