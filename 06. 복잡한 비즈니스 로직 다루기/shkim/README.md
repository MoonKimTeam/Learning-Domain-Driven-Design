# 복잡한 비즈니스 로직 다루기

앞장에서 트랜잭션 스크립트 및 액티브 레코드 같은 비교적 간단한 비즈니스 로직을 다루는 두 가지 패턴에 대해 논의했습니다.  
이번 장에서는 계속해서 비즈니스 로직의 구현에 대해 다루는데, 복잡한 비즈니스 로직에 사용되는 **도메인 모델 패턴**을 소개합니다.

## 배경

트랜잭션 스크립트와 액티브 레코드 패턴 모두 마틴 파울러의 책 '엔터프라이즈 애플리케이션 아키텍처 패턴'(위키북스, 2015)에서 처음 소개됐습니다.  
파울러는 패턴에 관한 논의 중에 '예제 데이터소스'가 지금 도메인 모델 개념에 관한 책을 집필하고 있다고 말했으며, 이 책이 바로 에릭 에반스의 '도메인 주도 설계'입니다.

에반스가 자신의 책에서 비즈니스 도메인의 모델과 코드를 긴밀하게 연결 짓는 데 쓰이는 **애그리게이트(aggregate)**, **밸류 오브젝트(value object)**, **리포지토리(repository)** 등과 같은 패턴을 제시했습니다.
이런 패턴은 파울러가 그의 책에서 다룬 부분을 보완하며 도메인 모델 패턴을 구현하는 데 효과적인 도구의 집합입니다.

에반스가 소개한 패턴은 종종 **'전술적 도메인 주도 설계(tactical domain-driven design)'** 로 불립니다.
도메인 주도 설계를 적용할 때 비즈니스 로직 구현에 반드시 이 패턴들을 사용해야 하는 것은 아니지만, 복잡한 도메인을 다룰 때 매우 효과적입니다.
'도메인 모델' 패턴과 '애그리게이트' 패턴은 이러한 전술적 패턴의 핵심 구성요소입니다.

## 도메인 모델

도메인 모델 패턴은 복잡한 비즈니스 로직을 다루기 위한 것입니다.
CRUD 인터페이스 대신 복잡한 상태 전환, 항상 보호해야 하는 규칙인 비즈니스 규칙과 불변성을 다룹니다.

헬프데스크 시스템을 구현한다고 가정하겠습니다. 그리고 다음 지원 티켓의 수명주기를 다루는 로직을 설명된 요구사항에서 발췌한 내용을 살펴봅시다:

- 고객은 직면한 문제를 설명하는 지원 티켓을 연다
- 고객과 티켓에 할당된 에이전트 모두 메시지를 추가할 수 있으며, 모든 내용은 지원 티켓에서 관리된다
- 각 티켓은 낮음, 중간, 높음, 긴급의 우선순위를 갖는다
- 에이전트는 티켓의 우선순위에 따른 SLA(응답 제한 시간) 내에 해법을 제시해야 한다
- 할당된 에이전트가 응답 제한 시간 내에 응답하지 못하면 고객은 티켓을 에이전트의 상위 관리자에게 요청할 수 있다
- 티켓이 상위 관리자에게 보고되면 에이전트의 응답 제한 시간이 33% 줄어든다
- 에이전트가 상위 보고된 티켓의 응답 제한 시간의 절반이 지나기 전에 티켓을 열람하지 않으면 자동으로 다른 에이전트에게 할당한다
- 할당된 에이전트의 질문이 고객이 7일 이내에 응답하지 않으면 티켓은 자동으로 닫힌다
- 보류된 티켓은 자동으로 또는 할당된 에이전트에 의해 닫힐 수 있다
- 고객은 티켓이 닫힌 지 7일 이내에 닫힌 티켓을 다시 열 수 있다

이 같은 요구사항은 다양한 규칙 간에 그물 같은 의존성을 형성하고, 모든 규칙은 지원 티켓의 수명주기 관리 로직에 영향을 줍니다.
이 예제는 앞장에서 논의한 CRUD 데이터 입력 화면 같이 간단하지 않습니다.
만약 액티브 레코드 객체를 사용하여 로직을 구현하면 로직이 중복되거나 일부 비즈니스 규칙이 잘못 구현되어 시스템의 상태를 손상시키기 쉽습니다.

### 구현

도메인 모델은 행동(behavior)과 데이터(data) 모두를 포함하는 도메인의 객체 모델입니다.  
DDD의 전술 패턴인 애그리게이트, 밸류 오브젝트, 도메인 이벤트, 도메인 서비스 모두 이 객체 모델의 구성요소입니다.

이제 도메인 모델의 다양한 빌딩 블록이 어떻게 공통 관심사를 다루는지 살펴봅시다.

### 복잡성

도메인 모델은 액티브 레코드 패턴과 완전히 다릅니다.  
단순한 CRUD 인터페이스 대신 복잡한 비즈니스 로직을 다루며, 객체는 순수 객체(plain old object)로 구현하여 데이터베이스나 프레임워크에 종속되지 않습니다.

### 유비쿼터스 언어

도메인 모델의 객체는 기술적 관심사가 아닌 비즈니스 로직을 담당하며, 바운디드 컨텍스트의 유비쿼터스 언어를 사용하여 이를 반영합니다.  
바운디드 컨텍스트의 코드에서 유비쿼터스 언어를 사용함으로써 코드에서 비즈니스 도메인의 개념을 표현하게 됩니다.

## 구성요소

이제 밸류 오브젝트, 애그리게이트, 도메인 서비스 같은 DDD에서 제공하는 도메인 모델 빌딩 블록인 구성요소의 전술적 패턴을 살펴봅니다.

## 밸류 오브젝트

밸류 오브젝트는 예를 들어, 색(color)처럼 **복합적(composition)** 값에 의해 식별되는 객체입니다.

```csharp
class Color
{
    int _red;
    int _blue;
    int _green;
}
```

적색, 녹색, 파랑의 세 필드 값이 복합적으로 색을 정의합니다.  
필드 중 하나라도 값이 바뀌면 새로운 색을 갖는 다른 개념의 새로운 객체가 생성됩니다.  
또한 두 밸류 오브젝트가 같은 값을 갖는다면 서로 교환 가능합니다. 같은 RGB 값을 가진 두 색상은 동일한 색입니다.

밸류 오브젝트의 동등성은 id 필드가 아닌 값으로 판단합니다. 두 개의 Color 인스턴스가 같은 red, green, blue 값을 가진다면 이 둘은 같은 색입니다.

### 유비쿼터스 언어

이름 외로 타입(primitive)에 포함된 문자열(string), 정수(integer), 딕셔너리(dictionary) 같은 데이터 객체로서의 비즈니스 도메인의 개념을 표현하는 것은 원시 집착 코드 악취(primitive obsession code smell)로 알려져 있습니다.

```csharp
class Person
{
    private int _id;
    private string _firstName;
    private string _lastName;
    private string _landlinePhone;
    private string _mobilePhone;
    private string _email;
    private int _heightMetric;
    private string _countryCode;

    public Person(...) {...}
}

static void Main(string[] args)
{
    var dave = new Person(
        id: 30217,
        firstName: "Dave",
        lastName: "Ancelovici",
        landlinePhone: "0237745001",
        mobilePhone: "0873712503",
        email: "dave@learning-ddd.com",
        heightMetric: 180,
        countryCode: "BG");
}
```

Person 클래스 구현에서 값의 대부분은 문자열 타입이고 관례에 따라 값이 할당되었습니다.  
예를 들어, landlinePhone 필드의 값은 유효한 유선 전화번호여야 하고 countryCode 필드는 두 자리의 대문자로 된 국가 코드여야 합니다.  
물론 사용자가 항상 올바른 값을 입력하리라 기대할 수 없기 때문에 결국에는 클래스의 모든 입력 필드를 검사해야 합니다.

이 같은 방식에는 몇 가지 설계 위험이 있습니다.  
우선 유효성 검사 로직이 중복되기 쉽습니다. 둘째, 값이 사용되기 전에 유효성 검사 로직을 호출하게 하기 어렵습니다.   
게다가 다른 엔지니어가 코드베이스를 개선하는 것과 같은 미래를 대비해 유지보수가 더 어렵습니다.

그렇다면 다음과 같이 동일한 Person 객체에 밸류 오브젝트를 사용하는 다른 설계와 비교해봅시다.

```csharp
class Person
{
    private PersonId _id;
    private Name _name;
    private PhoneNumber _landline;
    private PhoneNumber _mobile;
    private EmailAddress _email;
    private Height _height;
    private CountryCode _country;

    public Person(...) { ... }
}

static void Main(string[] args)
{
    var dave = new Person(
        id: new PersonId(30217),
        name: new Name("Dave", "Ancelovici"),
        landline: PhoneNumber.Parse("0237745001"),
        mobile: PhoneNumber.Parse("0873712503"),
        email: Email.Parse("dave@learning-ddd.com"),
        height: Height.FromMetric(180),
        country: CountryCode.Parse("BG"));
}
```

우선, 명료성이 향상됐음을 볼 수 있습니다.  
예를 들어, country 변수는 완전한 국가 이름 대신 국가 코드를 저장한다는 의도를 전달하기 위해 countryCode처럼 상세한 변수명을 쓸 필요가 없습니다.  
이처럼 밸류 오브젝트를 사용하면 짧은 변수 이름을 사용하더라도 의도를 명확하게 전달합니다.

둘째, 유효성 검사 로직이 밸류 오브젝트 자체에 들어 있기 때문에 값을 할당하기 전에 유효성 검사를 할 필요가 없습니다.  
게다가 밸류 오브젝트의 장점은 유효성 검사에 그치지 않습니다.  
밸류 오브젝트는 값을 조작하는 비즈니스 로직을 한곳에 모아 더 나은 응집력을 제공하며, 이렇게 모아둔 로직은 한곳에서 구현되고 쉽게 테스트할 수 있습니다.  
가장 중요한 것은 밸류 오브젝트는 바운디드 컨텍스트 코드에서 유비쿼터스 언어를 사용하게 하므로 코드에서 비즈니스 도메인의 개념을 표현하게 될 것입니다.

키(height), 전화번호(phone number), 색(color)과 같은 개념을 밸류 오브젝트로 표현할 때 구현된 시스템의 타입이 얼마나 더 풍부해지고 사용하기에 직관적인지 살펴봤습니다.  
밸류 변수를 정수 타입으로 받을 때보다 Height 밸류 오브젝트로 구현하면 의도가 명확해지고 특정 도량형에 종속되지 않습니다.  
예를 들어, Height 밸류 오브젝트는 미터법 또는 영국식 단위로 변환하거나 문자열로 표현, 또는 단위가 같이 비교하는 것 등이 쉬워집니다.

```csharp
var heightMetric = Height.Metric(180);
var heightImperial = Height.Imperial(5, 3);

var string1 = heightMetric.ToString();           // "180 cm"
var string2 = heightImperial.ToString();         // "5 feet 3 inches"
var string3 = heightMetric.ToImperial().ToString(); // "5 feet 11 inches"

var firstIsHigher = heightMetric > heightImperial; // true
```

PhoneNumber 밸류 오브젝트는 문자열 값의 파싱, 유효성 검사, 그리고 소속된 국가 또는 유선인지 휴대전화인지와 같은 다양한 전화번호 속성을 추출하는 로직을 담을 수 있습니다.

```csharp
var phone = PhoneNumber.Parse("+359877123503");
var country = phone.Country;                  // "BG"
var phoneType = phone.PhoneType;              // "MOBILE"
var isValid = PhoneNumber.IsValid("+972128266600"); // false
```

다음 예는 밸류 오브젝트의 데이터를 조작하거나 새로운 밸류 오브젝트 인스턴스를 생성할 수 있는 모든 비즈니스 로직을 모으는 등 밸류 오브젝트의 강력함을 보여줍니다.

```csharp
var red = Color.FromRGB(255, 0, 0);
var green = Color.Green;
var yellow = red.MixWith(green);
var yellowString = yellow.ToString();         // "#FFFF00"
```

앞의 예제를 보면 밸류 오브젝트 덕분에 문자열 값이 이메일인지 전화번호인지를 신경 쓸 필요가 없을 뿐 아니라 실수가 적고 더 직관적으로 객체 모델을 사용할 수 있습니다.  
이와 같이 밸류 오브젝트는 구현할 때 지켜야 할 규칙이 필요 없게 해줍니다.

### 구현

밸류 오브젝트는 불변의 객체로 구현되므로 밸류 오브젝트에 있는 필드가 하나라도 바뀌면 다른 값이 생성됩니다.  
다시 말해, 밸류 오브젝트의 필드 중 하나가 바뀌면 개념적으로 밸류 오브젝트의 다른 인스턴스가 생성됩니다.  
그러므로 다음 예제에서 MixWith 메서드에서 새로운 값을 반환하게 하듯이, 원래 인스턴스를 수정하지 않고 새로운 인스턴스를 생성해서 반환하게 할 수 있습니다.

```csharp
public class Color
{
    public readonly byte Red;
    public readonly byte Green;
    public readonly byte Blue;

    public Color(byte r, byte g, byte b)
    {
        this.Red = r;
        this.Green = g;
        this.Blue = b;
    }

    public Color MixWith(Color other)
    {
        return new Color(
            r:(byte) Math.Min(this.Red + other.Red, 255),
            g:(byte) Math.Min(this.Green + other.Green, 255),
            b:(byte) Math.Min(this.Blue + other.Blue, 255)
        );
    }
}
```

밸류 오브젝트의 동일성은 id 필드나 참조 대신 값을 기반으로 하므로 동일성 검사 함수를 오버라이드해서 적절히 구현하는 것이 중요합니다.  
다음은 C#으로 구현한 예제입니다.

```csharp
public class Color
{
    public override bool Equals(object obj)
    {
        var other = obj as Color;
        return other != null &&
            this.Red == other.Red &&
            this.Green == other.Green &&
            this.Blue == other.Blue;
    }

    public static bool operator == (Color lhs, Color rhs)
    {
        if (Object.ReferenceEquals(lhs, null))
            return Object.ReferenceEquals(rhs, null);
        return lhs.Equals(rhs);
    }

    public static bool operator != (Color lhs, Color rhs)
    {
        return !(lhs == rhs);
    }

    public override int GetHashCode()
    {
        return ToString().GetHashCode();
    }
}
```

도메인의 특정 값을 표현하는 데 원시 타입 중 하나인 문자열(String)을 사용하더라도 밸류 오브젝트로 감싸면 여전히 유용합니다.  
예를 들어, 문자열 밸류 오브젝트는 Trim(), 문자열 합치기(concatenate) 등의 일반적인 문자열 연산 외에도 도메인 특화 로직을 캡슐화할 수 있습니다.

### 밸류 오브젝트를 사용하는 경우

간단히 말해 밸류 오브젝트는 가능한 모든 경우에 사용하는 게 좋습니다.  
밸류 오브젝트는 코드의 표현성을 높여주고 분산되기 쉬운 비즈니스 로직을 한데 묶어줄 뿐만 아니라 코드를 더욱 안전하게 만들어줍니다.  
밸류 오브젝트는 불변이기 때문에 내포된 동작은 부작용과 동시성 문제가 없습니다.

비즈니스 도메인 관점에서 유용한 규칙은 다른 객체의 속성을 표현하는 도메인 요소에 밸류 오브젝트를 사용하는 것이며, 이것은 다음 절에서 논의하는 엔티티의 속성에 적용됩니다.  
앞의 예제에서 ID, 이름, 전화번호, 이메일 등을 포함하는 사람을 표현하는 데 밸류 오브젝트를 사용했습니다.  
다른 적용 예로는 상태값, 비밀번호, 그리고 명시적인 식별 필드가 필요 없이 값 자체로 식별되는 다양한 비즈니스 도메인 개념 등이 있습니다.  
특별히 중요한 적용 예로 돈(화폐 금액)이 있습니다. 원시 타입으로 표현하면 돈과 관련된 비즈니스 로직이 올바르게 수행되도록 보장하기 어렵기 때문에 중요합니다.

## 엔티티

엔티티는 밸류 오브젝트와 정반대입니다.  
엔티티는 다른 엔티티 인스턴스와 구별하기 위해 명시적인 식별 필드가 필요합니다.  
엔티티의 간단한 예로 사람(person)이 있습니다. 다음 클래스를 살펴봅시다.

```csharp
class Person
{
    public Name Name { get; set; }

    public Person(Name name)
    {
        this.Name = name;
    }
}
```

이 클래스에는 한 개의 필드가 있는데, 밸류 오브젝트인 name 필드입니다.  
그러나 다른 사람이 정확히 같은 이름을 가질 수 있기 때문에 이 설계는 최적이 아닙니다.  
물론, 이름이 같다고 해서 같은 사람은 아닙니다. 그러므로 사람을 적절히 식별하기 위해 식별 필드가 필요합니다.

```csharp
class Person
{
    public readonly PersonId Id;
    public Name Name { get; set; }

    public Person(PersonId id, Name name)
    {
        this.Id = id;
        this.Name = name;
    }
}
```

앞의 코드에서는 PersonId 타입의 Id 필드를 식별 필드로 도입했습니다.  
PersonId는 밸류 오브젝트이고 비즈니스 도메인에서 필요한 모든 기본 데이터 타입을 사용할 수 있습니다.  
예를 들어, Id 필드는 GUID, 숫자, 문자열 또는 사회 보장 번호와 같은 특정 도메인의 값일 수 있습니다.

식별 필드의 핵심 요구사항은 각 엔티티의 인스턴스마다 고유해야 한다는 것입니다.  
아주 드문 예외를 제외하고, 엔티티의 식별 필드 값은 엔티티의 생애 주기 내내 불변이어야 합니다.  
이것이 밸류 오브젝트와 엔티티의 두 번째 개념 차이입니다.

밸류 오브젝트와는 반대로, 엔티티는 불변이 아니고 변할 것으로 예상됩니다.  
엔티티와 밸류 오브젝트의 또 다른 차이점은 밸류 오브젝트는 엔티티의 속성을 설명한다는 것이며, 이번 장 초반부에서 Person 엔티티 예에서 인스턴스를 설명하는 두 개의 밸류 오브젝트인 PersonId와 Name을 볼 수 있습니다.

## 애그리게이트

**애그리게이트**는 엔티티입니다.  
즉, 명시적인 식별 필드가 필요하고 인스턴스의 생애주기 동안 상태가 변경될 것으로 예상합니다.  
하지만 애그리게이트는 단순한 엔티티가 아니라 그 이상이며, 이 패턴의 목표는 데이터의 일관성을 보호하는 데 있습니다.  
애그리게이트의 데이터는 변할 수 있기 때문에 비즈니스 로직의 불변성을 보호해야 합니다.

### 일관성 강화

애그리게이트의 상태는 변경될 수 있으므로 데이터가 손상될 수 있는 여러 경로가 있습니다.  
데이터의 일관성을 강화하려면 애그리게이트 패턴에서는 애그리게이트 주변에 명확한 경계를 설정해야 합니다.  
즉, 애그리게이트는 일관성을 강화하는 경계입니다.  
애그리게이트의 로직은 모든 변경을 검사해서 그 변경이 애그리게이트의 비즈니스 규칙을 위배하지 않는지 확인하는 것이 중요합니다.

구현 관점에서 보면 데이터 일관성은 애그리게이트의 비즈니스 로직을 통해서만 애그리게이트의 상태를 변경하게 해야 강화됩니다.  
애그리게이트의 외부의 모든 프로세스와 객체는 애그리게이트의 상태를 읽을 수만 있고 애그리게이트의 퍼블릭 인터페이스에 포함된 관련 메서드를 실행하여 상태를 변경할 수 있습니다.

애그리게이트의 퍼블릭 인터페이스로 노출된 상태 변경 메서드는 '어떤 것을 지시하는 명령을 뜻하는 의미에서 **커맨드**라고도 부릅니다.  
커맨드는 두 가지 방식으로 구현할 수 있습니다. 첫째, 퍼블릭 메서드로 구현하는 방식입니다.

```csharp
public class Ticket
{
    ...
    List<Message> _messages;
    ...

    public void AddMessage(UserId from, string body)
    {
        var message = new Message(from, body);
        _messages.Append(message);
    }
}
```

다른 방법은 커맨드의 실행에 필요한 모든 입력값을 포함하는 파라미터 객체를 전달하는 것입니다.

```csharp
public class Ticket
{
    public void Execute(AddMessage cmd)
    {
        var message = new Message(cmd.from, cmd.body);
        _messages.Append(message);
    }
}
```

어떤 방식으로 구현하건 선호도의 문제이며, 명시적으로 커맨드 구조를 정의해서 다룰지 여부의 차이입니다.  
애그리게이트의 커맨드는 입력값의 유효성을 검사하여 관련된 모든 비즈니스 규칙이 준수되도록 합니다.   
애그리게이트의 상태 변경은 오직 이러한 커맨드 메서드를 통해서만 수행됩니다.

```csharp
public ExecutionResult Escalate(TicketId id, EscalationReason reason)
{
    try
    {
        var ticket = _ticketRepository.Load(id);
        var cmd = new Escalate(reason);
        ticket.Execute(cmd);
        _ticketRepository.Save(ticket);
        return ExecutionResult.Success();
    }
    catch (ConcurrencyException ex)
    {
        return ExecutionResult.Error(ex);
    }
}
```

애그리게이트 상태의 일관성을 유지하는 것이 중요합니다.  
그러므로 여러 프로세스가 동시에 동일한 애그리게이트를 갱신하려고 할 때, 첫 번째 트랜잭션이 커밋한 변경을 나중의 트랜잭션이 은연중에 덮어쓰지 않게 해야 합니다.  
그럴 경우, 나중의 프로세스는 의사결정에 사용된 상태가 만료되었다는 것을 통지받고 오퍼레이션을 재시도해야 합니다.

그러므로 애그리게이트를 저장하는 데이터베이스에서 동시성 관리를 지원해야 합니다.  
가장 간단한 형태는 갱신할 때마다 증가하는 버전 필드를 애그리게이트에서 관리하는 것입니다.

```csharp
class Ticket
{
    TicketId _id;
    int _version;
    ...
}
```

데이터베이스에 변경을 커밋할 때 덮어쓰려는 버전이 처음 읽었던 원본의 버전과 동일한지 확인해야 합니다.  
다음의 SQL 예제를 보겠습니다.

```sql
UPDATE tickets
SET ticket_status = @new_status,
    agg_version = agg_version + 1
WHERE ticket_id=@id AND agg_version=@expected_version;
```

물론, 동시성 관리는 관계형 데이터베이스 이외에 다른 곳에서도 구현할 수 있습니다.  
또한 도큐먼트 데이터베이스는 애그리게이트를 다루는 데 많은 도움을 줍니다.  
즉, 애그리게이트의 데이터를 저장하는 데 쓰이는 데이터베이스가 동시성 관리를 지원하는지 확인하는 것이 중요합니다.

### 트랜잭션 경계

애그리게이트의 상태는 자신의 비즈니스 로직을 통해서만 수정될 수 있기 때문에 애그리게이트가 트랜잭션 경계의 역할을 합니다.  
모든 애그리게이트 상태 변경은 원자적인 단일 오퍼레이션으로 트랜잭션 처리해야 합니다.  
애그리게이트의 상태가 수정되면 모든 변경이 커밋되거나 모두 원래 상태로 돌아가야 합니다.

또한 다중 애그리게이트 트랜잭션을 지원하는 시스템 오퍼레이션은 없다고 가정합니다.  
애그리게이트의 상태 변경은 데이터베이스 트랜잭션 하나당 한 개의 애그리게이트로, 개별적으로만 커밋될 수 있습니다.

트랜잭션별로 하나의 애그리게이트 인스턴스만 갖게 제한하면 애그리게이트의 경계가 비즈니스 도메인의 불변성과 규칙을 따르도록 설계해야 합니다.  
여러 애그리게이트에서 변경을 커밋해야 한다면 이는 잘못된 트랜잭션 경계 설계이고 잘못된 애그리게이트의 경계입니다.

이는 마치 모델에 강제적 제한을 두는 것처럼 보입니다.  
동일한 트랜잭션에서 여러 객체를 수정해야 한다면 어떻게 할까요?  
이런 상황을 어떻게 패턴으로 다루는지 살펴봅시다.

### 엔티티 계층

이번 장의 초반부에 논의했듯이, 엔티티는 독립적 패턴이 아닌 애그리게이트의 일부로서만 사용합니다.  
이때 엔티티와 애그리게이트의 근본적인 차이점을 살펴보고 왜 엔티티가 중요한 도메인 모델의 구성요소가 아닌 애그리게이트의 구성요소가 되는지 알아봅니다.

애그리게이트는 엔티티와 밸류 오브젝트를 모두 담고 있습니다.  
이 요소들이 같은 모델링 경계에 묶여 도메인의 비즈니스 로직을 형성하기 때문에 **'애그리게이트'** 로 명명됐습니다.  
애그리게이트의 경계에 속한 엔티티와 밸류 오브젝트는 한데 묶여 하나의 단위로 취급됩니다.

### 다른 애그리게이트 참조하기

애그리게이트 내부의 엔티티와 밸류 오브젝트는 같은 트랜잭션 경계를 공유하기 때문에 서로 연결됩니다.  
하나의 애그리게이트가 다른 애그리게이트의 경계를 침범하면 여러 객체를 수정해 동시성 문제가 발생합니다.

경험상 애그리게이트를 가능한 한 작게 유지하고 애그리게이트의 비즈니스 로직에 따라 강력하게 일관적으로 상태를 유지할 필요가 있는 객체만 포함합니다.

```csharp
public class Ticket
{
    private UserId _customer;
    private List<ProductId> _products;
    private UserId _assignedAgent;
    private List<Message> _messages;
    ...
}
```

앞의 예제에서 티켓 애그리게이트는 경계 내에 속한 메시지의 모음을 참조합니다.  
반면에 티켓과 관련된 고객과 제품의 모음, 그리고 할당된 에이전트는 애그리게이트에 속하지 않고 ID로 참조됩니다.

외부 애그리게이트를 참조할 때 ID를 이용하는 이유는 이 같은 객체가 애그리게이트 경계에 속하지 않고 각 애그리게이트가 자신의 트랜잭션 경계를 강력하게 보장하기 위함입니다.

엔티티가 애그리게이트에 속하는지 판단하는 방법은 우선 비즈니스 로직 내에 궁극적으로 얻을 데이터를 다루는 상황이 되면 시스템의 상태를 손상시킬 수 있는지 여부를 판단한 후, 그 비즈니스 로직이 애그리게이트에 있는지 여부를 조사하는 것입니다.

### 애그리게이트 루트

앞서 봤듯이, 애그리게이트의 상태는 커맨드 중 하나를 실행해서만 수정할 수 있습니다.  
그림 6-5처럼 애그리게이트가 엔티티의 제층 구조를 대표하기 때문에 그 중 하나만 애그리게이트의 퍼블릭 인터페이스, 즉 **애그리게이트 루트**로 지정해야 합니다.

```csharp
public class Ticket
{
    ...
    List<Message> _messages;
    ...

    public void Execute(AcknowledgeMessage cmd)
    {
        var message = _messages.Where(x => x.Id == cmd.Id).First();
        message.WasRead = true;
    }
}
```

애그리게이트는 특정 메시지의 읽음 상태를 수정할 수 있는 커맨드를 노출합니다.  
비록 이 오퍼레이션은 Message 엔티티의 인스턴스를 수정하지만, 애그리게이트 루트인 Ticket을 통해서만 접근할 수 있습니다.

애그리게이트 루트의 퍼블릭 인터페이스 외에도 외부에서 애그리게이트와 커뮤니케이션 할 수 있는 다른 메커니즘이 있는데, 바로 **도메인 이벤트**입니다.

## 도메인 이벤트

**도메인 이벤트**는 비즈니스 도메인에서 일어나는 중요한 이벤트를 설명하는 메시지입니다.  
예를 들면:

- 티켓이 할당됨
- 티켓이 상부에 보고됨
- 메시지가 수신됨

도메인 이벤트는 이미 발생된 것이기 때문에 과거형으로 명명합니다.

도메인 이벤트의 목적은 비즈니스 도메인에서 일어난 일을 설명하고 이벤트와 관련된 모든 필요한 데이터를 제공하는 것입니다.  
예를 들어, 다음의 도메인 이벤트는 언제, 무슨 이유로 특정 티켓이 상부에 보고됐는지 설명합니다.

```json
{
    "ticket-id": "c9d286ff-3bca-4f57-94d4-4d4e4988674f",
    "event-id": 146,
    "event-type": "ticket-escalated",
    "escalation-reason": "missed-sla",
    "escalation-time": 1628890815
}
```

다른 대부분의 소프트웨어 엔지니어링과 마찬가지로 명명은 매우 중요하다.  
도메인 이벤트의 이름이 비즈니스 도메인에서 일어난 일을 간결하고 정확하게 반영해야 합니다.

도메인 이벤트는 애그리게이트의 퍼블릭 인터페이스의 일부입니다.  
애그리게이트는 자신의 도메인 이벤트를 발행합니다.  
그림 6-6처럼 다른 프로세스, 애그리게이트, 심지어 외부 시스템도 도메인 이벤트를 구독하고 자신만의 로직을 실행할 수도 있습니다.

```csharp
public class Ticket
{
    ...
    private List<DomainEvent> _domainEvents;
    ...

    public void Execute(RequestEscalation cmd)
    {
        if (!this.IsEscalated && this.RemainingTimePercentage <= 0)
        {
            this.IsEscalated = true;
            var escalatedEvent = new TicketEscalated(_id, cmd.Reason);
            _domainEvents.Append(escalatedEvent);
        }
    }
}
```

9장에서는 관심있는 도메인 이벤트 구독자에게 도메인 이벤트를 안정적으로 게시하는 방법에 대해 논의합니다.

### 유비쿼터스 언어

마지막으로, 애그리게이트는 유비쿼터스 언어를 사용해야 합니다.   
애그리게이트의 이름, 데이터 멤버, 동작 그리고 도메인 이벤트에 사용된 모든 용어는 모두 바운디드 컨텍스트의 유비쿼터스 언어로 명명해야 합니다.  
에반스가 말했듯이, 코드는 개발자가 다른 개발자 및 도메인 전문가와 소통하는 중요한 수단입니다.  
이제 도메인 모델의 세 번째이자 마지막 구성요소를 살펴봅시다.

## 도메인 서비스

언젠가는 애그리게이트에도 밸류 오브젝트에도 속하지 않거나 복수의 애그리게이트에 관련된 비즈니스 로직을 다루게 될 것이며, 이 경우 두 로직은 **도메인 서비스** 로직을 구현한 것을 제안합니다.

도메인 서비스는 비즈니스 로직을 구현한 상태가 없는 객체(stateless object)입니다.  
대부분의 경우 이런 로직은 어떤 계산이나 분석을 수행하기 위한 다양한 시스템 구성요소의 호출을 조율합니다.

티켓의 애그리게이트 예제로 돌아가 봅시다.  
할당된 엔지니어는 제한된 시간 내에 고객에게 솔루션을 제시해야 합니다.  
이 시간은 티켓의 데이터(우선순위와 상부 보고 상태)뿐만 아니라 에이전트의 송신 부서의 우선순위별 SLA 관련 정책, 그리고 에이전트의 스케줄(교대 스케쥴, 휴가 시간에 따라 에이전트는 응답할 수 없음)에 종속됩니다.

응답 시간을 계산하는 로직은 티켓, 할당된 에이전트의 부서, 그리고 업무 스케줄 등의 다른 출처에서 정보를 필요로 합니다.  
이런 경우가 도메인 서비스로 구현되면 이상적인 대상입니다.

```csharp
public class ResponseTimeFrameCalculationService
{
    ...
    public ResponseTimeframe CalculateAgentResponseDeadline(UserId agentId,
        Priority priority, bool escalated, DateTime startTime)
    {
        var policy = _departmentRepository.GetDepartmentPolicy(agentId);
        var maxProcTime = policy.GetMaxResponseTimeFor(priority);

        if (escalated)
        {
            maxProcTime = maxProcTime * policy.EscalationFactor;
        }

        var shifts = _departmentRepository.GetUpcomingShifts(agentId,
            startTime, startTime.Add(policy.MaxAgentResponseTime));

        return CalculateTargetTime(maxProcTime, shifts);
    }
    ...
}
```

도메인 서비스는 여러 애그리게이트의 작업을 쉽게 조율할 수 있습니다.  
그러나 한 개의 데이터베이스 트랜잭션에서 한 개의 애그리게이트 인스턴스만 수정할 수 있다고 했던 애그리게이트 패턴의 한계를 명심해야 합니다.

