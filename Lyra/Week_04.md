# Week 4

## 전체 순서도
![01](https://github.com/LeapRealm/Study/assets/43628076/498e74c2-5b31-409b-a033-4433f2635040)

<br>

## HeroComponent와 PawnExtensionComponent의 기초적인 구현
> LyraHeroComponent.h
```cpp
#pragma once

#include "Components/GameFrameworkInitStateInterface.h"
#include "Components/PawnComponent.h"
#include "LyraHeroComponent.generated.h"

// 카메라, 입력 등 플레이어가 제어하는 시스템의 초기화를 처리하는 컴포넌트
UCLASS(Blueprintable, meta=(BlueprintSpawnableComponent))
class ULyraHeroComponent : public UPawnComponent, public IGameFrameworkInitStateInterface
{
    GENERATED_BODY()
	
public:
    ULyraHeroComponent(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());
};
```
<br/>

> LyraHeroComponent.cpp
```cpp
#include "LyraHeroComponent.h"

#include UE_INLINE_GENERATED_CPP_BY_NAME(LyraHeroComponent)

ULyraHeroComponent::ULyraHeroComponent(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    // 기본적으로 Tick를 끕니다.
    PrimaryComponentTick.bStartWithTickEnabled = false;
    PrimaryComponentTick.bCanEverTick = false;
}
```

<br/>

> LyraPawnExtensionComponent.h
```cpp
#pragma once

#include "Components/GameFrameworkInitStateInterface.h"
#include "Components/PawnComponent.h"
#include "LyraPawnExtensionComponent.generated.h"

// 초기화 전반을 관장하는 컴포넌트
UCLASS()
class ULyraPawnExtensionComponent : public UPawnComponent, public IGameFrameworkInitStateInterface
{
    GENERATED_BODY()
	
public:
    ULyraPawnExtensionComponent(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());
};
```

<br/>

> LyraPawnExtensionComponent.cpp
```cpp
#include "LyraPawnExtensionComponent.h"

#include UE_INLINE_GENERATED_CPP_BY_NAME(LyraPawnExtensionComponent)

ULyraPawnExtensionComponent::ULyraPawnExtensionComponent(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    // 기본적으로 Tick를 끕니다.
    PrimaryComponentTick.bStartWithTickEnabled = false;
    PrimaryComponentTick.bCanEverTick = false;
}
```

<br/>

## LyraCharacter에 PawnExtensionComponent 추가하기
> LyraCharacter.h
```cpp
UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category="Lyra|Character")
TObjectPtr<ULyraPawnExtensionComponent> PawnExtensionComponent;
```

<br/>

> LyraCharacter.cpp
```cpp
ALyraCharacter::ALyraCharacter(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    // 기본적으로 Tick를 끕니다.
    PrimaryActorTick.bStartWithTickEnabled = false;
    PrimaryActorTick.bCanEverTick = false;

    // PawnExtensionComponent 생성
    PawnExtensionComponent = CreateDefaultSubobject<ULyraPawnExtensionComponent>(TEXT("PawnExtensionComponent"));
}
```

<br/>

## B_SimpleHeroPawn에 HeroComponent 추가하기

![02](https://github.com/LeapRealm/Study/assets/43628076/b6c850e2-f7af-4d3d-9274-0488a8db301a)
<br/>
![03](https://github.com/LeapRealm/Study/assets/43628076/0894e156-039b-423a-b0cd-c18aaee6bddb)

<br/>

## PawnExtensionComponent::OnRegister()
> LyraPawnExtensionComponent.h
```cpp
virtual void OnRegister() override;
virtual FName GetFeatureName() const override { return NAME_ActorFeatureName; }

static const FName NAME_ActorFeatureName;
```

<br/>

> LyraPawnExtensionComponent.cpp
```cpp
// Feature Name을 정할때 Component를 빼고 Pawn Extension만 넣은 것을 확인할 수 있습니다.
const FName ULyraPawnExtensionComponent::NAME_ActorFeatureName("PawnExtension");

void ULyraPawnExtensionComponent::OnRegister()
{
    Super::OnRegister();

    // 올바른 Actor에 등록되었는지 확입합니다.
    if (GetPawn<APawn>() == nullptr)
    {
        return;
    }

    // InitState를 사용하기 위해서 GameFrameworkComponentManager에 등록합니다.
    // 등록은 상속받았던 IGameFrameworkInitStateInterface의 가상 함수인 RegisterInitStateFeature()를 활욯합니다.
    RegisterInitStateFeature();
}
```

<br/>

> 엔진 함수 - GameFrameworkInitStateInterface.cpp
```cpp
void IGameFrameworkInitStateInterface::RegisterInitStateFeature()
{
    UObject* ThisObject = Cast<UObject>(this);	// LyraPawnExtensionComponent
    AActor* MyActor = GetOwningActor();			// LyraCharacter

    // GameInstance의 Subsystem으로 추가되어있는 GameFrameworkComponentManager를 MyActor(LyraCharacter)가 속한 UWorld를 통해 가져옵니다.
    UGameFrameworkComponentManager* Manager = UGameFrameworkComponentManager::GetForActor(MyActor);

    // 지금은 간단하게 PawnComponent를 하나의 GameFeature 단위라고 생각하면 됩니다.
    // PhysicsComponent, AudioComponent 같은 것들을 생각해보면 기능 단위 Component로서 Actor에 Physics와 Audio의 Feature를 할당하는 것으로 생각하면 됩니다.
    const FName MyFeatureName = GetFeatureName();

    if (MyActor && Manager)
    {
        // RegisterFeatureImplementer 함수는 Actor(LyraCharacter)에 ThisObject(LyraPawnExtensionComponent)를 FeatureName으로 등록해줍니다.
        // 내부적으로 GameFrameworkComponentManager의 ActorFeatureMap에 의해 관리됩니다.
        Manager->RegisterFeatureImplementer(MyActor, MyFeatureName, ThisObject);
    }
}
```

<br/>

## PawnExtensionComponent::BeginPlay()
> LyraPawnExtensionComponent.h
```cpp
virtual void BeginPlay() override;
```

<br/>

> LyraPawnExtensionComponent.cpp
```cpp
void ULyraPawnExtensionComponent::BeginPlay()
{
    Super::BeginPlay();

    // FeatureName에 NAME_None을 넣으면, Actor에 등록된 모든 Feature Component의 InitState 상태를 관찰하겠다는 의미입니다.
    BindOnActorInitStateChanged(NAME_None, FGameplayTag(), false);

    // InitState_Spawned로 상태 변환을 시도합니다.
    // - TryToChangeInitState는 아래와 같이 진행됩니다.
    //   1. CanChangeInitState로 상태 변환 가능 유무 판단
    //   2. HandleChangeInitState로 내부 상태 변경
    //   3. BindOnActorInitStateChanged로 Bind된 Delegate를 조건에 맞게 호출
    ensure(TryToChangeInitState(FLyraGameplayTags::Get().InitState_Spawned));

    // ForceUpdateInitState와 같은 느낌으로 이해하면 됩니다.
    // 업데이트 가능한 상태까지 계속 업데이트를 진행합니다. (CanChangeInitState와 HandleChangeInitState 또한 진행함)
    CheckDefaultInitialization();
}
```

<br/>

> 엔진 함수 - GameFrameworkInitStateInterface.cpp
```cpp
bool IGameFrameworkInitStateInterface::TryToChangeInitState(FGameplayTag DesiredState)
{
    UObject* ThisObject = Cast<UObject>(this);
    AActor* MyActor = GetOwningActor();
    UGameFrameworkComponentManager* Manager = UGameFrameworkComponentManager::GetForActor(MyActor);
    const FName MyFeatureName = GetFeatureName();

    if (Manager == false || ThisObject == false || MyActor == false)
    {
        return false;
    }

    // 현재 Feature의 상태를 가져옵니다.
    FGameplayTag CurrentState = Manager->GetInitStateForFeature(MyActor, MyFeatureName);

    // 현재 상태가 DesiredState와 동일하면 리턴합니다. (이미 같아서 변경할 필요가 없기 때문에)
    if (CurrentState == DesiredState)
    {
        return false;
    }

    // CanChangeInitState를 통해 InitState 변경이 불가능하면 리턴합니다.
    if (CanChangeInitState(Manager, CurrentState, DesiredState) == false)
    {
        return false;
    }

    // CanChangeInitState 함수의 조건을 만족했기때문에 상태 변환을 시도합니다.
    // 상태에 따라 Feature Component에서 로직 처리를 합니다.
    HandleChangeInitState(Manager, CurrentState, DesiredState);

    // ChangeFeatureInitState 함수를 통해 상태 변화 정리(Post Event)를 진행합니다.
    return ensure(Manager->ChangeFeatureInitState(MyActor, MyFeatureName, ThisObject, DesiredState));
}

// 엔진 함수
bool UGameFrameworkComponentManager::ChangeFeatureInitState(AActor* Actor, FName FeatureName, UObject* Implementer, FGameplayTag FeatureState)
{
    if (Actor == nullptr || FeatureName.IsNone() || FeatureState.IsValid() == false)
    {
        return false;
    }

    // Actor를 통해 ActorStruct(FActorFeatureData)를 찾습니다.
    FActorFeatureData& ActorStruct = FindOrAddActorData(Actor);

    // FeatureName에 맞는 Component를 ActorStruct의 RegisteredStates를 통해 찾습니다.
    FActorFeatureState* FoundState = nullptr;
    for (FActorFeatureState& State : ActorStruct.RegisteredStates)
    {
        if (State.FeatureName == FeatureName)
        {
            FoundState = &State;
        }
    }

    // 못 찾았을 경우, 하나 만들어줍니다.
    if (FoundState == false)
    {
        FoundState = &ActorStruct.RegisteredStates.Emplace_GetRef(FeatureName);
    }

    // 현재 찾은 FeatureState의 CurrentState와 Implementer를 업데이트 해줍니다.
    FoundState->CurrentState = FeatureState;
    FoundState->Implementer = Implementer;

    // BindOnActorInitStateChange로 등록한 Delegate를 InitState 상태 변화에 따라 호출해줍니다.
    ProcessFeatureStateChange(Actor, FoundState);

    return true;
}
```

<br/>

## PawnExtensionComponent::EndPlay()
> LyraPawnExtensionComponent.h
```cpp
virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;
```

<br/>

> LyraPawnExtensionComponent.cpp
```cpp
void ULyraPawnExtensionComponent::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    // OnRegister 함수안에서 호출한 RegisterInitStateFeature 함수와 쌍을 맞춰줍니다.
    UnregisterInitStateFeature();
	
    Super::EndPlay(EndPlayReason);
}
```

<br/>

## IGameFrameworkInitStateInterface의 함수 삼총사 오버라이딩
> LyraPawnExtensionComponent.h
```cpp
virtual void OnActorInitStateChanged(const FActorInitStateChangedParams& Params) override;
virtual bool CanChangeInitState(UGameFrameworkComponentManager* Manager, FGameplayTag CurrentState, FGameplayTag DesiredState) const override;
virtual void CheckDefaultInitialization() override;
```

<br/>

> LyraPawnExtensionComponent.cpp
```cpp
void ULyraPawnExtensionComponent::OnActorInitStateChanged(const FActorInitStateChangedParams& Params)
{
    if (Params.FeatureName != NAME_ActorFeatureName)
    {
        // PawnExtensionComponent는 다른 모든 Feature Component들의 상태가 DataAvailable일때 DataInitialized로 넘어갈 수 있습니다.
        // 여기서는 다른 Feature들이 DataAvailable로 변경될 때마다 지속적으로 CheckDefaultInitialization을 호출하여 DataInitialized로 상태 변경이 가능한지 확인합니다.
        const FLyraGameplayTags& InitTags = FLyraGameplayTags::Get();
        if (Params.FeatureState == InitTags.InitState_DataAvailable)
        {
            CheckDefaultInitialization();
        }
    }
}

bool ULyraPawnExtensionComponent::CanChangeInitState(UGameFrameworkComponentManager* Manager, FGameplayTag CurrentState, FGameplayTag DesiredState) const
{
    check(Manager);

    APawn* Pawn = GetPawn<APawn>();
    const FLyraGameplayTags& InitTags = FLyraGameplayTags::Get();

    // InitState_Spawned로 초기화
    if (CurrentState.IsValid() == false && DesiredState == InitTags.InitState_Spawned)
    {
        // Pawn이 잘 세팅만 되어있으면 바로 Spawned로 넘어갑니다.
        if (Pawn)
        {
            return true;
        }
    }

    // Spawned -> DataAvailable
    if (CurrentState == InitTags.InitState_Spawned && DesiredState == InitTags.InitState_DataAvailable)
    {
        // PawnData가 없으면 변경 불가능
        if (PawnData == nullptr)
        {
            return false;
        }

        // LocallyControlled인 Pawn이 Controller가 없으면 변경 불가능
        const bool bHasAuthority = Pawn->HasAuthority();
        const bool bIsLocallyControlled = Pawn->IsLocallyControlled();
        if (bHasAuthority || bIsLocallyControlled)
        {
            if (GetController<AController>() == nullptr)
            {
                return false;
            }
        }

        return true;
    }

    // DataAvailable -> DataInitialized
    if (CurrentState == InitTags.InitState_DataAvailable && DesiredState == InitTags.InitState_DataInitialized)
    {
        // Actor에 바인드된 모든 Feature들이 DataAvailable 상태일 때 DataInitialized로 넘어감
        return Manager->HaveAllFeaturesReachedInitState(Pawn, InitTags.InitState_DataAvailable);
    }

    // DataInitialized -> GameplayReady
    if (CurrentState == InitTags.InitState_DataInitialized && DesiredState == InitTags.InitState_GameplayReady)
    {
        return true;
    }
	
    // 이도저도 아니면 변경 불가능
    return false;
}

void ULyraPawnExtensionComponent::CheckDefaultInitialization()
{
    // PawnExtensionComponent는 Feature Component들의 초기화를 관장하는 Component입니다.
    // 따라서, Actor에 바인딩된 나를 제외한 다른 모든 Feature Component들에 대해 CheckDefaultInitialization 함수를 호출해줍니다.
    // 위와 같은 기능을 IGameFrameworkInitStateInterface의 CheckDefaultInitializationForImplementers 함수가 제공합니다.
    CheckDefaultInitializationForImplementers();

    const FLyraGameplayTags& InitTags = FLyraGameplayTags::Get();

    // 사용자 정의 InitState를 직접 넘겨줘야 합니다.
    static const TArray<FGameplayTag> StateChain =
    {
        InitTags.InitState_Spawned,
        InitTags.InitState_DataAvailable,
        InitTags.InitState_DataInitialized,
        InitTags.InitState_GameplayReady,
    };

    // CanChangeInitState 함수와 HandleChangeInitState 함수 그리고 ChangeFeatureInitState 함수 호출을 통한 OnActorInitStateChanged Delegate 호출까지 진행합니다.
    // - 가능할때까지 계속 상태를 업데이트합니다.
    // - InitState에 대한 변화는 Linear(선형적)입니다. (중간에 업데이트가 멈추면 누군가가 이어서 진행시켜줘야함을 의미(chaining))
    ContinueInitStateChain(StateChain);
}
```

<br/>

## HeroComponent::OnRegister()
> LyraHeroComponent.h
```cpp
virtual void OnRegister() override;
virtual FName GetFeatureName() const override { return NAME_ActorFeatureName; }

static const FName NAME_ActorFeatureName;
```

<br/>

> LyraHeroComponent.cpp
```cpp
const FName ULyraHeroComponent::NAME_ActorFeatureName("Hero");

void ULyraHeroComponent::OnRegister()
{
    Super::OnRegister();

    // 올바른 Actor에 등록되었는지 확인합니다.
    if (GetPawn<APawn>() == nullptr)
    {
        return;
    }

    RegisterInitStateFeature();
}
```

<br/>

## HeroComponent::BeginPlay()
> LyraHeroComponent.h
```cpp
virtual void BeginPlay() override;
```

<br/>

> LyraHeroComponent.cpp
```cpp
void ULyraHeroComponent::BeginPlay()
{
    Super::BeginPlay();

    // PawnExtensionComponent를 관찰 대상으로 등록합니다.
    BindOnActorInitStateChanged(ULyraPawnExtensionComponent::NAME_ActorFeatureName, FGameplayTag(), false);

    // InitState_Spawned로 변경을 시도합니다.
    ensure(TryToChangeInitState(FLyraGameplayTags::Get().InitState_Spawned));

    // Force Update를 진행합니다.
    CheckDefaultInitialization();
}
```

<br/>

## HeroComponent::EndPlay()
> LyraHeroComponent.h
```cpp
virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;
```

<br/>

> LyraHeroComponent.cpp
```cpp
void ULyraHeroComponent::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    UnregisterInitStateFeature();
	
    Super::EndPlay(EndPlayReason);
}
```

<br/>

## IGameFrameworkInitStateInterface의 함수 삼총사 오버라이딩
> LyraHeroComponent.h
```cpp
virtual void OnActorInitStateChanged(const FActorInitStateChangedParams& Params) override;
virtual bool CanChangeInitState(UGameFrameworkComponentManager* Manager, FGameplayTag CurrentState, FGameplayTag DesiredState) const override;
virtual void CheckDefaultInitialization() override;
```

<br/>

> LyraHeroComponent.cpp
```cpp
void ULyraHeroComponent::OnActorInitStateChanged(const FActorInitStateChangedParams& Params)
{
    if (Params.FeatureName == ULyraPawnExtensionComponent::NAME_ActorFeatureName)
    {
        // PawnExtensionComponent가 DataInitialized 상태로 변경되었을 경우, HeroComponent로 DataInitialized 상태로 변경합니다.
        const FLyraGameplayTags& InitTags = FLyraGameplayTags::Get();
        if (Params.FeatureState == InitTags.InitState_DataInitialized)
        {
            CheckDefaultInitialization();
        }
    }
}

bool ULyraHeroComponent::CanChangeInitState(UGameFrameworkComponentManager* Manager, FGameplayTag CurrentState, FGameplayTag DesiredState) const
{
    check(Manager);

    APawn* Pawn = GetPawn<APawn>();
    const FLyraGameplayTags& InitTags = FLyraGameplayTags::Get();
    ALyraPlayerState* LyraPS = GetPlayerState<ALyraPlayerState>();

    // InitState_Spawned로 초기화
    if (CurrentState.IsValid() == false && DesiredState == InitTags.InitState_Spawned)
    {
        // Pawn이 잘 세팅만 되어있으면 바로 Spawned로 넘어갑니다.
        if (Pawn)
        {
            return true;
        }
    }

    // Spawned -> DataAvailable
    if (CurrentState == InitTags.InitState_Spawned && DesiredState == InitTags.InitState_DataAvailable)
    {
        // PlayerState가 없으면 변경 불가능
        if (LyraPS == nullptr)
        {
            return false;
        }

        return true;
    }

    // DataAvailable -> DataInitialized
    if (CurrentState == InitTags.InitState_DataAvailable && DesiredState == InitTags.InitState_DataInitialized)
    {
        // PawnExtensionComponent가 DataInitialized 될 때까지 기다립니다. (모든 Feature Component가 DataAvailable인 상태)
        return LyraPS && Manager->HasFeatureReachedInitState(Pawn, ULyraPawnExtensionComponent::NAME_ActorFeatureName, InitTags.InitState_DataInitialized);
    }

    // DataInitialized -> GameplayReady
    if (CurrentState == InitTags.InitState_DataInitialized && DesiredState == InitTags.InitState_GameplayReady)
    {
        return true;
    }
	
    // 이도저도 아니면 변경 불가능
    return false;
}

void ULyraHeroComponent::CheckDefaultInitialization()
{
    // Hero Feature는 Pawn Extension Feature에 종속되어 있으므로 CheckDefaultInitializationForImplementers를 호출하지 않습니다.
    const FLyraGameplayTags& InitTags = FLyraGameplayTags::Get();
	
    static const TArray<FGameplayTag> StateChain =
    {
        InitTags.InitState_Spawned,
        InitTags.InitState_DataAvailable,
        InitTags.InitState_DataInitialized,
        InitTags.InitState_GameplayReady,
    };
	
    ContinueInitStateChain(StateChain);
}
```

<br/>

## IGameFrameworkInitStateInterface::HandleChangeInitState() 오버라이딩
> LyraHeroComponent.h
```cpp
virtual void HandleChangeInitState(UGameFrameworkComponentManager* Manager, FGameplayTag CurrentState, FGameplayTag DesiredState) override;
```

<br>

> LyraHeroComponent.cpp
```cpp
void ULyraHeroComponent::HandleChangeInitState(UGameFrameworkComponentManager* Manager, FGameplayTag CurrentState, FGameplayTag DesiredState)
{
    const FLyraGameplayTags& InitTags = FLyraGameplayTags::Get();

    // DataAvailable -> DataInitialized 단계
    if (CurrentState == InitTags.InitState_DataAvailable && DesiredState == InitTags.InitState_DataInitialized)
    {
        APawn* Pawn = GetPawn<APawn>();
        ALyraPlayerState* LyraPS = GetPlayerState<ALyraPlayerState>();
        if (ensure(Pawn && LyraPS) == false)
        {
            return;
        }

        // TODO: Input과 Camera 핸들링
    }
}
```

<br>

## PawnExtensionComponent에 PawnData 세팅
> LyraPawnExtensionComponent.h
```cpp
static ULyraPawnExtensionComponent* FindPawnExtensionComponent(const AActor* Actor) { return (Actor ? Actor->FindComponentByClass<ULyraPawnExtensionComponent>() : nullptr); }

void SetPawnData(const ULyraPawnData* InPawnData);

UPROPERTY(EditInstanceOnly, Category="Lyra|Pawn")
TObjectPtr<const ULyraPawnData> PawnData;
```

<br/>

> LyraPawnExtensionComponent.cpp
```cpp
void ULyraPawnExtensionComponent::SetPawnData(const ULyraPawnData* InPawnData)
{
    // Pawn에 대해 Authority가 없을 경우에는 SetPawnData를 진행하지 않습니다.
    APawn* Pawn = GetPawnChecked<APawn>();
    if (Pawn->GetLocalRole() != ROLE_Authority)
    {
        return;
    }

    if (PawnData)
    {
        return;
    }

    PawnData = InPawnData;
}
```

<br/>

> LyraGameMode.h
```cpp
virtual APawn* SpawnDefaultPawnAtTransform_Implementation(AController* NewPlayer, const FTransform& SpawnTransform) override;
```

<br/>

> LyraGameMode.cpp
```cpp
APawn* ALyraGameMode::SpawnDefaultPawnAtTransform_Implementation(AController* NewPlayer, const FTransform& SpawnTransform)
{
    FActorSpawnParameters SpawnInfo;
    SpawnInfo.Instigator = GetInstigator();
    SpawnInfo.ObjectFlags |= RF_Transient;
    SpawnInfo.bDeferConstruction = true;

    if (UClass* PawnClass = GetDefaultPawnClassForController(NewPlayer))
    {
        if (APawn* SpawnedPawn = GetWorld()->SpawnActor<APawn>(PawnClass, SpawnTransform, SpawnInfo))
        {
            if (ULyraPawnExtensionComponent* PawnExtensionComponent = ULyraPawnExtensionComponent::FindPawnExtensionComponent(SpawnedPawn))
            {
                if (const ULyraPawnData* PawnData = GetPawnDataForController(NewPlayer))
                {
                    PawnExtensionComponent->SetPawnData(PawnData);
                }
            }
            SpawnedPawn->FinishSpawning(SpawnTransform);
            return SpawnedPawn;
        }
    }

    return nullptr;
}
```

<br/>

## 순서도 상세히 살펴보기
### PawnExtensionComponent::OnRegister()
![04](https://github.com/LeapRealm/Study/assets/43628076/07e2b333-eb9b-4514-8a42-c3381ea7b027)

<br>

### HeroComponent::OnRegister()
![05](https://github.com/LeapRealm/Study/assets/43628076/a1223d94-f024-4fca-b763-f4861bdd5002)

<br>

### PawnExtensionComponent::BeginPlay()
![06](https://github.com/LeapRealm/Study/assets/43628076/e1d9fb51-a499-4d6e-8e53-e947a76bd59e)

<br>

### HeroComponent::BeginPlay()
![07](https://github.com/LeapRealm/Study/assets/43628076/3342f7aa-c8b2-46ef-b6d3-3d29338ac84d)

<br>

### PlayerController::SetupPlayerInputComponent()
![08](https://github.com/LeapRealm/Study/assets/43628076/43ee14cf-a638-4390-b6a4-9aa7364ce15b)

<br>

> LyraCharacter.h
```cpp
virtual void SetupPlayerInputComponent(UInputComponent* PlayerInputComponent) override;
```

<br>

> LyraCharacter.cpp
```cpp
void ALyraCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    Super::SetupPlayerInputComponent(PlayerInputComponent);

    // Pawn에 Possess하고 난 뒤 Controller와 PlayerState가 준비되었으니 InitState를 계속 진행합니다.
    PawnExtensionComponent->SetupPlayerInputComponent();
}
```

<br>

> LyraPawnExtensionComponent.h
```cpp
void SetupPlayerInputComponent();
```

<br>

> LyraPawnExtensionComponent.cpp
```cpp
void ULyraPawnExtensionComponent::SetupPlayerInputComponent()
{
    // ForceUpdate로 다시 InitState 상태 변환을 시작합니다. (연결 고리)
    CheckDefaultInitialization();
}
```

<br>

### HeroComponent에서 InitState_Spawned -> InitState_DataAvailable
![09](https://github.com/LeapRealm/Study/assets/43628076/56a46484-f8de-4a36-a567-d911b9aef5cf)

<br>

### PawnExtensionComponent와 HeroComponent에서 최종적인 InitState_GameplayReady까지 도달
![10](https://github.com/LeapRealm/Study/assets/43628076/29e4c0a3-5a7c-4c0b-aaaf-0b8276608e07)

<br>

## PlayerCameraManager 구현
> LyraPlayerCameraManager.h
```cpp
#pragma once

#include "LyraPlayerCameraManager.generated.h"

#define LYRA_CAMERA_DEFAULT_FOV (80.0f)
#define LYRA_CAMERA_DEFAULT_PITCH_MIN (-89.0f)
#define LYRA_CAMERA_DEFAULT_PITCH_MAX (89.0f)

UCLASS()
class ALyraPlayerCameraManager : public APlayerCameraManager
{
    GENERATED_BODY()
	
public:
    ALyraPlayerCameraManager(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());
};
```

<br>

> LyraPlayerCameraManager.cpp
```cpp
#include "LyraPlayerCameraManager.h"

#include UE_INLINE_GENERATED_CPP_BY_NAME(LyraPlayerCameraManager)

ALyraPlayerCameraManager::ALyraPlayerCameraManager(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    DefaultFOV = LYRA_CAMERA_DEFAULT_FOV;
    ViewPitchMin = LYRA_CAMERA_DEFAULT_PITCH_MIN;
    ViewPitchMax = LYRA_CAMERA_DEFAULT_PITCH_MAX;
}
```

<br>

## PlayerCameraManager를 PlayerController에서 세팅
> LyraPlayerController.cpp
```cpp
ALyraPlayerController::ALyraPlayerController(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    PlayerCameraManagerClass = ALyraPlayerCameraManager::StaticClass();
}
```

<br>

## CameraComponent 기본 구현
> LyraCameraComponent.h
```cpp
#pragma once

#include "Camera/CameraComponent.h"
#include "LyraCameraComponent.generated.h"

UCLASS()
class ULyraCameraComponent : public UCameraComponent
{
    GENERATED_BODY()
	
public:
    ULyraCameraComponent(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());
```

<br>

> LyraCameraComponent.cpp
```cpp
ULyraCameraComponent::ULyraCameraComponent(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer), CameraModeStack(nullptr)
{
    
}
```

<br>

## LyraCharacter에서 LyraCameraComponent 생성
> LyraCharacter.h
```cpp
UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category="Lyra|Character")
TObjectPtr<ULyraCameraComponent> CameraComponent;
```

<br>

> LyraCharacter.cpp
```cpp
ALyraCharacter::ALyraCharacter(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    // 이전에 작성한 코드...

    CameraComponent = CreateDefaultSubobject<ULyraCameraComponent>(TEXT("CameraComponent"));
    CameraComponent->SetRelativeLocation(FVector(-300.f, 0.f, 75.f));
}
```

<br>

## LyraCameraMode와 LyraCameraModeStack 구현
> LyraCameraMode.h
```cpp
#pragma once

#include "LyraCameraMode.generated.h"

UCLASS(Abstract, NotBlueprintable)
class ULyraCameraMode : public UObject
{
    GENERATED_BODY()

public:
    ULyraCameraMode(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());	
};

UCLASS()
class ULyraCameraModeStack : public UObject
{
    GENERATED_BODY()
	
public:
    ULyraCameraModeStack(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

public:
    // 생성된 복수개의 CameraMode를 관리
    UPROPERTY()
    TArray<TObjectPtr<ULyraCameraMode>> CameraModeInstances;

    // Camera Matrix Blend 업데이트 진행 스택
    UPROPERTY()
    TArray<TObjectPtr<ULyraCameraMode>> CameraModeStack;
};
```

<br>

> LyraCameraMode.cpp
```cpp
#include "LyraCameraMode.h"

#include UE_INLINE_GENERATED_CPP_BY_NAME(LyraCameraMode)

ULyraCameraMode::ULyraCameraMode(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
	
}

ULyraCameraModeStack::ULyraCameraModeStack(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    
}
```

<br>

## LyraCameraComponent에서 CameraMode와 관련된 멤버변수 추가
> LyraCameraComponent.h
```cpp
DECLARE_DELEGATE_RetVal(TSubclassOf<ULyraCameraMode>, FLyraCameraModeDelegate);

// 카메라의 블랜딩 기능을 지원하는 스택
UPROPERTY()
TObjectPtr<ULyraCameraModeStack> CameraModeStack;

// 현재 CameraMode를 가져오는 Delegate
FLyraCameraModeDelegate DetermineCameraModeDelegate;
```

<br>

## LyraCameraComponent에서 CameraModeStack 생성
> LyraCameraComponent.h
```cpp
virtual void OnRegister() override;
```

<br>

> LyraCameraComponent.cpp
```cpp
void ULyraCameraComponent::OnRegister()
{
    Super::OnRegister();

    if (CameraModeStack == nullptr)
    {
        // BeginPlay와 같은 초기화가 딱히 필요없는 객체이기 때문에 NewObject로 할당합니다.
        CameraModeStack = NewObject<ULyraCameraModeStack>(this);
    }
}
```

<br>

## Camera Update 흐름
### 전체 순서도
![11](https://github.com/LeapRealm/Study/assets/43628076/e1015a50-a6cb-4eb2-ac06-20dc614362a5)

<br>

### CameraComponent의 Update 시점
![12](https://github.com/LeapRealm/Study/assets/43628076/6bbf2863-778c-4121-a284-b190eefd56e1)
CameraComponent는 SceneComponent임에도 불구하고, 다른 SceneComponent와는 Update 시점이 다릅니다. 
일반적인 SceneComponent는 월드 내 레벨에서 Update가 진행되지만, CameraComponent는 PlayerController에서 시작해서 CameraManager를 거쳐 CameraComponent를 순회하며 Update가 진행됩니다.

<br>

### CameraComponent::GetCameraView()
![13](https://github.com/LeapRealm/Study/assets/43628076/ef2b4bf4-8298-4439-9cda-84644f1b0709)
CameraComponent의 틱(Tick) 업데이트는 GetCameraView를 통해 진행됩니다.

<br>

> LyraCameraComponent.h
```cpp
virtual void GetCameraView(float DeltaTime, FMinimalViewInfo& DesiredView) override;
void UpdateCameraMode();

static ULyraCameraComponent* FindCameraComponent(const AActor* Actor) { return (Actor ? Actor->FindComponentByClass<ULyraCameraComponent>() : nullptr); }
```

<br>

> LyraCameraComponent.cpp
```cpp
void ULyraCameraComponent::GetCameraView(float DeltaTime, FMinimalViewInfo& DesiredView)
{
    check(CameraModeStack);

    UpdateCameraMode();
}

void ULyraCameraComponent::UpdateCameraMode()
{
    check(CameraModeStack);

    // DetermineCameraModeDelegate는 CameraMode Class를 반환합니다.
    // 해당 delegate에는 HeroComponent의 멤버 함수가 바인딩되어 있습니다.
    if (DetermineCameraModeDelegate.IsBound())
    {
        if (const TSubclassOf<ULyraCameraMode> CameraMode = DetermineCameraModeDelegate.Execute())
        {
            // CameraModeStack->PushCameraMode(CameraMode);
        }
    }
}
```

<br>

### HeroComponent에서 DetermineCameraModeDelegate 바인딩
![14](https://github.com/LeapRealm/Study/assets/43628076/b6e65ae1-9c02-4155-9ebe-132d93fe329d)

<br>

> LyraHeroComponent.h
```cpp
TSubclassOf<ULyraCameraMode> DetermineCameraMode() const;
```

<br>

> LyraHeroComponent.cpp
```cpp
TSubclassOf<ULyraCameraMode> ULyraHeroComponent::DetermineCameraMode() const
{
    const APawn* Pawn = GetPawn<APawn>();
    if (Pawn == nullptr)
    {
        return nullptr;
    }

    if (ULyraPawnExtensionComponent* PawnExtensionComponent = ULyraPawnExtensionComponent::FindPawnExtensionComponent(Pawn))
    {
        if (const ULyraPawnData* PawnData = PawnExtensionComponent->GetPawnData<ULyraPawnData>())
        {
            return PawnData->DefaultCameraMode;
        }
    }

    return nullptr;
}
```

<br>

> LyraPawnExtensionComponent.h
```cpp
template<class T>
const T* GetPawnData() const { return Cast<T>(PawnData); }
```

<br>

> LyraPawnData.h
```cpp
UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category="Lyra|Camera")
TSubclassOf<ULyraCameraMode> DefaultCameraMode;
```

<br>

> LyraHeroComponent.cpp
```cpp
void ULyraHeroComponent::HandleChangeInitState(UGameFrameworkComponentManager* Manager, FGameplayTag CurrentState, FGameplayTag DesiredState)
{
    const FLyraGameplayTags& InitTags = FLyraGameplayTags::Get();

    // DataAvailable -> DataInitialized 단계
    if (CurrentState == InitTags.InitState_DataAvailable && DesiredState == InitTags.InitState_DataInitialized)
    {
        APawn* Pawn = GetPawn<APawn>();
        ALyraPlayerState* LyraPS = GetPlayerState<ALyraPlayerState>();
        if (ensure(Pawn && LyraPS) == false)
        {
            return;
        }

        // TODO: Input 핸들링

        const bool bIsLocallyControlled = Pawn->IsLocallyControlled();
        const ULyraPawnData* PawnData = nullptr;
        if (ULyraPawnExtensionComponent* PawnExtensionComponent = ULyraPawnExtensionComponent::FindPawnExtensionComponent(Pawn))
        {
            PawnData = PawnExtensionComponent->GetPawnData<ULyraPawnData>();
        }

        if (bIsLocallyControlled && PawnData)
        {
            // 현재 LyraCharacter에 Attach된 CameraComponent를 찾습니다.
            if (ULyraCameraComponent* CameraComponent = ULyraCameraComponent::FindCameraComponent(Pawn))
            {
                CameraComponent->DetermineCameraModeDelegate.BindUObject(this, &ThisClass::DetermineCameraMode);
            }
        }
    }
}
```

<br>

## CameraMode_ThirdPerson 구현
> LyraCameraMode_ThirdPerson.h
```cpp
#pragma once

#include "LyraCameraMode.h"
#include "LyraCameraMode_ThirdPerson.generated.h"

UCLASS(Abstract, Blueprintable)
class ULyraCameraMode_ThirdPerson : public ULyraCameraMode
{
    GENERATED_BODY()
	
public:
    ULyraCameraMode_ThirdPerson(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());
};
```

<br>

> LyraCameraMode_ThirdPerson.cpp
```cpp
#include "LyraCameraMode_ThirdPerson.h"

#include UE_INLINE_GENERATED_CPP_BY_NAME(LyraCameraMode_ThirdPerson)

ULyraCameraMode_ThirdPerson::ULyraCameraMode_ThirdPerson(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    
}
```

<br>

## CM_ThirdPerson 애셋 생성
![15](https://github.com/LeapRealm/Study/assets/43628076/eb3c5395-b523-442b-9c2f-906cf818c910)
![16](https://github.com/LeapRealm/Study/assets/43628076/e638102a-245a-4b9e-aaf8-247e4df5878a)

<br>

## DA_SimplePawnData의 Default Camera Mode 설정
![17](https://github.com/LeapRealm/Study/assets/43628076/05c6590f-c619-4b5d-90aa-b8017f7d304d)
![18](https://github.com/LeapRealm/Study/assets/43628076/ae5172ec-b974-4bf6-b019-3e923e31e04f)
