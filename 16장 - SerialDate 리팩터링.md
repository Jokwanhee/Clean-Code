# SerialDate 리팩터링

- [첫째, 돌려보자](#첫째-돌려보자)
___
JCommon 라이브러리를 뒤져보면 org.jfree.date라는 패키지가 있으며, 여기에 SerialDate라는 클래스가 있다.

SerialDate를 구현한 사람은 **데이비드 길버트**다. 우리는 우수한 코드인 SerialDate 코드를 낱낱이 까발긴다.

SerialDate는 날짜를 표현하는 자바 클래스이다. 자바는 이미 java.util.Data, java.util.Calendar 등과 같은 클래스를 제공한다. 
- SerialDate 클래스가 왜 필요할까?

자주 느꼈던 불편함을 없애고자 SerialDate 클래스를 구현했다. 데이비드는 아래의 주석처럼 이야기 했다. 중요한 핵심은 특정 날짜를 표현하기 위해서 SerialDate 클래스를 구현했다라고 생각한다. 몇 가지 핵심도 있을 수도 있다. 그래서 시간 기반 날짜 클래스보다 순수 날짜 클래스를 만들었다. 
```java
/**
 * 날짜를 처리하기 위해 우리 요구사항을 정의하는 추상 클래스로
 * 특정 구현에 얽매이지 않는다.
 * 
 * 요구사항 1: 최소한 액셀에서 제공하는 날짜 기능은 제공해야 한다.
 * 요구사항 2: 클래스는 불변이다.
 * 
 * java.util.Date를 그냥 쓰지 않는 이유는? 사리에 맞다면 그냥 썼을 테다.
 * 종종 java.util.Date는 너무 정밀하다. 이 클래스는 1000분의 1초의
 * 정밀도로 시각을 표현한다.(시간대에 따라 날짜가 달라진다.)
 * 하루 중 시각, 시간대에 무관하게
 * 종종 특정 날짜만 표현하기를 원한다.(에: 2015년 1월 21일)
 * 이게 바로 SerialDate를 재정의한 이유다.
 * 
 * 정확한 구현을 걱정하지 않고서도 getInstance()를 호출해,
 * SerialDate의 구체화된 하위 클래스를 얻을 수 있다.
```
## 첫째, 돌려보자
저자는 SerialDateTest라는 테스트 클래스에서 단위 테스트 케이스를 몇 개 포함한다. 하지만 이는 전체 테스트를 하지않고 있다. 그러므로 테스트 케이스를 직접 추가함으로써 리팩터링하려고 하는 것이다.

주석으로 빼놓은 코드가 대부분이 있다. SerialDate 설계상으로는 통과하지 않아야 맞다. 하지만 저자는 당연히 통과해야 한다고 생각했다.
```java
public class BobsSerialDateTest extends TestCase {
    public void testStringToWeekdayCode() throws Exception {
        // todo assertEquals(MONDAY, stringToWeekdayCode("monday"));
        // aseertEquals(MONDAY, stringToWeekdayCode("MONDAY"));
        // aseertEquals(MONDAY, stringToWeekdayCode("mon"));
        //...
    }
}
```
애초에 stringToWeekdayCode라는 메서드를 왜 만들었는지 모른다. 일단 만들었다면 대소문자 구분 없이 모두 통과해야 한다고 봤다. 이는 equalsIgnoreCase로 바꾸면 충분했다.
```java
public abstract class SerialDate implements Comparable, 
                                            Serializable,
                                            MonthConstants {
    //...
        if (s.equals(shortWeekdayNames[i])) { }
        if (s.equals(weekDayNames[i])) {}
    //...
}
```
위 조건문을 equalsIgnoreCase로 바꾸면 된다.

___
```java
public class BobsSerialDateTest extends TestCase {
    public void testStringToMonthCode() throws Exception {
        // assertEquals(1, stringToMonthCode("jan"));
        // assertEuqals(2, stringToMonthCode("feb"));
        //...
    }

}
```
assertEquals 코드 테스트도 바꾸기 쉽다.
```java
if ((result < 1) || (result > 12)) {
    result = -1;
    for (int i = 0; i < monthNames.length; i++) {
        if (s.equalsIgnoreCase(shortMonthNames[i])) {
            result = i + 1;
            break;
        }
        if (s.equalsIgnoreCase(monthNames[i])) {
            result = i + 1;
            break;
        }
    }
}
```
___
getFollowingDayOfWeek 메서드에 버그가 있다. 2004년 12월 25일은 토요일이었다. 다음 토요일은 2005년 1월 1일이다. 하지만 테스트를 돌리면 getFollowingdayOfWeek 메서드가 12월 25일 다음 토요일로 반환한다. 명백한 버그다.
```java
assertEquals(d(1, JANUARY, 2005), getFollowingDayOfWeek(SATURDAY, d(25, DECEMBER, 2004)));
```
버그는 아래 코드처럼 전형적인 경계 조건 오류였다.
```java
if (baseDOW > targetWeekday) {}
```
개선된 코드
```java
if (baseDOW >= targetWeekday) {}
```
___

아래 코드는 결코 실행되지 않는다. 즉, if문이 항상 거짓이라는 말이다. 변수 adjust는 항상 음수다. 그러므로 4보다 크거나 같을 가능성은 전혀 없다.
```java
if (adjust >= 4) {
    adjust = 7 - adjust;
}
```
즉, 알고리즘이 틀렸다. 올바른 알고리즘은 아래와 같다.
```java
int delta = targetDOW - base.getDayOfWeek();
int positiveDelta = delta + 7;
int adjust = positiveDelta % 7;
if (adjust > 3) {
    adjust -= 7;
}

return SerialDate.addDays(adjust, base);
```
___
마지막으로, weekInMonthToString과 relativeToString에서 오류 문자열을 반환하는 대신 IllegalArgumentException을 던져 테스트를 통과시켰다.

위와 같이 변경한 SerialDate는 모든 테스트 케이스르 통과했다. 이제는 SerialDate 코드가 제대로 돈다고 믿는다. 이제부터 SerialDate 코드를 '올바로' 고쳐보자.
