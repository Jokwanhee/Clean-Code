# Junit 들여다보기
JUnit은 자바 프레임워크 중에서 가장 유명하다. 이 장에서는 JUnit 프레임워크에서 가져온 코드를 평가한다.

- [JUnit 프레임워크](#1-junit-프레임워크)
    - [ComparisonCompactorTest.java](#comparisoncompactortestjava)
    - [ComparisonCompactor.java](#comparisoncompactorjava)
    - [ComparisonCompactor.java (디팩터링 결과)](#comparisoncompactorjava-디팩터링-결과)
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
- 그렇다면 코드를 어떻게 개선하면 좋을까?

