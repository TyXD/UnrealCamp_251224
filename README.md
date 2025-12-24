***

# 📐 [언리얼엔진] 중복 코드 제거: 원뿔 시야(Cone Sight) 체크 함수 일반화 (DRY)

> ## **학습 키워드**
>
> *   `DRY Principle (Don't Repeat Yourself)`: 같은 로직을 반복해서 작성하지 말고, 하나의 공통 함수로 추출하여 관리하라는 소프트웨어 개발 원칙.
> *   `Dot Product (내적)`: 두 벡터 사이의 각도를 계산하는 데 사용되는 벡터 연산. 시야각(FOV) 판별의 핵심.
> *   `LineTrace (라인 트레이스)`: 보이지 않는 광선을 쏘아 충돌체를 감지하는 기능. 장애물 여부 확인에 필수.

> 💡 **학습 노트**
>
> 이번 포스트에서는 **Fabrication** 프로젝트의 `BlinkHunter`(우는 천사 컨셉 몬스터)를 개발하며 진행한 리팩토링 사례를 소개합니다.
> 이 몬스터는 **"플레이어가 나를 보고 있는가?"**와 **"손전등 빛이 나를 비추고 있는가?"**라는 두 가지 조건을 검사해야 합니다. 두 로직은 대상만 다를 뿐, 기하학적인 계산 방식은 **'원뿔(Cone) 형태의 시야 판별'**로 완전히 동일합니다. 이를 하나의 함수로 통합하여 코드 중복을 제거하는 과정을 다룹니다.

***

## **1. 문제 상황: 복사-붙여넣기 된 기하학 연산**

초기 구현 단계에서 `IsPlayerLookingAtMe`(플레이어 시선 체크)와 `IsExposedToFlash`(손전등 체크) 함수는 각각 50줄이 넘는 코드로 작성되었습니다. 하지만 내부를 들여다보면 로직은 100% 동일했습니다.

1.  **거리 체크**: 대상과 나 사이의 거리가 유효한가?
2.  **각도 체크**: 대상이 바라보는 방향(Forward)과 나를 향한 방향 사이의 각도가 원뿔 범위 내인가?
3.  **장애물 체크**: 사이에 벽이 없는가?

이런 중복 코드는 유지보수를 어렵게 만듭니다. 시야각 계산 방식을 수정하려면 두 함수를 모두 고쳐야 하고, 한쪽만 고치면 버그가 발생하며, 새로운 "시야 체크" 기능(예: CCTV)을 추가할 때마다 같은 코드를 또 복사해야 하는 악순환이 반복됩니다.

***

## **2. 해결책: 일반화된 함수 `CheckConeLineOfSight`**

이 공통 로직을 추출하여 **어떤 위치, 어떤 방향에서든 원뿔 시야를 검사할 수 있는 함수**로 만들었습니다.

### 📐 **핵심 로직 구현 (C++)**

```cpp
// MonsterBlinkHunter.cpp

bool AMonsterBlinkHunter::CheckConeLineOfSight(
    const FVector& SourceLocation, // 바라보는 주체의 위치 (눈, 손전등)
    const FVector& SourceForward,  // 바라보는 방향 벡터
    float MaxDistance,             // 최대 감지 거리
    float ConeAngleDegrees,        // 감지 각도 (시야각의 절반)
    AActor* IgnoredActor           // 라인 트레이스 시 무시할 액터 (바라보는 주체)
) const
{
    // 1. 거리 체크 (Distance Check)
    const float Distance = FVector::Dist(SourceLocation, GetActorLocation());
    if (Distance > MaxDistance) return false;

    // 2. 각도 체크 (Dot Product)
    // Source에서 Monster를 향하는 단위 벡터 계산
    const FVector ToMonster = (GetActorLocation() - SourceLocation).GetSafeNormal();
    
    // 내적(Dot) 계산: 두 벡터가 얼마나 일치하는가? (1: 정면, 0: 수직, -1: 반대)
    const float DotProduct = FVector::DotProduct(SourceForward, ToMonster);
    
    // 각도를 코사인 값으로 변환 (내적 값과 비교하기 위함)
    // 주의: Cos 함수는 라디안 값을 받으므로 변환 필수
    const float AngleCosine = FMath::Cos(FMath::DegreesToRadians(ConeAngleDegrees));

    // 내적 값이 기준 코사인 값보다 작으면 시야각 밖임
    if (DotProduct < AngleCosine) return false;

    // 3. 장애물 체크 (Visibility LineTrace)
    FHitResult HitResult;
    FCollisionQueryParams CollisionParams;
    CollisionParams.AddIgnoredActor(this); // 나 자신(몬스터)은 충돌 제외
    if (IgnoredActor) CollisionParams.AddIgnoredActor(IgnoredActor); // 바라보는 주체 제외

    // Source -> Monster 방향으로 레이 발사
    const bool bHit = GetWorld()->LineTraceSingleByChannel(
        HitResult, 
        SourceLocation, 
        GetActorLocation(), 
        ECC_Visibility, 
        CollisionParams
    );

    // 무언가에 맞았다면 장애물이 있다는 뜻 (단, 몬스터 본체는 Ignore했으므로)
    // 즉, bHit가 false여야 장애물 없이 뻥 뚫린 상태
    return !bHit; 
}
```

> #### 💡 **학습 TIP: Dot Product를 활용한 시야각 체크**
>
> * 벡터 내적(Dot Product)은 \(\text{A} \cdot \text{B} = |\text{A}||\text{B}|\cos\theta\) 공식을 따릅니다.
>   두 벡터가 **정규화(Normalized)**되어 있으면 내적 결과가 바로 \(\cos\theta\)가 됩니다.
> * 시야각이 60도라면, 앞쪽 중심에서 좌우 30도씩이므로 `ConeAngleDegrees = 30`으로 설정하고,  
>   `Cos(30°) ≈ 0.866`과 내적 값을 비교하는 방식입니다.
> * 이 방식은 실제 각도를 계산(`FMath::Acos`)하는 것보다 **훨씬 빠르며**, 게임 AI나 센서 시스템에서 표준으로 사용됩니다.

> #### 💡 **학습 TIP: `FCollisionQueryParams`의 활용**
>
> * `LineTrace`를 사용할 때, 시작점(플레이어 눈)이나 끝점(몬스터) 자체가 레이에 걸리는 것을 방지해야 합니다.
>   `AddIgnoredActor`를 사용하여 트레이스 과정에서 특정 액터를 투명 인간 취급하는 것이 중요합니다.
> * 여러 액터를 무시해야 한다면 `AddIgnoredActors(TArray<AActor*>)`를 사용하거나, 반복적으로 `AddIgnoredActor`를 호출할 수 있습니다.
> * **디버깅 팁**: 의도대로 동작하지 않을 때는 `CollisionParams.bTraceComplex = true`로 설정하거나, `DrawDebugLine`으로 실제 레이 경로를 시각화하면 문제를 빠르게 찾을 수 있습니다.

***

## **3. 리팩토링 결과: 간결해진 코드**

이제 복잡한 기하학 연산이 사라지고, 함수 호출 하나로 의도가 명확해졌습니다.

### **Before & After 비교**

```cpp
// [After] IsPlayerLookingAtMe 구현부
bool AMonsterBlinkHunter::IsPlayerLookingAtMe(APlayerCharacter* Player)
{
    if (!Player) return false;

    FVector PlayerViewLocation;
    FRotator PlayerViewRotation;
    // 플레이어의 카메라 위치와 회전 가져오기
    Player->GetActorEyesViewPoint(PlayerViewLocation, PlayerViewRotation);

    // 공통 함수 호출로 55줄 -> 5줄로 단축!
    return CheckConeLineOfSight(
        PlayerViewLocation,          // 위치: 플레이어 눈
        PlayerViewRotation.Vector(), // 방향: 플레이어 시선
        SightCheckDistance,          // 거리: 데이터 테이블 값
        PlayerViewAngle,             // 각도: 시야각
        Player                       // 무시: 플레이어 본신
    );
}
```

```cpp
// [After] IsExposedToSpotLight 구현부
bool AMonsterBlinkHunter::IsExposedToSpotLight(AFlashLight* FlashLight)
{
    if (!FlashLight) return false;

    return CheckConeLineOfSight(
        FlashLight->GetActorLocation(),        // 위치: 손전등
        FlashLight->GetActorForwardVector(),   // 방향: 손전등 앞
        FlashLight->GetLightDistance(),        // 거리: 손전등 사거리
        FlashLight->GetLightConeAngle(),       // 각도: 손전등 퍼짐 각
        FlashLight                             // 무시: 손전등 액터
    );
}
```

***

## **4. DRY 원칙의 실전 효과**

이 리팩토링을 통해 얻은 구체적인 이점은 다음과 같습니다:

- **코드 양 감소**: 총 110줄 이상의 중복 코드 → 하나의 20줄 함수로 통합.
- **유지보수성 향상**: 시야각 계산 로직을 수정할 때 한 곳만 고치면 모든 곳에 반영됩니다.
- **확장성 확보**: 추후 **"CCTV가 몬스터를 보고 있는가?"**, **"AI 동료가 적을 발견했는가?"** 같은 새로운 기능을 추가할 때도 `CheckConeLineOfSight` 함수를 재사용할 수 있습니다.
- **버그 위험 감소**: 한쪽 함수만 고치고 다른 쪽을 깜빡해서 생기는 불일치 문제가 원천 차단됩니다.

> #### 💡 **보너스 TIP: 과도한 추상화 주의**
>
> * DRY 원칙은 강력하지만, "비슷해 보이는 모든 코드"를 무조건 합치는 것은 오히려 독이 될 수 있습니다.
> * **진짜로 같은 개념**인지 확인하세요: 지금 예제처럼 "원뿔 시야 판별"이라는 동일한 수학적/물리적 개념을 공유하는 경우는 합치는 게 맞지만, 단순히 "코드 몇 줄이 비슷하다"는 이유만으로 억지로 합치면 나중에 한쪽만 수정하고 싶을 때 오히려 복잡해질 수 있습니다.
> * **"두 번 반복되면 참는다, 세 번 반복되면 리팩토링한다"**(Rule of Three)는 좋은 경험 법칙입니다.

***

이제 `CheckConeLineOfSight` 함수는 프로젝트의 **"시야 체크 유틸리티"**로 자리잡았고, 앞으로 비슷한 기능이 필요할 때마다 바로 가져다 쓸 수 있게 되었습니다. 이것이 바로 **DRY 원칙**의 힘입니다.

***

#언리얼엔진 #리팩토리ng #CleanCode #벡터수학 #게임개발 #C++

***
