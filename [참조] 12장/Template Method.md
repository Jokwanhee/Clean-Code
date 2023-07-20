# Template Method Pattern
- **Template Method Pattern 이 적용되지 않은 코드 (중복이 존재함을 알 수 있다.)**
```JAVA
public class VacationPolicy {
    public void accrueUSDivisionVacation() {
        // 지금까지 근무한 시간을 바탕으로 휴가 일수를 계산하는 코드
        // ...
        // 휴가 일수가 미국 최소 법정 일수를 만족하는지 확인하는 코드
        // ...
        // 휴가 일수를 급여 대장에 적용하는 코드
        // ...
    }
    public void accrueEUDivisionVacation() {
        // 지금까지 근무한 시간을 바탕으로 휴가 일수를 계산하는 코드
        // ...
        // 휴가 일수가 유럽연합 최소 법정 일수를 만족하는지 확인하는 코드
        // ...
        // 휴가 일수를 급여 대장에 적용하는 코드
        // ...
    }
}
```
- **Template Method Pattern 이 적용된 코드**
```JAVA
abstract public class VacationPolicy {
    public void accrueVacation() {
        calculateBaseVacationHours();
        alterForLegalMinimums();
        applyToPayroll();
    }

    private void calculateBaseVacationHours() { /*...*/ }
    abstract protected void alterForLegalMinimums();
    private void applyToPayroll() { /*...*/ }
}

public class USVacationPolicy extends VacationPolicy {
    @Override protected void alterForLegalMinimums() {
        // 미국 최소 법정 일수를 사용한다.
    }
}

public class EUVacationPolicy extends VacationPolicy {
    @Override protected void alterForLegalMinimums() {
        // 유럽연합 최소 법정 일수를 사용한다.
    }
}
```
위 코드를 살펴보자.
- 우선 Template Method(템플릿 메서드)는 accrueVacation 메서드이다.

위 코드는 Template Method Pattern 을 적용한 코드이다. 원래는 미국 휴가 처리와 유럽연합 휴가 처리에 대한 함수 2개를 두고 그 내부에서 완전히 똑같지는 않지만 비슷한 맥락의 알고리즘을 중복하기 때문에 이 중복을 없애고자 해당 패턴이 적용된 것이다.

accrueVacation 에서 반복되는 메서드를 내부에 정의하고, 외부에서 변경되어야 할 메서드만 추상 메서드로 외부 클래스에서 따로 재정의한다. 이는 다형성을 제공하고 중복을 없앨 수 있따.

### 장점
- 중복 코드를 줄인다.
- 자식 클래스의 역할을 줄일 수 있다. (즉, 핵심 로직의 관리가 용이)
- 코드를 객체지향적으로 구성할 수 있다.
### 단점
- 추상 메서드가 많아지면서 클래스 관리가 복잡해진다.
- 클래스간의 관계와 코드가 꼬일 염려가 있다.

___
### 참조
https://coding-factory.tistory.com/712   
https://refactoring.guru/design-patterns/template-method