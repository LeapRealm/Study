# Week 2

## 언리얼 게임 프레임워크
![01](https://github.com/LeapRealm/Study/assets/43628076/a70f4a65-e80b-4ebc-9116-2dc768137c0b)

### GameMode
전통적으로 게임모드는 게임의 규칙을 결정하는 액터로 사용되어 왔지만, Lyra에서는 더 경량화된 Experience를 도입함으로써 게임모드의 역할이 많이 퇴색되었습니다. </br>
Lyra에서는 게임모드를 월드마다 별도로 구성하지 않고 게임내에서 하나만 사용하며, 게임의 모드를 변경하고 싶을때는 Experience를 변경하는 방식을 사용하고 있습니다.

### GameState
게임 스테이트는 접속된 모든 클라이언트가 알아야 하는 정보 또는 플레이어 개개인이 아닌 전체 게임에 관련된 정보를 관리합니다. <br/>
접속된 플레이어 목록, 게임의 점수, 오픈 월드 게임에서 완료한 미션 등과 같은 게임의 전반적인 기록을 관리합니다. <br/>
Lyra에서는 게임 스테이트가 가지고 있는 ExperienceManager 컴포넌트를 통해서 Experience를 관리하고 있습니다.

### PlayerController
플레이어 컨트롤러는 게임 세계에서 현실 세계의 플레이어를 대변하며, 폰을 조종하는 역할을 담당합니다. <br/>
플레이어가 입력은 플레이어 컨트롤러를 거쳐 플레이어 컨트롤러가 빙의한 폰에 전달됩니다. <br/>
Lyra에서는 플레이어 컨트롤러가 PlayerCameraManager 컴포넌트를 가지고 있으며, 플레이어의 카메라를 관리합니다.

### PlayerState
플레이어 스테이트는 각 플레이어의 상태 정보를 관리하며, 플레이어 이름, 점수, 레벨과 같은 데이터를 관리하기 적합합니다. <br/>
Lyra에서 플레이어 스테이트는 PawnData를 가지고 있으며, PawnData는 어떤 폰을 스폰하고 빙의할지, 어떤 인풋 매핑을 사용할지, 어떤 카메라 모드를 사용할지와 같은 데이터를 가지고 있습니다.

### Pawn
폰은 플레이어나 AI가 빙의하여 제어할 수 있으며, 월드 내 플레이어나 AI에 대해서 화면상에 랜더링되는 물리적인 표현을 담당합니다. </br>
플레이어나 AI의 시각적인 모습 뿐만 아니라, 콜리전과 같은 물리적 반응과 상호작용 방식도 폰이 담당합니다. </br>
폰 중에서 걸어다닐 수 있는 능력을 지닌 폰을 캐릭터라고 합니다. 캐릭터는 CharacterMovement 컴포넌트와 Capsule 컴포넌트를 기본적으로 기자고 있습니다. </br>
Lyra에서는 폰이 PawnExtension 컴포넌트와 Camera 컴포넌트를 가지고 있으며, 각각 폰에 대한 추가적인 정보와 카메라의 속성 및 기능을 가지고 있습니다.

<br/>

## LyraGameMode::InitGame()
![02](https://github.com/LeapRealm/Study/assets/43628076/7b04e3ed-56dc-440e-a57c-b8a07cf42ed2)

> LyraGameMode.h
```cpp
#pragma once

#include "GameFramework/GameModeBase.h"
#include "LyraGameMode.generated.h"

UCLASS()
class ALyraGameMode : public AGameModeBase
{
    GENERATED_BODY()
	
public:
    ALyraGameMode(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

public:
    virtual void InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage) override;

public:
    void HandleMatchAssignmentIfNotExpectingOne();
};
```

</br>

> LyraGameMode.cpp
```cpp
#include "LyraGameMode.h"

#include "LyraExperienceManagerComponent.h"
#include "LyraClone/GameModes/LyraGameState.h"
#include "LyraClone/Player/LyraPlayerController.h"
#include "LyraClone/Player/LyraPlayerState.h"
#include "LyraClone/Character/LyraCharacter.h"

#include UE_INLINE_GENERATED_CPP_BY_NAME(LyraGameMode)

ALyraGameMode::ALyraGameMode(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    // 언리얼 게임 프레임워크 클래스들을 만든다음 기본 값으로 세팅해줍니다.
    GameStateClass = ALyraGameState::StaticClass();
    PlayerControllerClass = ALyraPlayerController::StaticClass();
    PlayerStateClass = ALyraPlayerState::StaticClass();
    DefaultPawnClass = ALyraCharacter::StaticClass();
}

void ALyraGameMode::InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage)
{
    Super::InitGame(MapName, Options, ErrorMessage);

    // GameInstance를 통해 초기화 작업이 진행되고 있는 현 프레임에는 아직 Experience가 로딩되지 않았기 때문에 Experience 처리를 진행할 수 없습니다.
    // Experience를 처리하기 위해, 한 프레임 뒤에 이벤트를 받아 처리를 이어서 진행합니다.
    GetWorld()->GetTimerManager().SetTimerForNextTick(this, &ThisClass::HandleMatchAssignmentIfNotExpectingOne);
}

void ALyraGameMode::HandleMatchAssignmentIfNotExpectingOne()
{
	
}
```

> [!NOTE]
> 언리얼 에디터를 실행하고 프로젝트 세팅에서 Default GameMode 클래스를 LyraGameMode로 변경해줘야합니다.

</br>

## GameState와 ExperienceManagerComponent
![03](https://github.com/LeapRealm/Study/assets/43628076/54a96db8-835b-479c-b4a6-c13092386d9a)

ExperienceManagerComponent는 게임 스테이트를 오너로 가지면서, Experience의 상태 정보를 가지고 있는 컴포넌트입니다.</br>
Manager라는 단어에서 유추할 수 있듯이, Experience의 로딩 상태 업데이트 및 이벤트를 관리합니다.

<br/>

> LyraGameState.h
```cpp
#pragma once

#include "GameFramework/GameStateBase.h"
#include "LyraGameState.generated.h"

class ULyraExperienceManagerComponent;

UCLASS()
class ALyraGameState : public AGameStateBase
{
    GENERATED_BODY()
	
public:
    ALyraGameState(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

public:
    UPROPERTY()
    TObjectPtr<ULyraExperienceManagerComponent> ExperienceManagerComponent;
};
```
</br>

> LyraGameState.cpp
```cpp
#include "LyraGameState.h"

#include "LyraExperienceManagerComponent.h"

#include UE_INLINE_GENERATED_CPP_BY_NAME(LyraGameState)

ALyraGameState::ALyraGameState(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    // 게임스테이트는 ExperienceManager 컴포넌트를 통해서 Experience를 관리합니다.
    ExperienceManagerComponent = CreateDefaultSubobject<ULyraExperienceManagerComponent>(TEXT("ExperienceManagerComponent"));
}
```

</br>

> LyraExperienceManagerComponent.h
```cpp
#pragma once
#include "Components/GameStateComponent.h"

#include "LyraExperienceManagerComponent.generated.h"

class ULyraExperienceDefinition;

// Experience 로딩 상태를 관리하기 위한 열거형
enum class ELyraExperienceLoadState
{
    Unloaded,
    Loading,
    Loaded,
    Deactivating,
};

DECLARE_MULTICAST_DELEGATE_OneParam(FOnLyraExperienceLoaded, const ULyraExperienceDefinition*);

UCLASS()
class ULyraExperienceManagerComponent : public UGameStateComponent
{
    GENERATED_BODY()
	
public:
    ULyraExperienceManagerComponent(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

public:
    bool IsExperienceLoaded() { return (LoadState == ELyraExperienceLoadState::Loaded) && (CurrentExperience != nullptr); }
    void CallOrRegister_OnExperienceLoaded(FOnLyraExperienceLoaded::FDelegate&& Delegate);
	
private:
    // Lyra에서는 'ReplicatedUsing'으로 선언되어있지만, 현재는 싱글플레이만 고려할 것이기 때문에 네트워크 코드를 최대한 배제하면서 진행하겠습니다.
    UPROPERTY()
    TObjectPtr<const ULyraExperienceDefinition> CurrentExperience;

    // Experience의 로딩 상태를 모니터링
    ELyraExperienceLoadState LoadState = ELyraExperienceLoadState::Unloaded;

    // Experience의 로딩이 완료된 이후, 브로드캐스팅해줄 델리게이트 변수
    FOnLyraExperienceLoaded OnExperienceLoaded;
};
```

</br>

> LyraExperienceManagerComponent.cpp
```cpp
#include "LyraExperienceManagerComponent.h"

#include UE_INLINE_GENERATED_CPP_BY_NAME(LyraExperienceManagerComponent)

ULyraExperienceManagerComponent::ULyraExperienceManagerComponent(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    
}

void ULyraExperienceManagerComponent::CallOrRegister_OnExperienceLoaded(FOnLyraExperienceLoaded::FDelegate&& Delegate)
{
    // 이미 Experience 로딩이 완료되었다면 인자로 넘어온 델리게이트를 바로 실행하고, 아니면 OnExperienceLoaded 델리게이트 변수에 바인딩합니다.
    if (IsExperienceLoaded())
    {
        Delegate.Execute(CurrentExperience);
    }
    else
    {
        // 델리게이트 객체는 캡처한 변수들을 메모리에 할당해놓기 때문에 복사 비용을 아끼기 위해서 MoveTemp를 사용하여 소유권을 넘겨줄 수 있도록 만듭니다.
        OnExperienceLoaded.Add(MoveTemp(Delegate));
    }
}
```

</br>

> [!NOTE]
> 1. GameStateBase 클래스와 GameStateComponent 클래스를 사용하기 위해서 에디터에서 GameFeatures, ModularGameplay, GameplayAbilities 플러그인을 활성화시켜야합니다.
> 2. [프로젝트이름].Build.cs 파일에서 PublicDependencyModuleNames에 GameplayTags와 ModularGameplay 모듈을 추가해야합니다.

</br>

## LyraGameMode::InitGameState()
![04](https://github.com/LeapRealm/Study/assets/43628076/21e49819-0005-46b4-9a04-53e42cfea022)

Experience가 로드되었을때 게임모드에서 추가적인 작업을 해줄 수 있도록 게임모드의 OnExperienceLoaded 함수를 ExperienceManager 컴포넌트의 델리게이트 변수에 등록합니다.
게임스테이트가 초기화되어야 ExperienceManager 컴포넌트가 존재하기 때문에 게임모드의 InitGameState 함수에서 게임스테이트가 초기화된 이후에 델리게이트 등록을 진행합니다.

</br>

> LyraGameMode.h
```cpp
virtual void InitGameState() override;
void OnExperienceLoaded(const ULyraExperienceDefinition* CurrentExperience);
```

</br>

> LyraGameMode.cpp
```cpp
void ALyraGameMode::InitGameState()
{
    Super::InitGameState();

    ULyraExperienceManagerComponent* ExperienceManagerComponent = GameState->FindComponentByClass<ULyraExperienceManagerComponent>();
    check(ExperienceManagerComponent);

    // Experience가 로드되었을 때, 게임모드의 OnExperienceLoaded 함수가 호출될 수 있도록 델리게이트를 바인딩해줍니다.
    ExperienceManagerComponent->CallOrRegister_OnExperienceLoaded(FOnLyraExperienceLoaded::FDelegate::CreateUObject(this, &ThisClass::OnExperienceLoaded));
}

void ALyraGameMode::OnExperienceLoaded(const ULyraExperienceDefinition* CurrentExperience)
{
	
}
```

</br>

## PlayerController와 PlayerState
![05](https://github.com/LeapRealm/Study/assets/43628076/d8d424b7-e6c5-46f1-b15e-be8b04ff6c15)

지금까지 게임에 대한 초기화 과정을 진행했었다면, 이제 플레이어에 대한 초기화를 시작합니다.

![06](https://github.com/LeapRealm/Study/assets/43628076/3f5c83e9-1443-435a-9d63-093ac52193c4)

게임인스턴스부터 시작해서 플레이어와 관련된 플레이어컨트롤러와 플레이어스테이트를 초기화합니다.

![07](https://github.com/LeapRealm/Study/assets/43628076/9870f83c-e31d-47b8-a3a9-4932351760fc)

플레이어스테이트는 플레이어컨트롤러가 빙의할 폰에 대한 정보가 필요하기 때문에 PawnData가 필요합니다.
하지만 PawnData는 ExperienceDefinition에 있고 아직 Experience가 로딩되지 않았기 때문에 지금 당장 세팅할 수는 없습니다.
따라서 Experience가 로드되었을 경우 이벤트를 받아서 Experience의 DefaultPawnData를 플레이어스테이트에 세팅해야합니다.
그렇게하기 위해서는 ExperienceManagerComponent의 OnExperienceLoaded에 델리게이트를 등록해야 합니다.

</br>

> LyraPlayerState.h
```cpp
#pragma once

#include "GameFramework/PlayerState.h"
#include "LyraPlayerState.generated.h"

class ULyraExperienceDefinition;
class ULyraPawnData;

UCLASS()
class ALyraPlayerState : public APlayerState
{
    GENERATED_BODY()
	
public:
    ALyraPlayerState(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

public:
    virtual void PostInitializeComponents() override;
	
public:
    void OnExperienceLoaded(const ULyraExperienceDefinition* CurrentExperience);
	
public:
    UPROPERTY()
    TObjectPtr<const ULyraPawnData> PawnData;
};
```

</br>

> LyraPlayerState.cpp
```cpp
#include "LyraPlayerState.h"

#include "GameFramework/GameStateBase.h"
#include "LyraClone/GameModes/LyraExperienceDefinition.h"
#include "LyraClone/GameModes/LyraExperienceManagerComponent.h"

#include UE_INLINE_GENERATED_CPP_BY_NAME(LyraPlayerState)

ALyraPlayerState::ALyraPlayerState(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    
}

void ALyraPlayerState::PostInitializeComponents()
{
    Super::PostInitializeComponents();

    AGameStateBase* GameState = GetWorld()->GetGameState();
    check(GameState);

    ULyraExperienceManagerComponent* ExperienceManagerComponent = GameState->FindComponentByClass<ULyraExperienceManagerComponent>();
    check(ExperienceManagerComponent);

    ExperienceManagerComponent->CallOrRegister_OnExperienceLoaded(FOnLyraExperienceLoaded::FDelegate::CreateUObject(this, &ThisClass::OnExperienceLoaded));
}

void ALyraPlayerState::OnExperienceLoaded(const ULyraExperienceDefinition* CurrentExperience)
{
	
}
```

## PawnData와 Pawn
전에 만들어둔 LyraCharacter를 스폰하기 위해서 PawnData에 PawnClass 변수를 추가합니다.

> LyraPawnData.h
```cpp
UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category="Lyra|Pawn")
TSubclassOf<APawn> PawnClass;
```

<br/>

Lyra를 실행했을때 처음 등장하는 SimpleHeroPawn를 블루프린트 클래스로 만들겠습니다. <br/>
LyraCharacter를 상속받는 블루프린트 클래스를 만들고 이름을 B_SimpleHeroPawn로 지정합니다. <br/><br/>
![08](https://github.com/LeapRealm/Study/assets/43628076/2f625898-5e8b-45fe-a7f1-fab877e4bd35)

<br/>

B_SimpleHeroPawn을 열어서 Sphere와 Cylinder StaticMesh 컴포넌트를 추가합니다. <br/><br/>
![09](https://github.com/LeapRealm/Study/assets/43628076/2b82896d-33a9-4027-953d-a52c37e65624)

<br/>

Sphere StaticMesh 컴포넌트의 위치는 (0, 0, 50), 스케일은 (0.5, 0.5, 0.5)로 설정합니다. <br/><br/>
![10](https://github.com/LeapRealm/Study/assets/43628076/32471289-340e-4813-81fd-dc0999d2fee9)

<br/>

Cylinder StaticMesh 컴포넌트의 위치는 (0, 0, -20), 스케일은 (0.5, 0.5, 0.5)로 설정합니다. <br/><br/>
![11](https://github.com/LeapRealm/Study/assets/43628076/09b88dbf-7413-4c90-8880-fad45e0a6443)

<br/>

LyraPawnData를 상속받는 데이터애셋을 만들고 이름을 DA_SimplePawnData로 지정합니다. <br/><br/>
![12](https://github.com/LeapRealm/Study/assets/43628076/d1a2c0a8-6657-44da-887b-9c4c8d5e1e98)

<br/>

DA_SimplePawnData를 열고 Pawn Class를 B_SimpleHeroPawn으로 설정합니다. <br/><br/>
![13](https://github.com/LeapRealm/Study/assets/43628076/2e6fe9ce-e9d5-471d-bb99-fefbf356dba7)

<br/>

B_LyraDefaultExperience를 열고 생성한 DA_SimplePawnData를 Default Pawn Data에 설정합니다. <br/><br/>
![14](https://github.com/LeapRealm/Study/assets/43628076/35891169-5142-47e2-8d75-7a6814c4e7d5)

<br/>

## Experience 로딩이 완료되기 전까지 폰 스폰 막기
![15](https://github.com/LeapRealm/Study/assets/43628076/2f2c7451-2e89-406a-ac02-55fa15344a69)

플레이어컨트롤러와 플레이어스테이트 생성이 끝났다면, 플레이어스타트중에 한 곳을 선택하고 디폴트 폰 클래스를 가져와서 실제로 폰을 스폰하기 시작합니다.
하지만 폰 클래스는 Experience에 있는데 아직 Experience가 로딩되지 않았기 때문에 아직 폰을 스폰해서는 안됩니다.

![16](https://github.com/LeapRealm/Study/assets/43628076/b4e850a2-3fff-4761-9245-e1dfde80aed6)

실제로 폰을 스폰하는 지점인 HandleStartNewPlayer 함수에서 Experience가 아직 로딩이 안되었다면 폰의 스폰이 안되도록 막아주는 작업이 필요합니다.

![17](https://github.com/LeapRealm/Study/assets/43628076/7ed0a408-c3e9-4309-bcb0-e641234c2491)

이후에 Experience가 로딩되었으면 RestartPlayerAtPlayerStart를 호출하여 폰이 스폰되도록 만들어줘야합니다.

<br/>

> LyraGameMode.h
```cpp
virtual void HandleStartingNewPlayer_Implementation(APlayerController* NewPlayer) override;
bool IsExperienceLoaded() const;
```

<br/>

> LyraGameMode.cpp
```cpp
void ALyraGameMode::HandleStartingNewPlayer_Implementation(APlayerController* NewPlayer)
{
    if (IsExperienceLoaded())
    {
        Super::HandleStartingNewPlayer_Implementation(NewPlayer);
    }
}

bool ALyraGameMode::IsExperienceLoaded() const
{
    check(GameState);
    ULyraExperienceManagerComponent* ExperienceManagerComponent = GameState->FindComponentByClass<ULyraExperienceManagerComponent>();
    check(ExperienceManagerComponent);

    return ExperienceManagerComponent->IsExperienceLoaded();
}
```

</br>

> [!NOTE]
> GameMode의 HandleStartingNewPlayer 함수의 선언을 보면 BlueprintNativeEvent로 선언되어있습니다. </br>
> BlueprintNativeEvent는 C++에서 함수를 구현할 예정이지만 필요하다면 블루프린트에서 재정의 가능하다는 것을 의미합니다. </br>
> 블루프린트에서 오버라이딩 되었다면 오버라이딩된 블루프린트 함수를 호출하고 아니면 C++에서 구현한 함수를 호출합니다.
