# SerialDate 리팩터링

- [첫째, 돌려보자](#첫째-돌려보자)
- [둘째, 고쳐보자](#둘째-고쳐보자)
    - [주석 정보 없애기](#주석-정보-없애기)
    - [와일드카드 사용하기](#와일드-카드-사용하기)
    - [Javadoc 주석, HTML 태그 사용?](#javadoc-주석-html-태그-사용)
    - [SerialDate 클래스에서 DayDate 클래스로 바뀐 이유](#serialdate-클래스에서-daydate-클래스로-바뀐-이유)
    - [직렬화 제어](#직렬화-제어)
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
## 둘째, 고쳐보자
### 주석 정보 없애기
라이선스 정보, 저작권, 작성자, 변경 이력이 나온다. 법적인 정보는 필요하므로 라이선스 정보와 저작권은 보존한다. 반면, 변경 이력은 1960년대에 나온 방식이므로 없애도 된다.
### 와일드 카드 사용하기
import문 중
```java
import java.text.DateFormatSymbols;
import java.text.SimpleDateFormat;
import java.util.Calendar;
import java.util.GregorianCalendar;
```
위의 라이브러리 import를 아래의 코드처럼 와일드카드를 사용한다.
```java
import java.text.*;
import java.util.*;
```
### Javadoc 주석, HTML 태그 사용?
Javadoc 주석은 HTML 태그를 사용한다. 한 소스 코드에 여러 언어를 사용하다니, 탐탁치 못하다. 이 주석은 네 가지 언어(자바, 영어, Javadoc, HTML)를 사용한다. 그러니 양식을 맞추기 어렵다. 
```JAVA
/**
 * 날짜...
 * <P>
 * ...
 * <P>
 * java.util.Date ...
 * 
 * @author 데이비드 길버트
```
### SerialDate 클래스에서 DayDate 클래스로 바뀐 이유
- 클래스 이름이 SerialDate인 이유가 무엇일까?
- 'Serial'이라는 단어가 왜 들어갈까?
- 클래스가 Serializable에서 파생하니까?

'Serial'이라는 단어가 들어가는 이유는 `SERIAL_LOWER_BOUND`와 `SERIAL_UPPER_BOUND`라는 상수에 있다. 
```java
/** 1900년 1월 1일에서 시작하는 직렬 번호 */
public static final int SERIAL_LOWER_BOUND = 2;

/** 9999년 12월 31일에 끝나는 직렬 번호 */
public static final int SERIAL_UPPER_BOUND = 2958465;
```

클래스 이름이 SerialDate인 이뉴는 '일련번호(Serial Number)'를 사용해 클래스를 구현했기 때문이다. 여기서는 1899년 12월 30일 기준으로 경과한 날짜 수를 사용한다.

두 가지 거슬리는 문제가 있다.

첫 번째, '일련번호'라는 용어는 정확하지 못하다. 일련번호보다 '상대 오프셋(Relative Offset)'이 더 정확하다. '일련번호'라는 용어는 날짜보다 제품 식별 번호로 더 적합하다. 그래서 `SerialDate`클래스 이름은 별로다. 좀 더 서술적인 용어로 '서수(Ordinal)'가 있다.

두 번째, 이 문제가 더 중요하다. SerialDate라는 이름은 구현(implement)을 암시하는데 실상은 추상 클래스다. 구현을 암시할 필요가 전혀 없다. 구현은 숨기는 편이 좋다. 그래서 SerialDate라는 이름의 추상화 수준이 올바르지 못하다고 생각한다. 그냥 Date가 좋다. 하지만 Date는 자바 라이브러리에서 너무 많다. 그러므로 최적은 아니고, 날짜를 다루므로 Day라는 이름도 괜찮다. 하지만 Day도 Date와 마찬가지이다. 밥 아저씨는 DayDate라는 클래스 이름을 사용하기로 하였다.

이제부터 DayDate 는 SerialDate 를 가르킨다.

Comparable과 Serializable을 상속하는 이유는 알겠다. 
- 하지만 MonthConstants를 상속하는 이유는 무엇일까?

MonthConstants 클래스는 달을 정의하는 static final 상수 모음에 불과하다. 상수 클래스를 상속하면 MonthConstants.January와 같은 표현을 사용할 필요가 없다. MonthConstants는 enum으로 정의해야 마땅하다.

```java
public abstract class DayDate implements Comparable, Serializable {
    public static enum Month {
        JANUARY(1),
        FEBRUARY(2),
        MARCH(3),
        APRIL(4),
        MAY(5),
        //...
    }

    Month(int index) {
        this.index = index;
    }

    public static Month make(int monthIndex) {
        for (Month m : Month.values()) {
            if (m.index == monthIndex) {
                return m;
            }
        }
        throw new IllegalArgumentException("Invalid month index " + monthIndex);
    }
    public final int index;
}
```
이제는 testIsValidMonthCode 함수도 없어도 된다.
```java 
public void testIsValidMonthCode() throws Exception {
    for (int i = 1; i <= 12; i++) {
        assertTrue(isValidMonthCode(i));
    }
    assertFalse(isValidMonthCode(0));
    assertFalse(isValidMonthCode(13));
}
```
추가로 넘어온 달이 속한 분기를 반환하는 monthCodeToQuarter 함수도 없어도 된다.
```java
public static int monthCodeToQuarter(final int code) {
    switch (code) {
        case JANUARY:
        case FEBRUARY:
        case MARCH: return 1;
        //...
    }
}
```
### 직렬화 제어
serialVersionUID를 살펴보자. serialVersionUID 변수는 직렬화를 제어한다. 이 변수 값을 변경하면 이전 소프트웨어 버전에서 직렬화한 DayDate를 더 이상 인식하지 못한다. 즉, 이전 버전에서 직렬화한 DayDate 클래스를 복원하려 시도하면 InvalidClassException이 발생하는 것이다.

serialVersionUID 변수를 선언하지 않으면 컴파일러가 자동으로 생성한다. 즉, 모듈을 변경할 때마다 변수 값이 달라진다.

그래서 serialVersionUID 변수를 없애기로 결정했다.


