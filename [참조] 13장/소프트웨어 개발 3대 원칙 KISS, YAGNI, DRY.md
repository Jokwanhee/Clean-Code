# 소프트웨어 개발 3대 원칙
우리는 더 좋은 소프트웨어를 만들기 위해 품질 좋은 코드를 작성한다. 그렇기 위해서는 아래의 대표적인 원칙 세 가지가 있다.
- KISS
- YAGNI
- DRY
## KISS (Keep It Simple Stupid)
소프트웨어 설계하는 작업이나 코딩을 하는 행위에서 되도록이면 간단하고 단순하게 만드는 것이 좋다는 원리로 소스 코드나 설계 내용이 '불필요하게' 장황하거나 복잡해지는 것을 경계하라는 원칙이다.

- 단순할수록 이해하기 쉽다.
- 이해하기 쉬울수록 버그가 발생할 가능성이 줄어든다.
- 곧 생산성 향상으로 연결된다.
> 작업이 불필요하게 복잡해지는 것을 항상 경계하자
## YAGNI (You Ain't Gonna Need It)
"필요한 작업만 해라" 라는 뜻이다.

프로그램을 작성하다 보면, 현재는 사용하지 않지만 확장성을 고려해서 미리 작업해 놓은 것들이 있다. 그런 작업들을 하지 말자는 원칙이다.

> 당장 필요한 작업에 집중하고 쓸데없는 작업은 하지 말라는 것이다.
## DRY (Do not Repeat Yourself)
"반복하지마라" 라는 뜻이다.

소스 코드에서 동일한 코드가 반복이 된다는 것은 잠재적인 버그의 위협을 증가시킨다. 반복되는 코드 내용이 변경될 필요가 있는 경우, 반복되는 모든 코드에 찾아가서 수정을 해야 한다. 이 과정에서 실수가 발생한다면 버그가 발생하게 된다.

> 작은 코드라도 반복해서 사용하지 않는 습관이 중요하다.

___
### 참조
https://blog.naver.com/complusblog/221163007357