# Junit 들여다보기
JUnit은 자바 프레임워크 중에서 가장 유명하다. 이 장에서는 JUnit 프레임워크에서 가져온 코드를 평가한다.

- [JUnit 프레임워크](#1-junit-프레임워크)
    - [ComparisonCompactorTest.java](#comparisoncompactortestjava)
    - [ComparisonCompactor.java](#comparisoncompactorjava)
    - [ComparisonCompactor.java (디팩터링 결과)](#comparisoncompactorjava-디팩터링-결과)
    - [ComparisonCompactor 개선하기](#comparisoncompactor-개선하기)
    - [ComparisonCompactor.java (최종 버전)](#comparisoncompactorjava-최종-버전)
___
## 1. JUnit 프레임워크
JUnit의 시작은 켄트 벡과 에릭 감마, 두 사람이다. 이 두사람은 함께 아틀란타 행 비행기를 타고 가다 JUnit을 만들었다.

살펴볼 모듈은 문자열 비교 오류를 파악할 때 유용한 코드다.
ComparisonCompactor라는 모듈이다. ComparisonCompactor는 두 문자열을 받아 차이를 반환한다. 예를 들어, ABCDE와 ABXDE를 받아 비교하는 것이다. 코드를 보는 것이 더 이해하기 쉬울 것이다.

### ComparisonCompactorTest.java
```JAVA
public class ComparisonCompactorTest extends TestCase {
    public void testMessage() {
        String failure = new ComparisonCompactor(0, "b", "c").compact("a");
        assertTrue("a expected:<[b]> but was:<[c]>".equals(failure));
    }

    public void testStartSame() {
        String failure = new ComparisonCompactor(1, "ba", "bc").compact(null);
        assetEquals("expected:<b[a]> but was:<b[c]>", failure);
    }

    public void testEndSame() {
        String failure = new ComparisonCompactor(1, "ab", "cb").compact(null);
        assertEquals("expected:<[a]b> but was:<[c]b>", failure);
    }
    
    public void testSame() {
        String failure = new ComparisonCompactor(1, "ab", "ab").compact(null);
        assertEquals("expected:<ab> but was:<ab>", failure);
    }

    public void testNoContextStartAndEndSame() {
        String failure = new ComparisonCompactor(0, "abc", "adc").compact(null);
        assertEquals("expected:<...[b]...> but was:<...[d]...>", failure);
    }

    public void testStartAndEndContext() {
        String failure = new ComparisonCompactor(1, "abc", "adc").compact(null);
        assertEquals("expected:<a[b]c> but was:<a[d]c>", failure);
    }

    public void testStartAndEndContextWithEllipses() {
        String failure = 
            new ComparisonCompactor(1, "abcde", "abfde").compact(null);
        assertEquals("expected:<...b[c]d...> but was:<...b[f]d...>", failure);
    }

    public void testComparisonErrorStartSameComplete() {
        String failure = new ComparisonCompactor(2, "ab", "abc").compact(null);
        assertEquals("expected:<ab[]> but was:<ab[c]>", failure);
    }

    public void testComparisonErrorEndSameComplete() {
        String failure = new ComparisonCompactor(0, "bc", "abc").compact(null);
        assertEquals("expected:<[]...> but was:<[a]...>", failure);
    }

    public void testComparisonErrorEndSameCompleteContext() {
        String failure = new ComparisonCompactor(2, "bc", "abc").compact(null);
        assertEquals("expected:<[]bc> but was:<[a]bc>", failure);
    }

    public void testComparisonErrorOverlapingMatches() {
        String failure = new ComparisonCompactor(0, "abc", "abbc").compact(null);
        assertEquals("expected:<...[]...> but was:<...[b]...>", failure);
    }

    public void testComparisonErrorOverlapingMatchesContext() {
        String failure = new ComparisonCompactor(2, "abc", "abbc").compact(null);
        assertEquals("expected:<ab[]c> but was:<ab[b]c>", failure);
    }

    public void testComparisonErrorOverlapingMatches2() {
        String failure = new ComparisonCompactor(0, "abcdde", "abcde").compact(null);
        assertEquals("expected:<...[d]...> but was:<...[]...>", failure);
    }    

    public void testComparisonErrorOverlapingMatches2Context() {
        String failure = new ComparisonCompactor(2, "abcdde", "abcde").compact(null);
        assertEquals("expected:<...cd[d]e> but was:<...cd[]e>", failure);
    }

    public void testComparisonErrorWithActualNull() {
        String failure = new ComparisonCompactor(0, "a", null).compact(null);
        assertEquals("expected:<a> but was:<null>", failure);
    }

    public void testComparisonErrorWithActualNullContext() {
        String failure = new ComparisonCompactor(2, "a", null).compact(null);
        assertEquals("expected:<a> but was:<null>", failure);
    }

    public void testComparisonErrorWithExpectedNull() {
        String failure = new ComparisonCompactor(0, null, "a").compact(null);
        assertEquals("expected:<null> but was:<a>", failure);
    }

    public void testComparisonErrorWithExpectedNullContext() {
        String failure = new ComparisonCompactor(2, null, "a").compact(null);
        assertEquals("expected:<null> but was:<a>", failure);
    }

    public void testBug609972() {
        String failure = new ComparisonCompactor(10, "S&P500", "0").compact(null);
        assertEquals("expected:<[S&P50]0> but was:<[]0>", failure);
    }
}
```
위에 테스트 케이스가 모든 행, 모든 if문, 모든 for문을 실행한다는 의미이다.

이제부터 ComparisonCompactor 모듈을 살펴보자. 아래의 조건들을 생각하며 살펴보자.
- 코드가 잘 분리되었는지?
- 표현력이 적절한지?
- 구조가 단순한지?
### ComparisonCompactor.java
```java
public class ComparisonCompactor {
    private static final String ELLIPSIS = "...";
    private static final String DELTA_END = "]";
    private static final String DELTA_START = "[";

    private int fContextLength;
    private String fExpected;
    private String fActual;
    private int fPrefix;
    private int fSuffix;

    public ComparisonCompactor(
        int contextLength,
        String expected,
        String actual
    ) {
        fContextLength = contextLength;
        fExpected = expected;
        fActual = actual;
    }

    public String compact(String message) {
        if (fExpected == null || fActual == null || areStringEqual()) {
            return Assert.format(message, fExpected, fActual);
        }

        findCommonPrefix();
        findCommonSuffix();
        String expected = compactString(fExpected);
        String actual = CompactString(fActual);
        return Assert.format(message, expected, actual);
    }

    private String compactString(String source) {
        String result = DELTA_START + 
                source.substring(fPrefix, source.length() - fSuffix + 1) +
                DELTA_END;
        if (fPrefix > 0) {
            result = computeCommonPrefix() + result;
        }
        if (fSuffix > 0) {
            result = result + computeCommonSuffix();
        }
        return result;
    }

    private void findCommonPrefix() {
        fPrefix = 0;
        int end = Math.min(fExpected.length(), fActual.length());
        for (; fPrefix < end; fPrefix++) {
            if (fExpected.charAt(fPrefix) != fActual.charAt(fPrefix))
            break;
        }
    }

    private void findCommonSuffix() {
        int expectedSuffix = fExpected.length() - 1;
        int actualSuffix = fActual.length() - 1;
        for (;
            actualSuffix >= fPrefix && expectedSuffix >= fPrefix;
            actualSuffix--, expectedSuffix--
        ) {
            if (fExpected.charAt(expectedSuffix) != fActual.charAt(actualSuffix))
                break;
        }
        fSuffix = fExpected.length() - expectedSuffix;
    }

    private String computeCommonPrefix() {
        return (fPrefix > fContxtLength ? ELLIPSIS : "") +
            fExpected.substring(Math.max(0, fPrefix - fContextLength),
            fPrefix);
    }

    private String computeCommonSuffix() {
        int end = Math.min(
            fExpected.length() - fSuffix + 1 + fContextLength,
            fExpected.length()
        );
        return fExpected.substring(fExpected.length() - fSuffix + 1, end) + 
            (fExpected.length() - fSuffix + 1 < fExpected.length() -
            fContextLength ? ELLIPSIS : "");
    }

    private boolean areStringEqual() {
        return fExpected.equals(fActual);
    }
}
```
위에서 살펴볼 수 있는 몇 가지 불편사항은 긴 표현식 몇 개와 이상한 +1 등이 눈에 띈다. 하지만 전반적으로 상당히 훌륭한 모듈이라고 할 수 있다. 

안 좋은 모듈을 살펴보자.

디팩터링은 리팩터링의 반대되는 말로 개선되기 전에 코드를 말한다.
### ComparisonCompactor.java (디팩터링 결과)
```java
public class ComparisonCompactor {
    private int ctxt;
    private String s1;
    private String s2;
    private int pfx;
    private int sfx;

    public ComparisonCompactor(int ctxt, String s1, String s2) {
        this.ctxt = ctxt;
        this.s1 = s1;
        this.s2 = s2;
    }

    public String compact(String msg) {
        if (s1 == null || s2 == null || s1.equals(s2)) {
            return Assert.format(msg, s1, s2);
        }
        pfx = 0;
        for (; pfx < Math.min(s1.length(), s2.length()); pfx++) {
            if (s1.charAt(pfx) != s2.charAt(pfx)) {
                break;
            }
        }
        int sfx1 = s1.length() - 1;
        int sfx2 = s2.length() - 1;
        for (; sfx2 >= pfx && sfx1 >= pfx; sfx2--, sfx1--) {
            if (s1.charAt(sfx1) != s2.charAt(sfx2)) {
                break;
            }
        }
        sfx = s1.length() - sfx1;
        String cmp1 = compactString(s1);
        String smp2 = compactString(s2);
        return Assert.format(msg, cmp1, cmp2);
    }

    private String compactString(String s) {
        String result = 
            "[" + s.substring(pfx, s.length() - sfx + 1) + "]";
        if (pfx > 0) {
            result = (pfx > ctxt ? "..." : "") +
                s1.substring(Math.max(0, pfx - ctxt), pfx) + result;
        }
        if (sfx > 0) {
            int end = Math.min(s1.length() - sfx + 1 + ctxt, s1.length());
            result = result + (s1.substring(s1.length() - sfx + 1, end) + 
                (s1.length() - sfx + 1 < s1.length() - ctxt ? "..." : "")
            );
        }
        return result;
    }
}
``` 

처음에서 클린 코드를 소개할 때, 보이스카우트 규칙을 말한 적이 있다. 처음에 왔을 때보다 마지막이 깔끔해야 한다는 규칙이다. 그렇다면 디팩터링한 ComparisonCompactor 말고 그 전에 잘 정리된 ComparisonCompactor 모듈을 개선해야하는 것인가? 
- 개선해야하면 코드를 어떻게 개선하면 좋을까?

### ComparisonCompactor 개선하기
가장 먼저 눈에 거슬리는 부분은 멤버 변수 앞에 붙인 접두어 f다.
그러므로 접두어 f를 모두 제거하자.
```java
private int contextLength;
private String expected;
private String actual;
private int prefix;
private int suffix;
```

다음으로 compact 함수 시작부에 캡슐화되지 않은 조건문이 보인다.
```java
public String compact(String message) {
    if (expected == null || actual == null || areStringsEquals()) {
        return Assert.format(message, expected, actual);
    }

    findCommonPrefix();
    findCommonSuffix();
    String expected = compactString(this.expected);
    String actual = compactString(this.actual);
    return Assert.format(message, expected, actual);
}
```
의도를 분명히 표현하려면 조건문을 캡슐화해야 한다. 즉, 조건문을 메서드로 뽑아내 적절한 이름을 붙인다.
```java
if (expected == null || actual == null || areStringsEquals())
```
```java
if (shouldNotCompact()) { 
    //... 
}

private boolean shouldNotCompact() {
    return expected == null || actual == null || areStringsEqual();
}
```
- 함수에서 멤버 변수와 이름이 똑같은 변수를 사용하는 이유가 무엇일까?
```java
private String expected;

public String compact(String message) {
    String expected = compactString(this.expected);
}
```

서로 다른 의미이므로 이름은 명확하게 붙인다.
```java
private String expected;
private String actual;

public String compact(String message) {
    String compactExpected = compactString(expected);
    String compactActual = compactString(actual);
}
```
부정문은 긍정문보다 이해하기 약간 더 어렵다. 그러므로 첫 문장 if를 긍정으로 만들어 조건문을 반전한다.
```java
public String compact(String message) {
    if (canBeCompated()) {
        //...
    }
}

private boolean canBeCompacted() {
    return expected != null && actual != null && !areStringsEqual();
}
```
compact() 함수 이름이 이상하다. 실제로 해당 함수가 false이면 압축하지 않는다. 그러므로 formatCompactedComparsion 이라는 이름이 적합해보인다.
```java
public String formatCompactedComparison(String message) {}
```

if문 안에서 예상 문자열과 실제 문자열을 진짜로 압축한다. 이 부분을 빼내어 compactExpectedAndActual이라는 메서드로 만든다.
```java
private String compactExpected;
private String compactActual;

public String formatCompactedComparison(String message) {
    if (canBeCompacted()) {
        compactExpectedAndActual();
        return Assert.format(message, compactExpected, compactActual);
    } //...
}

private void compactExpectedAndActual() {
    findCommonPrefix();
    findCommonSuffix();
    compactExpected = compactString(expected);
    compactActual = compactString(actual);
}
```
위에서 compactExpected와 compactActual을 멤버 변수로 승격했다. compactExpectedAndActual 함수는 마지막 두 줄은 변수를 반환하지만 첫째 줄과 둘째 줄은 반환값이 없다 즉, 함수 사용방식이 일관적이지 못하다.

findCommonPrefix와 findCommonSuffix를 변경해 접두어 값과 접미어 값을 반환한다.
```java
private void compactExpectedAndActual() {
    prefixIndex = findCommonPrefix();
    suffixIndex = findCommonSuffix();
    compactExpected = compactString(expected);
    compactActual = compactString(actual);
}

private int findCommonPrefix() {
    int prefixIndex = 0;
    int end = Math.min(expected.length(), actual.length());
    for (; prefixIndex < end; prefixIndex++) {
        if (expected.charAt(prefixIndex) != actual.charAt(prefixIndex)) 
            break;
    }
    return prefixIndex;
}

private int findCommonSuffix() {
    int expectedSuffix = expected.length() - 1;
    int actualSuffix = actual.length() - 1;
    for (; actualSuffix >= prefixIndex && expectedSuffix >= prefixIndex; actualSuffix--, expectedSuffix--) {
        if (expected.charAt(expectedSuffix) != actual.charAt(actualSuffix))
            break;
    }
    return expected.length() - expectedSuffix;
}
```

findCommonSuffix 함수를 주의 깊게 살펴보면 숨겨진 시간적인 결합(hidden temporal coupling)이 존재한다. 다시 말해, findCommonSuffix는 findCommonPrefix가 prefixIndex를 계산한다는 사실에 의존한다.

만약에 findCommonPrefix와 findCommonSuffix를 잘못된 순서로 호출하면 오류가 발생할 수 있다. 그래서 시간 결합을 외부에 노출하고자 findCommonSuffix를 고쳐 prefixIndex를 인수로 넘겼다.
```java
private vodi compactExpectedAndActual() {
    prefixIndex = findCommonPrefix();
    suffixIndex = findCommonSuffix(prefixIndex);
    //...
}

private int findCommonSuffix(int prefixIndex) {
    int expectedSuffix = expected.length() - 1;
    int actualSuffix = actual.length() - 1;
    for (; actualSuffix >= prefixIndex && expectedSuffix >= prefixIndex; actualSuffix--, expectedSuffix--) {
        if (expected.charAt(expectedSuffix) != actual.charAt(actualSuffix))
            break;
    }
    return expected.length() - expectedSuffix;
}
```
인수로 뺀 prefixIndex 좀 자의적이다. 필요한 이유를 설명하지 못하고 있다. 다른 방식을 고안해보자.
```java
private void compactExpectedAndActual() {
    findCommonPrefixAndSuffix();
    //...
}

private void findCommonPrefixAndSuffix() {
    findCommonPrefix();
    int expectedSuffix = expected.length() - 1;
    int actualSuffix = actual.length() - 1;
    for (; 
        actualSuffix >= prefixIndex && expectedSuffix >= prefixIndex;
        actualSuffix--, expectedSuffix--
    ) {
        if (expected.charAt(expectedSuffix) != actual.charAt(actualSuffix))
            break;
    }
    suffixIndex = expected.length() - expectedSuffix;
}

private void findCommonPrefix() {
    prefixIndex = 0;
    int end = Math.min(expected.length(), actual.length());
    for (; prefixIndex < end; prefixIndex++) {
        if (expected.charAt(prefixIndex) != actual.charAt(prefixIndex))
            break;
    }
}
```

findCommonPrefix와 findCommonSuffix를 원래대로 되돌리고, findCommonSuffix라는 이름을 findCommonPrefixAndSuffix로 바꾸고, findCommonPrefixAndSuffix에서 가장 먼저 findCommonPrefix를 호출한다.

그러면 두 함수를 호출하는 순서가 앞서 고친 코드보다 훨씬 분명해진다. 이제 함수를 정리해보자.
```java
private void findCommonPrefixAndSuffix() {
    findCommonPrefix();
    int suffixLength = 1;
    for (; !suffixOverlapsPrefix(suffixLength); suffixLength++) {
        if (charFromEnd(expected, suffixLength) !=
            charFromEnd(actual, suffixLength))
            break;
    }
    suffixIndex = suffixLength;
}

private char charFromEnd(String s, int i) {
    return s.charAt(s.length() - i);
}

private boolean suffixOverlapsPrefix(int suffixLength) {
    return actual.length() - suffixLength < prefixLength ||
        expected.length() - suffixLength < prefixLength;
}
```
코드가 훨씬 나아졌다. 코드를 고치니 suffixIndex가 실제로는 접미어 길이라는 사실이 드러난다. 이름이 적절하지 못하다는 의미이다.

prefixIndex도 마찬가지로, "index"와 "length"가 동의어다. 비록 그렇다 하더라도, "length"가 더 합당하다. 실제로 suffixIndex는 0에서 시작하지 않는다. 1에서 시작하므로 진정한 길이가 아니다.

computeCommonSuffix에 +1이 곳곳에 등장하는 이유가 여기에 있다. 고쳐보자.
```java
public class ComparisonCompactor {
    private int suffixLength;

    private void findCommonPrefixAndSuffix() {
        findCommonPrefix();
        suffixLength = 0;
        for (; !suffixOverlapsPrefix(suffixLength); suffixLength++) {
            if (charFromEnd(expected, suffixLength) != 
                charFromEnd(actual, suffixLegnth)) 
                    break;
        }
    }

    private char charFromEnd(String s, int i) {
        return s.charAt(s.length() - i - 1);
    }

    private boolean suffixOverlapsPrefix(int suffixLength) {
        return actual.length() - suffixLength <= prefixLength ||
            expected.length() - suffixLength <= prefixLength;
    }

    private String compactString(String source) {
        String result = 
            DELTA_START +
            source.substring(prefixLength, source.length() - 
                suffixLength) + 
            DELTA_END;
        if (prefixLength > 0)
            result = computeCommonPrefix() + result;
        if (suffixLength > 0)
            result = result + computeCommonSuffix();
        return result;
    }

    private String computeCommonSuffix() {
        int end = Math.min(expecetd.length() - suffixLength +
            contextLength, expected.length()
        );
        return
            expected.substring(expected.length() - suffixLength, end) +
            (expected.length() - suffixLength <
            expected.length() - contextLength ? 
            ELLIPSIS : "");
    }
}
```
computeCommonSuffix에서 +1을 없애고, charFromEnd에 -1을 추가하고 suffixOverlapsPrefix에 <=를 사용했다. 논리적으로 타당하다.

다음 suffixIndex를 suffixLength로 바꿨다. 이는 코드 가독성을 위해서이다.

그런데 문제가 하나 발생했다. +1을 제거하던 중 compactString에서 다음 행을 발견했다.
```java
if (suffixLength > 0)
```

suffixLength가 1씩 감소하므로 >연산자가 아닌 >=연산자로 바꾸어야 한다. 하지만 >=연산자는 말도 안된다. 즉, 코드가 틀렸으며 필경 버그라는 말이다. 엄밀히 말하면 버그는 아니다. 

코드를 조금 분석해보면 이제 if문은 길이가 0이 접미어를 걸러내 첨부하지는 않는다. 원래 코드는 suffixIndex가 언제나 1 이상이었으므로 if 문 자체가 있으나마나였다.

compactString에 있는 if문 둘 다 의심스럽다. 둘 다 필요 없어보인다.
불필요한 if문을 제거하고 compactString 구조를 다듬어 좀 더 깔끔하게 만들자.
```java
private String compactString(String source) {
    return
        computeCommonPrefix() +
        DELTA_START +
        source.substring(prefixLength, source.length() - suffixLength) +
        DELTA_END +
        computeCommonSuffix();
}
```
좀 더 깔끔하게 정리할 여지는 존재한다. 사실 사소하게 이것저것 손볼 곳이 아직 많다. 그래서 최종 코드를 제시하려고 한다.

## ComparisonCompactor.java (최종 버전)
```java
public class ComparisonCompactor {
    private static final String ELLIPSIS = "...";
    private static final String DELTA_END = "]";
    private static final String DELTA_START = "[";

    private int contextLength;
    private String expected;
    private String actual;
    private int prefixLength;
    private int suffixLength;

    public ComparisonCompactor(
        int contextLength, String expected, String actual
    ) {
        this.contextLength = contextLength;
        this.expected = expected;
        this.actual = actual;
    }

    public String formatCompactedComparison(String message) {
        String compactExpected = expected;
        String compactActual = actual;
        if (shouldBeCompacted()) {
            findCommonPrefixAndSuffix();
            compactExpected = compact(expected);
            compactActual = compact(actual);
        }
        return Assert.format(message, compactExpected, compactActual);
    }

    private boolean shouldBeCompacted() {
        return !shouldNotBeCompacted();
    }

    private boolean shouldNotBeCompacted() {
        return expected == null ||
            acutal == null ||
            expected.equals(actual);
    }

    private void findCommonPrefixAndSuffix() {
        findCommonPrefix();
        suffixLength = 0;
        for (; !suffixOverlapsPrefix(); suffixLength++) {
            if (charFromEnd(expected, suffixLength) !=
                charFromEnd(actual, suffixLength))
                break;
        }
    }

    private char charFromEnd(String s, int i) {
        return s.charAt(s.length() - i - 1);
    }

    private boolean suffixOverlapsPrefix() {
        return actual.length() - suffixLength <= prefixLength ||
            expected.length() - suffixLength <= prefixLength;
    }

    private void findCommonPrefix() {
        prefixLength = 0;
        int end = Math.min(expected.length(), actual.length());
        for (; prefixLength < end; prefixLength++) {
            if (expected.charAt(prefixLength) != actual.charAt(prefixLength))
                break;
        }
    }

    private String compact(String s) {
        return new StringBuilder()
            .append(startingEllipsis())
            .append(startingContext())
            .append(DELTA_START)
            .append(delta(s))
            .append(DELTA_END)
            .append(endingContext())
            .append(endingEllipsis())
            .toString();
    }

    private String startingEllipsis() {
        return prefixLength > contextLength ? ELLIPSIS : "";
    }

    private String startingContext() {
        int contextStart = Math.max(0, prefixLength - contextLength);
        int contextEnd = prefixLength;
        return expected.substring(contextStart, contextEnd);
    }

    private String delta(String s) {
        int deltaStart = prefixLength;
        int deltaEnd = s.length() - suffixLength;
        return s.substring(deltaStart, deltaEnd);
    }

    private String endingContext() {
        int contextStart = expected.length() - suffixLength;
        int contextEnd = 
            Math.min(contextStart + contextLength, expected.length());
        return expected.substring(contextStart, contextEnd);
    }

    private String endingEllipsis() {
        return (suffixLength > contextLength ? ELLIPSIS : "");
    }
}
```
코드는 상당히 깔끔하다. 전체 함수는 위상적으로 정렬했으므로 각 함수가 사용된 직후에 정의된다. 분석 함수가 먼저 나오고 조합 함수가 그 뒤를 이어서 나온다.

이 장 초반에서 내렸던 결정 일부를 번복했다는 사실을 눈치챌 수 있다.
- 처음에 추출했던 메서드 몇 개를 formatCompactedComparison에다 도로 집어넣었다다.

리팩터링 하다 보면 원래 했던 변경을 되돌리는 경우가 흔하다. 리팩터링은 코드가 어느 수준에 이를때까지 수많은 시행착오를 반복하는 작업이기 때문이다.

## 결론
우리는 보이스카우트 규칙을 지켰다. 코드를 처음보다 조금 깨끗하게 만드는 책임은 우리 모두에게 있다.