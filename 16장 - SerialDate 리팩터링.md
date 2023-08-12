# SerialDate 리팩터링

- [첫째, 돌려보자](#첫째-돌려보자)
- [둘째, 고쳐보자](#둘째-고쳐보자)
    - [주석 정보 없애기](#주석-정보-없애기)
    - [와일드카드 사용하기](#와일드-카드-사용하기)
    - [Javadoc 주석, HTML 태그 사용?](#javadoc-주석-html-태그-사용)
    - [SerialDate 클래스에서 DayDate 클래스로 바뀐 이유](#serialdate-클래스에서-daydate-클래스로-바뀐-이유)
    - [직렬화 제어](#직렬화-제어)
    - [첫 해 변수와 마지막 해 변수](#첫-해-변수와-마지막-해-변수)
    - [DayDateFactory.java](#daydatefactoryjava)
    - [SpreadsheetFactory.java](#spreadsheetdatefactoryjava)
    - [요일 상수](#요일-상수)
    - [기본 생성자](#기본-생성자)
    - [for 문 내 if 문 두 개](#for-문-내-if-문-두-개)
    - [Day enum 클래스](#day-enum-클래스)
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

### 일렬번호를 나타내는 두 변수
일렬 번호를 나타내는 두 변수는 DayDate 클래스가 표현할 수 있는 최초 날짜와 최후 날짜를 의미한다. 
```java
/** 1900년 1월 1일에서 시작하는 직렬 번호 */
public static final int SERIAL_LOWER_BOUND = 2;

/** 9999년 12월 31일에 끝나는 직렬 번호 */
public static final int SERIAL_UPPER_BOUND = 2958465;
```
좀 더 깔끔하게 바꿔보자.
```java
public static final int EARLIEST_DATE_ORDINARY = 2; // 1/1/1900
public static final int LATEST_DATE_ORDINARY = 2958465; // 12/31/9999
```

EARLIEST_DATE_ORDINARY이 0이 아니라 2인 이유는 분명치 않지만 마이크로소프트 엑셀에서 날짜를 표현하는 방식과 관련이 있는 듯 하다. 이 사안은 SpreadsheetDate 클래스에서 설명하고 있다.

```java
/**
 * 엑셀은 1900년 1월 1일을 1로 취급하는 관례를 사용한다.
 * 이 클래스는 1900년 1월 1일을 2로 취급하는 관례를 사용한다.
 * 이 클래스를 사용할 경우 1900년도 1월과 2월을 계산하면
 * 날짜 번호가 엑셀 계산과 달라진다. 하지만 엑셀은
 * (1900년 2월 29일이 실제로 존재하지 않지만!) 하루를 추가하므로
 * 이 날짜 이후부터 계산한 결과는 일치한다.
```
그런데 이 사안은 SpreadsheetDate의 구현과 관련되지 DayDate와 아무러 상관이 없다. 그래서 EARLIEST_DATE_ORDINARY과 LATEST_DATE_ORDINARY이 DayDate에 속하지 않으며 SpreadsheetDate 클래스로 옮겨져야 한다고 판단한다.
### 첫 해 변수와 마지막 해 변수
```java
/** 이 클래스 형식이 지원하는 첫 해 */
public static final int MINIMUM_YEAR_SUPPORTED = 1900;
/** 이 클래스 형식이 지원하는 마지막 해 */
public static final int MAXIMUM_YEAR_SUPPORTED = 9999;
```
두 변수 때문에 갈등이 생긴다. DayDate는 어디까지나 추상 클래스로, 구체적인 구현 정보를 포함할 필요가 없다. 그렇다면 최소 년도와 최대 년도를 지정할 이유도 없다. 그래서 두 변수를 SpreadsheetDate 클래스로 옮길 생각이다. 하지만 RelativeDayOfWeekRule이라는 클래스에서도 두 변수를 사용한다. 

즉, 추상 클래스 사용자가 구현 정보를 알아야 한다는 딜레마가 생긴다.

DayDate 자체를 훼손하지 않으면서 구현 정보를 전달할 방법이 필요하다. 일반적으로 파생 클래스의 인스턴스로부터 구현 정보를 전달할 방법이 필요하다.

DayDate 인스턴스는 getPreviousDayOfWeek, getNearestDayOfWeek, getFollowingDayOfWeek 메서드 중 하나가 생선한다. 세 메서드는 모두 addDays가 생성하는 DayDate 인스턴스를 반환한다. 그리고 addDays 메서드는 createInstance 메서드를 호출해 DayDate 인스턴스를 생성한다. 그런데 createInstance 메서드는 SpreadsheeDate 인스턴스를 생성한다!

일반적으로 부모 클래스는 파생 클래스를 몰라야 바람직하다. Abstract Factory 패턴을 적용해 DayDateFactory 를 만들어서, DayDate 인스턴스를 생성하여 (최대 날짜와 최소 날짜 등) 구현 관련 질문에 답할 수 있다
### DayDateFactory.java
```java
public abstract class DayDateFactory {
    private static DayDateFactory factory = new SpreadsheetDateFactory();
    public static void setInstance(DayDateFactory factory) {
        DayDateFactory.factory = factory;
    }

    protected abstract DayDate _makeDate(int ordinal);
    protected abstract DayDate _makeDate(int day, DayDate.Month month, int year);
    protected abstract DayDate _makeDate(int day, int month, int year);
    protected abstract DayDate _makeDate(java.util.Date date);
    protected abstract int _getMinimumYear();
    protected abstract int _getMaximumYear();

    public static DayDate makeDate(int ordinal) {
        return factory._makeDate(ordinal);
    }

    public static DayDate makeDate(int day, DayDate.Month month, int year) {
        return factory._makeDate(day, month, year);
    }

    public static DayDate makeDate(int day, int month, int year) {
        return factory._makeDate(day, month, year);
    }

    public static DayDate makeDate(java.util.Date date) {
        return factory._makeDate(date);
    }

    public static int getMinimumYear() {
        return factory._getMinimumYear();
    }

    public static int getMaximumYear() {
        return factory._getMaximumYear();
    }
}
```
위 클래스에서 createInstance 메서드를 makeDate라는 이름으로 바꾸었다. 이름이 좀 더 서술적이다.

추상 메서드로 위임하는 정적 메서드는 Singleton, Decorator, Abstract Factory 패턴 조합을 사용한다.
### SpreadsheetDateFactory.java
```java
public class SpreadsheetDateFactory extends DayDateFactory {
    public DayDate _makeDate(int ordinal) {
        return new SpreadsheetDate(ordinal);
    }

    public DayDate _makeDate(int day, DayDate.Month month, int year) {
        return new SpreadsheetDate(day, month, year);
    }

    public DayDate _makeDate(int day, int month, int year) {
        return new SpreadsheetDate(day, month, year);
    }

    public DayDate _makeDate(Date date) {
        final GregorianCalendar calendar = new GregorianCalendar();
        calendar.setTime(date);
        return new SpreadsheetDate(
            calendar.get(Calendar.DATE),
            DayDate.Month.make(calendar.get(Calendar.MONTH) + 1),
            calendar.get(Calendar.YEAR)
        );
    }

    protected int _getMinimumYear() {
        return SpreadsheetDate.MINIMUM_YEAR_SUPPORTED;
    }

    protected int _getMaximumYear() {
        return SpreadsheetDate.MAXIMUM_YEAR_SUPPORTED;
    }
}
```
위에서 보면 MINIMUM_YEAR_SUPPORTED 변수와 MAXIMUM_YEAR_SUPPORTED 변수가 적절한 위치인 SpreadsheetDate 클래스로 옮긴 것을 확인할 수 있다.

### 요일 상수
DayDate에 나타나는 문제점은 요일 상수이다. 이들은 enum이어야 마땅하다. 
```java
public static final int MONDAY = Calendar.MONDAY;
//...
```
### 기본 생성자
기본 생성자도 제거했다. 그 이유는 기본 생성자는 컴파일러가 기본으로 생성한다.
```java
protected SerialDate() {}
```
### for 문 내 if 문 두 개
for 루프 안에 if 문이 두 번 나온다. 별로 맘에 들지 않는다. 그래서 || 연산자를 사용해 if 문을 하나로 만든다. 
```java
for (int i = 0; i < weekDayNames.length; i++) {
    if (s.equals(shortWeekdayNames[i]) || s.equals(weekDayNames[i])) {
        result = i;
        break;
    }
}
```
### Day enum 클래스
```java
public enum Day {
    MONDAY(Calendar.MONDAY),
    TUESDAY(Calendar.TUESDAY),
    //...

    public final int index;

    pirvate static DateFormatSymbols dateSymbols = new DateFormatSymbols();

    Day(int day) {
        index = day;
    }

    public static Day make(int index) throws IllegalArgumentException {
        for (Day d : Day.values()) {
            if (d.index == index) {
                return d;
            }
        }
        throw new IllegalArgumentException(
            String.format("Illegal day index: %d.", index)
        );
    }

    public static Day parse(String s) throws IllegalArgumentException {
        String[] shortWeekdayNames =    
            dateSymbols.getShortWeekdays();
        String[] weekDayNames =
            dateSymbols.getWeekdays();

        s = s.trim();
        for (Day day : Day.values()) {
            if (s.equalsIgnoreCase(shortWeekdayNames[day.index]) ||
                s.equalsIgnoreCase(weekDayNames[day.index])) {
                   return day;
            }
        }
        throw new IllegalArgumentException(
            String.format("%s is not a valid weekday string", s)
        );
    }

    public String toString() {
        return dateSymbols.getWeekdays()[index];
    } 
}
```
