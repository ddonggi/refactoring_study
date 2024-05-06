### Intro
리팩토링이란?  
리팩토링은 겉으로 드러나는 코드의 기능은 바꾸지 않으면서 내부 구조를 개선하는 방식으로
소프트웨어 시스템을 수정하는 과정이다. 버그가 생길 가능성을 최소화하며 코드를 정리하는 정제된 방법이다.
>리팩토링 한다는 것은 코드를 작성하고 난 뒤 설계를 향상시키는 일  

리팩토링의 단계는 간단하다. 한 클래스의 필드를 다른 클래스로 옮기거나, 일부 코드를 메서드 밖으로 빼서 별도의 메서드로
만들고, 일부 코드를 계층구조의 위아래로 옮기는 등의 작업이다.
이러한 사소한 수정도 누적되면 설걔가 놀랍도록 향상된다.  
**모든 설계를 미리 떠올리는게 아니라 개발 도중에 꾸준히 떠올리개 되며
시스템 제작을 통해 설계 개선 방법을 배운다. 그 결과, 개발이 지속되도 프로그램 설계가 계속 좋은 상태로 유지된다.**  

# 01 맛보기 예제
비디오 대여점에서 고객의 대여료 내역을 계산후 출력하는 프로그램  
고객이 대여한 비디오와 대여 기간을 표시한 후, 비디오 종류(일반,아동,최신)와 대여 기간을 토대로  
대여료를 계산한다. 내역을 바탕으로 적립 포인트도 계산되는데, 이 포인트는 비디오가 최신물
인지 아닌지에 따라 달라진다.

## 원래의 프로그램
**Movie Class**
```java
public class Movie {
    public static final int REGULAR = 0;
    public static final int NEW_RELEASE = 1;
    public static final int CHILDRENS = 2;

    private String _title;
    private int _priceCode;

    public Movie(String title, int priceCode) {
        _title = title;
        _priceCode = priceCode;
    }

    public int getPriceCode() {
        return _priceCode;
    }

    public void setPriceCode(int arg) {
        _priceCode = arg;
    }

    public String getTitle() {
        return _title;
    }
}
```

**Rental Class**
```java
public class Rental {
    private Movie _movie;
    private int _daysRented;

    public Rental(Movie movie, int daysRented) {
        _movie = movie;
        _daysRented = daysRented;
    }

    public int getDaysRented() {
        return _daysRented;
    }

    public Movie getMovie() {
        return _movie;
    }
}
```
**Customer Class**
```java
public class Customer {
    private String _name;
    private Vector _rentals = new Vector();
    
    public Customer(String name) {
        _name = name;
    }
    public void addRental(Rental arg) {
        _rentals.addElement(arg);
    }
    public String getName() {
        return _name;
    }
    
    public String statement() {
        double totalAmount = 0;
        int frequentRenterPoints = 0;
        Enumeration rentals = _rentals.elements();
        String result = "Rental Record for " + getName() + "\n";
        
        while (rentals.hasMoreElements()) {
            double thisAmount = 0;
            Rental each = (Rental) rentals.nextElement();
            
            // 비디오 종류별 대여료 계산
            switch (each.getMovie().getPriceCode()) {
                case Movie.REGULAR:
                    thisAmount += 2;
                    if (each.getDaysRented() > 2)
                        thisAmount += (each.getDaysRented() - 2) * 1.5;
                    break;
                case Movie.NEW_RELEASE:
                    thisAmount += each.getDaysRented() * 3;
                    break;
                case Movie.CHILDRENS:
                    thisAmount += 1.5;
                    if (each.getDaysRented() > 3)
                        thisAmount += (each.getDaysRented() - 3) * 1.5;
                    break;
            }
            
            // 적립 포인트를 1 포인트 증가
            frequentRenterPoints++;
            // 최신물을 이틀 이상 대여하면 보너스 포인트 지급
            if ((each.getMovie().getPriceCode() == Movie.NEW_RELEASE) && each.getDaysRented() > 1)
                frequentRenterPoints++;
            
            // 이번에 대여하는 비디오 정보와 대여료를 출력
            result += "\t" + each.getMovie().getTitle() + "\t" + String.valueOf(thisAmount) + "\n";
            // 현재까지 누적된 총 대여료
            totalAmount += thisAmount;
            // 푸터 행 추가
            result += "누적 대여료 : " + String.valueOf(totalAmount) + "\n";
            result += "적립 대여료 : " + String.valueOf(frequentRenterPoints);
            return result;
        }
    }
}
```
* 엉터리 설계 / 객체지향적이지 않음
* Customer 클래스의 statement 메서드에 지나치게 많은 기능
  * 대부분의 기능은 다른 두 클래스에 들어가야 맞다
* 만약 statement 메서드를 복사해서 다른 메서드를 만들면, 한 기능을 수정할 때마다 두 메서드를 똑같이 수정해야 한다.
> 프로그램에 기능을 추가해야 하는데 코드 구조가 조잡해서 그 기능을 추가하기 힘들다면, 우선 리팩토링을 실시해서 기능을 추가하기 쉽게 만든 후 해당 기능을 추가하자

## 리팩토링 첫 단계
리팩토링할 코드 부분에 대한 신뢰도 높은 각종 테스트를 작성한다. 적절한 테스트 코드를 작성하는 것은 리팩토링의 기본
> 리팩토링 하기 전에 반드시 신뢰도 높은 테스트 스위트가 준비됐는지 확인하자. 이 테스트들은 반드시 자체검사가 되게 작성한다

## statement 메서드 분해와 기능 재분배
statement 메서드같은 긴 메서드에서 작은 부분들로 쪼갤 수 있는지 살펴본다.  
그 후 각 부분을 알맞은 클래스로 옮긴다. 이것은 중복 코드를 줄이고 복사한 다른 메서드를 좀 더 간편하게 작성하기 위해서다.

statement()
* 논리적 코드 뭉치를 찾아서 메서드 추출(Extract Method)을 적용한다. (여기서는 switch문)
* 리팩토링 기법을 사용할 때에는 무슨 문제가 생길 수 있는지 먼저 알아야 한다.(잘못 추출하면 버그 발생)
* **메서드 안에서만 효력이 있는 모든 지역변수 / 매개변수에 해당하는 부분을 살펴봐야한다.(each, thisAmount)**
* 변경되지 않는 변수는 매개변수로 전달할 수 있다.

```java
public class Customer {
    public String statement() {
        double thisAmount = 0;
        /*
        * ..
        */
        //비디오 종류별 대여료 계산 함수를 호출
        thisAmount = amountFor(each);
    }
  //메소드 추출 기법 - 비디오 종류별 대여료 계산 기능을 빼내어 별도 함수로 작성
    private int amountFor(Rental rental) {
        double result = 0;
        // 비디오 종류별 대여료 계산
        switch (rental.getMovie().getPriceCode()) {
            case Movie.REGULAR:
              result += 2;
                if (rental.getDaysRented() > 2)
                  result += (rental.getDaysRented() - 2) * 1.5;
                break;
            case Movie.NEW_RELEASE:
              result += rental.getDaysRented() * 3;
                break;
            case Movie.CHILDRENS:
              result += 1.5;
                if (rental.getDaysRented() > 3)
                  result += (rental.getDaysRented() - 3) * 1.5;
                break;
        }
        return thisAmount;
    }
}
```
> 리팩터링은 프로그램을 조금씩 단계적으로 수정하므로 실수해도 버그를 찾기 쉽다.

변수명 및 매개변수명도 수정하였다. 좋은 코드는 그것이 무슨 기능을 하는지 분명히 드러나야 한다.
코드의 기능을 분명히 드러내는 열쇠가 바로 **직관적인 변수명**이다.
> 컴퓨터가 인식 가능한 코드는 바보라도 작성할 수 있지만, 인간이 이해할 수 있는 코드는 실력 있는 프로그래머만 작성할 수 있다.

메서드 옮기기  
amountFor 메서드를 보면 자신이 속한 Customer 클래스의 정보가 아닌 
Rental 클래스의 정보를 이용한다.
**메서드는 대체로 자신이 사용하는 데이터와 같은 객체에 들어 있어야 한다.**
메서드 이동 기법(Move Method)을 사용하여 Rental 클래스로 옮긴다.
매서드명 변경 및 매개변수를 삭제하였다

```java
class Rental {
    /*
     * ..
     */
    double getCharge(){
        double result = 0;
        switch (getMovie().getPriceCode()) {
            case Movie.REGULAR:
                result += 2;
                if (getDaysRented() > 2)
                    result += (getDaysRented() - 2) * 1.5;
                break;
            case Movie.NEW_RELEASE:
                result += getDaysRented() * 3;
                break;
            case Movie.CHILDRENS:
                result += 1.5;
                if (getDaysRented() > 3)
                    result += (getDaysRented() - 3) * 1.5;
                break;
        }
    }
}
class Customer {
    /*
     * ..
     */
    private double amountFor(Rental rental) {
        return rental.getCharge();
    }
    /*
     * ..
     */
}
```

기존 Customer에 있던 amountFor메서드는 삭제해야 한다.  
또한 thisAmount 변수도 불필요하게 중복이 되므로 삭제 한다.  
(each.charge()메서드의 결과를 저장하는데만 사용되고 그 후엔 전혀 사용되지 않기 때문이다.)  
아래와 같이 **임시변수를 메서드 호출로 전환(Replace Temp with Query)** 을 적용하여 삭제한다.
```java
import java.util.Enumeration;

class Customer {
    /*
     * ..
     */
  
    Enumeration rentals = _rentals.elements();
    while(rentals.hasMoreElements()) {
        //double thisAmount = 0; //임시변수
        Rental each = (Rental) rentals.nextElement();
        //thisAmount = amountFor(each);
      /*
       * ..
       */
    
        // 이번에 대여하는 비디오 정보와 대여료를 출력
        //result += "\t" + each.getMovie().getTitle() + "\t" + String.valueOf(thisAmount) + "\n";
        //메서드 호출로 전환(Replace Temp with Query)을 적용
        result += "\t" + each.getMovie().getTitle() + "\t" + String.valueOf(each.getCharge()) + "\n";
        // 현재까지 누적된 총 대여료
        totalAmount += each.getCharge();
    }
    
    /*
    private double amountFor(Rental rental) {
      return rental.getCharge();
    }
    */
}

```
적립 포인트 계산 부분도 메서도르 빼낸 후 Rental 클래스로 옮긴다.
```java
class Custormer {
    /*
     * ..
     */
  public String statement() {
      double totalAmount = 0;
      int frequentRenterPoints = 0;
      Enumeration rentals = _rentals.elements();
      while (rentals.hasMoreElements()) {
          Rental each = (Rental) rentals.nextElement();
          // 적립 포인트를 1 포인트 증가
          //frequentRenterPoints++;
          // 최신물을 이틀 이상 대여하면 보너스 포인트 지급
          //if ((each.getMovie().getPriceCode() == Movie.NEW_RELEASE) && each.getDaysRented() > 1)
          //    frequentRenterPoints++;
          frequentRenterPoints += each.getFrequentRenterPoints();
      }
      // 푸터 행 추가
      result += "누적 대여료 : " + String.valueOf(totalAmount) + "\n";
      result += "적립 포인트 : " + String.valueOf(frequentRenterPoints);      
  }
    private int frequentRenterPoints(Rental each) {
        return each.getFrequentRenterPoints();
    }
    /*
     * ..
     */
}

class Rental {
    /*
     * ..
     */
    int getFrequentRenterPoints() {
        if ((getMovie().getPriceCode() == Movie.NEW_RELEASE) && getDaysRented() > 1)
            return 2;
        else
            return 1;
    }
}
```
  
임시변수 없애기
totalAmount 변수와 frequentRenterPoints 변수도 불필요하게 중복이 되므로 질의메서드(Query Method)로 고친다.  
질의 메서드는 필요한 값을 반환하고자 호출되는 메서드 이다.
```java
class Customer {
    public String statement() {
      //double totalAmount = 0;
      //int frequentRenterPoints = 0;
        Enumeration rentals = _rentals.elements();
        String result = getName() + "고객님의 대여 기록 \n";
        while (rentals.hasMoreElements()) {
            Rental each = (Rental) rentals.nextElement();
            // 이번에 대여하는 비디오 정보와 대여료를 출력
            result += "\t" + each.getMovie().getTitle() + "\t" + String.valueOf(each.getCharge()) + "\n";
        }
        // 푸터 행 추가
      //result += "누적 대여료 : " + String.valueOf(totalAmount) + "\n";
      //result += "적립 포인트 : " + String.valueOf(frequentRenterPoints);
        result += "누적 대여료 : " + String.valueOf(getTotalCharge()) + "\n";
        result += "적립 포인트 : " + String.valueOf(getTotalFrequentRenterPoints());
        return result;
    }
    //질의 메서드
    private double getTotalCharge() {
        double result = 0;
        Enumeration rentals = _rentals.elements();
        while (rentals.hasMoreElements()) {
            Rental each = (Rental) rentals.nextElement();
            result += each.getCharge();
        }
        return result;
    }
    //질의 메서드
    private int getTotalFrequentRenterPoints() {
        int result = 0;
        Enumeration rentals = _rentals.elements();
        while (rentals.hasMoreElements()) {
            Rental each = (Rental) rentals.nextElement();
            result += each.getFrequentRenterPoints();
        }
        return result;
    }
}
```
해당 리팩토링은 코드가 늘고, while문도 1회에서 3회로 늘어났다. 하지만 while문은 최적화 단계에서 걱정해도 늦지 않다.
이제 getTotalCharge()와 getTotalFrequentRenterPoints() 메서드는 Customer 클래스 안 어디서나 사용할 수 있다.
statement를 복사한 메서드 안에서도 재사용 할 수 있다. 이렇게 하면 메서드 안의 계산식을 바꿔야 할 때도 한군데만 수정하면 된다.  

그러나 사용자들의 요구가 생겼다!! 대여점의 비디오 분류를 바꾸려고 준비 중이다.
조건문 코드를 수정해서 비디오 분류를 변경해야 한다. 분류를 어떻게 변경할지는 결정되지 않았고, 시존과 다른 방식으로 분류하리란 것만 알 수 있다.
수정하는 각 비디오 분류마다 대여료와 적립 포인트으이 적립 비율도 결정해야 한다. 현재 단계에서 수정하기엔 무리이기에
우선 대여료 메서드와 적립 포인트 메서드 부터 마무리 짓고 조건문 코드를 수정하여 비디오 분류를 변경해야 한다.
## 가격 책정 부분의 조건문의 재정의로 교체
제일 먼저 고칠 부분은 switch 부분이다.
타 객체의 속성을 switch문으로 처리하는 것은 나쁜 방법이다. 자신의 데이터를 사용해야 한다.
기존 Rental 클래스의 getCharge() 메서드를 Movie 클래스로 옮긴다.
```java
class Rental {
    double getCharge(){
        double result = 0;
        switch (getMovie().getPriceCode()) {
            case Movie.REGULAR:
                result += 2;
                if (getDaysRented() > 2)
                    result += (getDaysRented() - 2) * 1.5;
                break;
            case Movie.NEW_RELEASE:
                result += getDaysRented() * 3;
                break;
            case Movie.CHILDRENS:
                result += 1.5;
                if (getDaysRented() > 3)
                    result += (getDaysRented() - 3) * 1.5;
                break;
    }
    return result;
}

//Movie 클래스로 옮기기
class Movie {
    double getCharge(int daysRented) {
        double result = 0;
        switch (getPriceCode()) {
            case Movie.REGULAR:
                result += 2;
                if (daysRented > 2)
                    result += (daysRented - 2) * 1.5;
                break;
            case Movie.NEW_RELEASE:
                result += daysRented * 3;
                break;
            case Movie.CHILDRENS:
                result += 1.5;
                if (daysRented > 3)
                    result += (daysRented - 3) * 1.5;
                break;
        }
        return result;
    }
}
```
getCharege() 메서드의 매개변수로 daysRented를 받아서 사용하도록 변경하였다.
## 고찰

### 내 고찰
* 리팩토링 전 리팩토링할 코드 부분에 대한 테스트를 작성하는 것에 대해 생각지도 못하였다.
* 리팩토링이 뭔지 알고싶거나, 까먹는다면 이 부분만 보면 된다고 하여 작성하였다.
