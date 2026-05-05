---
title: '[아키텍쳐] SOLID 원칙의 첫번째 S'
description: 'SOLID 원칙 정리하기'
pubDate: '2026-05-01'
thumbnail: './Images/broken_S.png'
draft: false
---

![SOLID S 개념 정리 이미지](./Images/broken_S.png)

> 객체 지향 프로그래밍에서 단일 책임 원칙(single responsibility principle)이란 모든 클래스는 하나의 책임만 가지며, 클래스는 그 책임을 완전히 캡슐화해야 함을 일컫는다. 클래스가 제공하는 모든 기능은 이 책임과 주의 깊게 부합해야 한다.
> [위키백과](https://ko.wikipedia.org/wiki/%EB%8B%A8%EC%9D%BC_%EC%B1%85%EC%9E%84_%EC%9B%90%EC%B9%99)

단일 책임이라는 표현은 종종 "클래스는 기능 하나만 가져야 한다"로 오해된다.  
핵심은 기능 개수가 아니라 **변경 이유**다.  
즉, 책임은 "무엇을 하느냐"보다 "왜 바뀌느냐"를 기준으로 나누는 것이 더 정확하다.

예를 들어 `Calculator`와 `Player` 클래스가 있다고 해보자.

`Calculator`는 기본적으로 사칙연산(`+`, `-`, `*`, `/`)을 처리한다.

```csharp
public class Calculator
{
public Add(int a, int b)
public Subtract(int a, int b)
public Multiply(int a, int b)
public Divide(int a, int b)
}
```

`Player`는 이동, 전투, 애니메이션 출력 등을 처리한다고 가정해보자.

```csharp
public class Player
{
public Move();
public Combat();
public DoAnimation();
}
```

겉보기에는 둘 다 여러 메서드를 가진 클래스지만, **변경 이유** 관점에서 보면 다르다.

`Calculator`의 변경 이유는 대부분 "계산 방식의 변경"이라는 하나의 범주에 묶인다.  
사칙연산 중 하나를 수정해도 결국 같은 책임 안에서의 변경이다.

반면 `Player`는 이동 방식, 전투 공식, 애니메이션 처리 방식 등 서로 다른 이유로 자주 수정될 수 있다.  
이런 구조는 SRP 관점에서 좋지 않다.

그래서 아래처럼 책임을 분리할 수 있다.

```csharp
public class ActorMover
{
public void Move()
public void Jump()
public void Rotate()
}
```

`ActorMover`는 이동, 점프, 회전 등 여러 기능을 갖지만 모두 "Actor의 움직임"이라는 같은 책임 범주에 속한다.  
이 정도 분리는 충분히 SRP에 부합한다.

어디까지 세분화할지는 프로젝트 규모와 팀 운영 방식에 맞게 결정하면 된다.  
유니티 프로젝트 기준으로 내가 자주 쓰는 방식은 아래와 같다.  
크게 `UI`와 `게임 오브젝트(Actor)`로 나누고, 게임 오브젝트는 `MVP`, UI는 `MVVM` 흐름을 따른다.

- 게임 오브젝트

    ```csharp
    // 게임 오브젝트의 View 영역 담당 
    public class ActorView : MonoBehavior
    {
    Transform _tr;
    RigidBody _rb;
    Animator _animator;
    
    public void MoveObject(vector pos)
    {
    _tr.position = pos
    }
    }
    
    // 게임오브젝트의 Presenter 영역을 담당하며 실제 런타임중 객체의 메인을 담당한다
    public class ActorRuntime
    {
    private ActorView _view;
    
    private readonly ActorMovment _movement;
    private readonly ActorCombat _combat;
    
    public virtual void Tick()
    {
    _movement.CalcuateVelocity();
    _combat.CalculateCombat();
    }
    public virtual void FixedTick()
    {
    _view.MoveObject(_movement.NextPos());
    }
    }
    
    public class AIActorRuntime
    {
    private readonly ActorAI _ai;
    public override void Tick()
    {
    _ai.DoAI();
    base.Tick();
    }
    }
    
    //각 역할군을 담당하는 Model들 아래 부분의 세분화 정도를 프로젝트의 크기에따라 조정한다
    public class ActorMovment {}
    public class ActorCombat {}
    public class ActorAI{} 
    ```

- UI

    ```csharp
    UI의 View를 담당 ViewModel을 구독하여 실제 GUI의 변경만 담당한다.
    
    public class LoadingView
    {
    private LoadingViewModel Model;
    
    [serializeField] private Image _progressBar;
    
    public void BindModel(LoadingViewModel model)
    {
    Model = model;
    Model.Progress.Subscribe(UpdateProgress).AddTo(this);
    }
    private UpdateProgress(float progress)
    {
    _progressBar.fillAmount = progress;
    }
    }
    
    UI값 을 변경하는 모델들을 구독하여 값을 View에 전달 하는 역할
    public interface ILoadingSource
    {
    public ReadOnlyReactiveProperty<float> Progress {get;}
    }
        
    public class LoadingViewModel
    {
    private readonly ReactiveProperty<float> _progress = new(0f);
    public ReadOnlyReactiveProperty<float> Progress => _progress;
    
    public void BindSource(ILoadingSource source)
    {
    source.Progress.Subscribe(x => _progress.Value = x).AddTo(this)
    }
    }
    
    // 실제 로딩진행율을 담당하는 모델 예시
    // 씬매니저에서 씬을 로드할때 로딩진행 Progree를 기록
    public class SceneManager : ILoadingSource
    {
    private readonly ReactiveProperty<float> _loadingProgress = new(0f);
    ReadOnlyReactiveProperty<float> Progress => _loadingProgress;
    
    public asyn UniTask LoadSceneAsync()
    {
    var loadOp = UnityEngine.SceneManagement.SceneManager.LoadSceneAsync(targetSceneName, LoadSceneMode.Single);
    loadOp.allowSceneActivation = false;
    while (loadOp.progress < 0.9f)
    {
    	ct.ThrowIfCancellationRequested();
    	_loadingProgress.Value = loadOp.progress;
    	await UniTask.Yield(PlayerLoopTiming.Update, ct);
    }
    }
    ```


`partial class`와의 차이

C#의 `partial class`를 사용하면 파일은 분리할 수 있지만, 책임이 실제로 분리되는 것은 아니다.  
가장 큰 문제는 필드와 메서드가 그대로 공유된다는 점이다.  
예를 들어 `Player`를 partial로 나눠 `Movement`와 `Combat`을 분리해도, 서로의 코드에 자유롭게 접근해 값을 바꿀 수 있다.

그래서 partial class는 가능하면 Roslyn 같은 자동 코드 생성 용도로만 제한하는 편이 안전하다.  
SRP를 잘 지키면, 애초에 한 클래스가 partial이 필요할 만큼 비대해지는 상황 자체를 줄일 수 있다.

세 줄 요약

SRP는 "기능 수"가 아니라 "변경 이유"를 기준으로 클래스를 분리하는 원칙이다.  
캐릭터 이동을 수정할 때 전투나 애니메이션 로직까지 함께 흔들린다면 설계가 잘못된 신호다.  
그래서 캐릭터 관련 책임은 이동, 전투, 애니메이션처럼 변경 이유에 따라 분리하는 것이 좋다.
