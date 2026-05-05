---
title: '[아키텍처] SOLID 원칙의 두번째 O'
description: '개방-폐쇄 원칙 정리하기'
pubDate: '2026-05-05'
thumbnail: './Images/solid_o.png'
---

![SOLID O 개념 정리 이미지](./Images/solid_o.png)

> 개방-폐쇄 원칙(OCP, Open-Closed Principle)은 '소프트웨어 개체(클래스, 모듈, 함수 등등)는 확장에 대해 열려 있어야 하고, 수정에 대해서는 닫혀 있어야 한다'는 프로그래밍 원칙이다.
> [위키백과](https://ko.wikipedia.org/wiki/%EA%B0%9C%EB%B0%A9-%ED%8F%90%EC%87%84_%EC%9B%90%EC%B9%99)

개방-폐쇄 원칙은 기존 코드를 계속 수정하지 않고도 새로운 기능을 추가할 수 있게 만드는 원칙이다.  
이름만 보면 추상적으로 느껴지지만, 핵심은 단순하다.

**새로운 요구사항이 들어올 때, 이미 안정화된 코드는 최대한 건드리지 않고 새 코드만 추가해서 처리할 수 있어야 한다.**

여기서 "수정에 닫혀 있다"는 말은 코드를 절대 수정하지 말라는 뜻이 아니다.  
이미 검증된 흐름이나 공통 로직을 매번 뜯어고치지 않도록 구조를 잡으라는 의미에 가깝다.

예를 들어 계산기 코드를 아래처럼 만들었다고 해보자.

```csharp
public class Calculator
{
    public int Calculate(OperationType type, int a, int b)
    {
        switch (type)
        {
            case OperationType.Add:
                return a + b;
            case OperationType.Subtract:
                return a - b;
            case OperationType.Multiply:
                return a * b;
            default:
                throw new NotSupportedException();
        }
    }
}
```

처음에는 문제가 없어 보인다.  
하지만 나눗셈, 나머지, 제곱 같은 계산이 추가될 때마다 `Calculate` 함수 내부의 `switch`를 계속 수정해야 한다.

"계산기를 누가 이렇게 짜냐"라고 생각할 수 있지만, 실제 프로젝트에서는 비슷한 형태를 꽤 자주 보게 된다.  
스킬 타입, 아이템 타입, 상태 타입, UI 팝업 타입이 늘어나면서 `switch`, `if-else`가 점점 커지는 코드가 대표적이다.

게임의 스킬 실행 구조를 예로 들면 아래와 같은 코드가 생기기 쉽다.

```csharp
public class SkillManager
{
    public void ExecuteSkill(SkillData skillData)
    {
        switch (skillData.Type)
        {
            case SkillType.Projectile:
                ExecuteProjectile(skillData);
                break;
            case SkillType.Buff:
                ExecuteBuff(skillData);
                break;
            case SkillType.Area:
                ExecuteArea(skillData);
                break;
        }
    }
}
```

이 구조에서는 신규 스킬 타입이 추가될 때마다 `SkillManager`가 변경된다.  
스킬이 많아질수록 `SkillManager`는 실행 흐름을 관리하는 클래스가 아니라 모든 스킬 규칙이 몰려 있는 거대한 분기문이 된다.

OCP를 적용하면 스킬 타입별 실행 로직을 별도 모듈로 분리할 수 있다.

```csharp
public interface ISkillModule
{
    SkillType Type { get; }
    bool Execute(SkillData data);
}

public sealed class ProjectileSkillModule : ISkillModule
{
    public SkillType Type => SkillType.Projectile;

    public bool Execute(SkillData data)
    {
        // 투사체 생성 및 발사 처리
        return true;
    }
}

public sealed class BuffSkillModule : ISkillModule
{
    public SkillType Type => SkillType.Buff;

    public bool Execute(SkillData data)
    {
        // 버프 적용 처리
        return true;
    }
}
```

그리고 `SkillManager`는 구체적인 스킬 실행 방식을 몰라도 된다.

```csharp
public class SkillManager
{
    private readonly Dictionary<SkillType, ISkillModule> _modules;

    public SkillManager(IEnumerable<ISkillModule> modules)
    {
        _modules = modules.ToDictionary(module => module.Type);
    }

    public bool ExecuteSkill(SkillKey key)
    {
        SkillData skillData = SkillDataManager.Get(key);

        if (!_modules.TryGetValue(skillData.Type, out ISkillModule module))
        {
            return false;
        }

        return module.Execute(skillData);
    }
}
```

이렇게 만들면 새로운 스킬 타입이 추가될 때 `SkillManager`의 실행 로직을 수정하지 않아도 된다.  
새로운 `ISkillModule` 구현체를 추가하고 등록만 하면 된다.

물론 등록 과정까지 완전히 자동화하지 않았다면, 새 모듈을 컨테이너나 팩토리에 연결하는 코드는 필요하다.  
하지만 핵심 실행 흐름이 매번 수정되지 않는다는 점이 중요하다.

OCP를 적용할 때 조심할 점도 있다.  
무조건 인터페이스와 추상 클래스를 많이 만든다고 좋은 구조가 되는 것은 아니다.

변경 가능성이 낮은 코드까지 억지로 확장 포인트로 만들면 오히려 읽기 어려운 코드가 된다.  
반대로 타입이 계속 늘어나고 분기문이 자주 수정되는 영역이라면 OCP를 적용할 가치가 높다.

내가 기준으로 삼는 신호는 대략 이렇다.

- 같은 `switch`나 `if-else`에 새 케이스가 계속 추가된다.
- 기존 클래스 수정 때문에 이미 잘 동작하던 기능까지 다시 테스트해야 한다.
- 특정 매니저 클래스가 여러 도메인의 세부 규칙을 모두 알고 있다.
- 신규 기능 추가보다 기존 코드 영향 범위 확인에 시간이 더 많이 든다.

세 줄 요약

OCP는 기존 코드를 절대 수정하지 말라는 원칙이 아니라, 안정화된 흐름을 자주 흔들지 않도록 만드는 원칙이다.  
타입별 분기문이 계속 커진다면 인터페이스나 전략 패턴으로 확장 포인트를 분리하는 것이 좋다.  
다만 변경 가능성이 낮은 영역까지 과하게 추상화하면 복잡도만 늘어나므로, 반복적으로 바뀌는 지점을 기준으로 적용해야 한다.
