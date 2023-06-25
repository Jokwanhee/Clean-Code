# SOLID 원칙 중 SRP
## SRP(단일책임의 원칙 : Single Responsibility Principle)
어떤 변화에 의해 클래스를 변경해야 하는 이유는 오직 하나뿐이어야 함을 의미한다. 책임을 적절히 분배함으로써 코드의 가독성 향상, 유지보수 용이라는 이점까지 누릴 수 있다.
### 적용방법
- 여러 원인에 의한 변경 (Divergent change) 
    - 혼재된 각 책임을 각각의 개별 클래스로 분할하여 클래스 당 하나의 책임만을 맡도록 하는 것
- 산탄총 수술 (Shotgun surgery)
    - 산발적으로 여러 곳에 분포된 책임들을 한 곳에 모으면서 설계를 깨끗하게 한다. 이는 응집성을 높이는 작업이다.
> 응집성이 높은 것은? 일반적으로 프로그램의 한 요소가 특정 목적을 위해 밀접하게 연관된 기능들이 모여서 구현되어 있고, 지나치게 많은 일을 하지 않으면 그것을 응집도가 높다고 표현한다.
### 적용사례
- SRP 적용 전
```java
class Guitar(){
    public Guitar(
        String serialNumber,
        Double price,
        Type type,
        String model
    ){
        this.serialNumber = serialNumber;
        this.price = price;
        this.type = type;
        this.model = model;
    }

    private String serialNumber;
    private Double price;
    private Type type;
    private String model;

    public String getSerialNumber();
    public Double getPrice();
    public Type getType();
    public String getModel();
}
```
- SRP 적용 후
```java
class Guitar(){
    public Guitar(
        String serialNumber,
        GuitarSpec spec
    ){
        this.serialNumber = serialNumber;
        this.spec = spec;
    }

    private String serialNumber;
    private GuitarSpec spec;
}

class GuitarSpec(){
    Double price;
    Type type;
    String model;
}
```
> Guitar 에 존재하는 모든 정보 중 동적으로 바뀌는 정보를 GuitarSpec 클래스에 별도로 관리함으로써 변화에 의해 변경되는 부분을 한 곳에서 관리할 수 있게되었다. (serialNumber 는 기타의 고유 번호이므로 정적이다.)



___
### 참조
https://www.nextree.co.kr/p6960/