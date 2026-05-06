---
title: '[아키텍처] SOLID 원칙의 세번째 L'
description: '리스코프 치환 원칙 정리하기'
pubDate: '2026-05-06'
thumbnail: './Images/solid_l.png'
draft: false
---
![SOLID L 개념 정리 이미지](./Images/solid_l.png)
# LSP 리스코프 치환 원칙

> 치환성(영어: substitutability)은 객체 지향 프로그래밍 원칙이다. 컴퓨터 프로그램에서 자료형 S가 자료형 T의 서브타입이라면 필요한 프로그램의 속성(정확성, 수행하는 업무 등)의 변경 없이 자료형 T의 객체를 자료형 S의 객체로 교체(치환)할 수 있어야 한다는 원칙이다.
> [위키백과](https://ko.wikipedia.org/wiki/%EB%A6%AC%EC%8A%A4%EC%BD%94%ED%94%84_%EC%B9%98%ED%99%98_%EC%9B%90%EC%B9%99)

리스코프 치환 원칙은 말이 어렵지만, 핵심은 단순하다.  
**부모 타입을 쓰는 코드는 자식 타입이 들어와도 깨지지 않아야 한다.**

즉, 상속을 받았다는 이유만으로 같은 타입처럼 보이게 만들었는데 실제 동작은 전혀 다르다면 LSP를 위반할 가능성이 높다.

개인적으로는 이 원칙을 이렇게 이해한다.

**상속은 기능 재사용이 아니라 정체성의 약속이다.**  
자식 클래스는 부모 클래스가 외부에 약속한 행동을 유지해야 한다.

## 부모 메서드를 자식이 막는 경우

예를 들어 게임에 `Block` 클래스가 있고, 플레이어 공격에 의해 파괴될 수 있다고 해보자.

```csharp
public class Block
{
    public virtual async UniTask DestroyBlock()
    {
        await PlayDestroyAnimation();
        await WaitDespawnDelay();
        gameObject.SetActive(false);
    }
}
```

여기에 부서지지 않는 블록이 필요해서 아래처럼 만들 수 있다.

```csharp
public class UnbreakableBlock : Block
{
    public override async UniTask DestroyBlock()
    {
        // 이 블록은 부서질 수 없다.
    }
}
```

겉으로는 자연스러워 보일 수 있다.  
하지만 `Block`을 사용하는 외부 코드는 `DestroyBlock()`을 호출하면 블록이 파괴된다고 기대한다.

```csharp
public async UniTask HitBlock(Block block)
{
    await block.DestroyBlock();
    AddScore();
}
```

이 함수에 `UnbreakableBlock`이 들어오면 문제가 생긴다.  
타입은 `Block`인데, 부모가 약속한 동작을 수행하지 않는다.  
이런 구조가 LSP 위반이다.

## 함수 이름부터 실패 가능성을 표현하기

부서질 수도 있고 안 부서질 수도 있는 동작이라면, 함수 이름부터 그 사실을 드러내는 편이 좋다.

```csharp
public interface IDestroyPolicy
{
    bool CanDestroy(Block block);
}

public sealed class DestroyablePolicy : IDestroyPolicy
{
    public bool CanDestroy(Block block) => true;
}

public sealed class UnbreakablePolicy : IDestroyPolicy
{
    public bool CanDestroy(Block block) => false;
}
```

```csharp
public class Block
{
    private readonly IDestroyPolicy _destroyPolicy;

    public Block(IDestroyPolicy destroyPolicy)
    {
        _destroyPolicy = destroyPolicy;
    }

    public async UniTask<bool> TryDestroyBlock()
    {
        if (!_destroyPolicy.CanDestroy(this))
        {
            return false;
        }

        await PlayDestroyAnimation();
        await WaitDespawnDelay();
        gameObject.SetActive(false);

        return true;
    }
}
```

이제 외부 코드는 블록 파괴가 실패할 수 있다는 사실을 타입과 함수명만 보고 알 수 있다.

```csharp
public async UniTask HitBlock(Block block)
{
    bool destroyed = await block.TryDestroyBlock();

    if (destroyed)
    {
        AddScore();
    }
}
```

이 구조에서는 `UnbreakableBlock` 같은 자식 클래스를 억지로 만들지 않아도 된다.  
파괴 가능 여부는 상속이 아니라 정책으로 주입된다.

`isDestroyable` 같은 프로퍼티를 `virtual`로 두는 방법도 가능하다.

```csharp
public class Block
{
    public virtual bool IsDestroyable => true;
}

public class UnbreakableBlock : Block
{
    public override bool IsDestroyable => false;
}
```

하지만 이 방식은 다시 상속 계층에 속성을 묶어버린다.  
블록의 정체성과 파괴 정책이 강하게 결합되는 셈이다.

프로젝트가 작다면 충분히 쓸 수 있지만, 속성이 데이터나 스테이지 규칙에 따라 자주 바뀐다면 정책 객체로 빼는 쪽이 더 유연하다.

## 상속은 정체성, 기능은 인터페이스

게임 오브젝트 계층을 아래처럼 잡았다고 해보자.

```csharp
public abstract class Actor
{
    public int Id { get; private set; }

    public virtual void Tick(float deltaTime)
    {
    }

    public void Despawn()
    {
        poolingManager.Despawn(this);
    }
}

public abstract class CharacterActor : Actor
{
    public Animator Animator { get; private set; }
    public Status Status { get; private set; }

    public void Move(Vector3 direction)
    {
        transform.position += direction;
    }

    public void Battle()
    {
    }
}

public abstract class NpcActor : CharacterActor
{
    public AISystem AI { get; private set; }
}

public sealed class PlayerActor : CharacterActor
{
    public void HandleInput(InputData input)
    {
    }
}

public abstract class StructureActor : Actor
{
    public void BuildProcess()
    {
    }
}
```

처음에는 괜찮아 보인다.  
그런데 기획에서 이런 요청이 들어올 수 있다.

"건물인데 터렛이라서 전투해야 해요."

이때 `TurretActor`를 `CharacterActor`로 만들면 이상해진다.  
터렛은 전투는 하지만 캐릭터는 아니다.  
반대로 `StructureActor`에 `Battle()`을 넣으면 모든 건물이 전투 기능을 가진 것처럼 보인다.

이런 상황에서는 상속 계층을 기능 중심으로 늘리는 대신, 상속은 정체성만 표현하고 기능은 인터페이스로 분리하는 편이 낫다.

```csharp
public abstract class Actor
{
    public int Id { get; private set; }

    public void Despawn()
    {
        poolingManager.Despawn(this);
    }
}

public abstract class CharacterActor : Actor
{
    public Animator Animator { get; private set; }
}

public abstract class NpcActor : CharacterActor
{
}

public sealed class PlayerActor : CharacterActor
{
}

public abstract class StructureActor : Actor
{
}

public sealed class TurretActor : StructureActor, IBattleActor
{
    public void BattleTick(float deltaTime)
    {
        // 터렛 전투 처리
    }
}
```

필요한 기능은 인터페이스로 쪼갠다.

```csharp
public interface IMovableActor
{
    void Move(IMoveSource source);
}

public interface IBattleActor
{
    void BattleTick(float deltaTime);
}

public interface IStatusActor
{
    ActorStatus Status { get; }
}

public interface IAIActor
{
    void AITick(float deltaTime);
}

public interface IInteractableActor
{
    void Interact();
}
```

이렇게 하면 `PlayerActor`는 캐릭터이면서 이동/전투/상태를 가질 수 있고, `TurretActor`는 건물이면서 전투만 가질 수 있다.

```csharp
public sealed class PlayerActor : CharacterActor, IMovableActor, IBattleActor, IStatusActor
{
    public ActorStatus Status { get; private set; }

    public void Move(IMoveSource source)
    {
    }

    public void BattleTick(float deltaTime)
    {
    }
}

public sealed class MerchantNpcActor : NpcActor, IInteractableActor
{
    public void Interact()
    {
        // 상점 UI 열기
    }
}
```

부모 클래스는 "무엇인가"를 표현하고, 인터페이스는 "무엇을 할 수 있는가"를 표현한다.  
이 둘을 섞기 시작하면 상속 계층이 금방 꼬인다.

## LSP를 의심해야 하는 신호

아래 상황이 반복되면 LSP를 다시 점검하는 편이 좋다.

- 자식 클래스에서 부모 메서드를 빈 구현으로 막는다.
- 자식 클래스에서 부모 메서드를 호출하면 예외를 던진다.
- 부모 타입을 받는 함수 안에서 특정 자식 타입을 검사한다.
- `virtual` 메서드가 많고, 자식마다 의미가 크게 달라진다.
- "A는 B처럼 보이지만 이 경우에는 B처럼 동작하면 안 된다"는 설명이 자주 나온다.

특히 `if (actor is TurretActor)` 같은 코드가 늘어난다면 상속 계층이 실제 도메인을 잘 표현하지 못하고 있을 가능성이 높다.

세 줄 요약

LSP는 부모 타입을 쓰는 코드가 자식 타입을 받아도 깨지지 않아야 한다는 원칙이다.  
상속은 기능 재사용이 아니라 정체성의 약속이고, 자식은 부모의 행동 계약을 지켜야 한다.  
기능 조합이 자주 바뀌는 게임 오브젝트는 깊은 상속보다 작은 인터페이스와 정책 객체로 나누는 편이 안전하다.
