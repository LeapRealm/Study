# Week 3

## Experience 로딩
![01](https://github.com/LeapRealm/Study/assets/43628076/90f8d93f-e003-4127-83fb-9f7bb164fb0d)

<br/>

> LyraGameMode.h
```cpp
void OnMatchAssignmentGiven(FPrimaryAssetId ExperienceId);
```

<br/>

> LyraGameMode.cpp
```cpp
// 로딩할 Experience에 대해 PrimaryAssetId를 생성하여 OnMatchAssignmentGiven으로 넘겨줍니다.
void ALyraGameMode::HandleMatchAssignmentIfNotExpectingOne()
{
    FPrimaryAssetId ExperienceId;

    UWorld* World = GetWorld();
	
    if (ExperienceId.IsValid() == false)
    {
        // 일단 기본 옵션으로 B_LyraDefaultExperience를 세팅해놓습니다.
        ExperienceId = FPrimaryAssetId(FPrimaryAssetType("LyraExperienceDefinition"), FName("B_LyraDefaultExperience"));
    }
	
    OnMatchAssignmentGiven(ExperienceId);
}

// ExperienceManagerComponent를 활용하여 Experience를 로딩하기 위해서 ExperienceManagerComponent의 ServerSetCurrentExperience를 호출합니다.
void ALyraGameMode::OnMatchAssignmentGiven(FPrimaryAssetId ExperienceId)
{
    check(ExperienceId.IsValid());

    ULyraExperienceManagerComponent* ExperienceManagerComponent = GameState->FindComponentByClass<ULyraExperienceManagerComponent>();
    check(ExperienceManagerComponent);
    ExperienceManagerComponent->ServerSetCurrentExperience(ExperienceId);
}
```

<br/>

> LyraExperienceManagerComponent.h
```cpp
void ServerSetCurrentExperience(FPrimaryAssetId ExperienceId);
void StartExperienceLoad();
void OnExperienceLoadComplete();
void OnExperienceFullLoadCompleted();
```

<br/>

> LyraExperienceManagerComponent.cpp
```cpp
void ULyraExperienceManagerComponent::ServerSetCurrentExperience(FPrimaryAssetId ExperienceId)
{
    ULyraAssetManager& AssetManager = ULyraAssetManager::Get();
	
    TSubclassOf<ULyraExperienceDefinition> AssetClass;
    {
        FSoftObjectPath AssetPath = AssetManager.GetPrimaryAssetPath(ExperienceId);
        AssetClass = Cast<UClass>(AssetPath.TryLoad());
    }

    const ULyraExperienceDefinition* Experience = GetDefault<ULyraExperienceDefinition>(AssetClass);
    check(Experience != nullptr);
    check(CurrentExperience == nullptr);

    // CDO를 CurrentExperience로 설정합니다.
    CurrentExperience = Experience;

    StartExperienceLoad();
}

void ULyraExperienceManagerComponent::StartExperienceLoad()
{
    check(CurrentExperience);
    check(LoadState == ELyraExperienceLoadState::Unloaded);

    LoadState = ELyraExperienceLoadState::Loading;

    ULyraAssetManager& AssetManager = ULyraAssetManager::Get();

    TSet<FPrimaryAssetId> BundleAssetList;

    // CDO를 사용해 로딩할 애셋을 지정하고 BundleAssetList에 추가해줍니다.
    BundleAssetList.Add(CurrentExperience->GetPrimaryAssetId());

    // GameFeature를 이용해서 Experience에 바인딩된 GameFeature Plugin을 로딩시킬 Bundle 이름을 추가합니다.
    // Bundle은 로딩할 애셋의 카테고리 이름이라고 생각하면 됩니다.
    TArray<FName> BundleToLoad;
    {
        // OwnerNetMode가 NM_Standalone이면, Client와 Server 둘다 로딩에 추가됩니다.
        const ENetMode OwnerNetMode = GetOwner()->GetNetMode();
        bool bLoadClient = GIsEditor || (OwnerNetMode != NM_DedicatedServer);
        bool bLoadServer = GIsEditor || (OwnerNetMode != NM_Client);
        if (bLoadClient)
        {
            BundleToLoad.Add(UGameFeaturesSubsystemSettings::LoadStateClient);
        }
        if (bLoadServer)
        {
            BundleToLoad.Add(UGameFeaturesSubsystemSettings::LoadStateServer);
        }
    }

    FStreamableDelegate OnAssetsLoadedDelegate = FStreamableDelegate::CreateUObject(this, &ThisClass::OnExperienceLoadComplete);

    TSharedPtr<FStreamableHandle> Handle = AssetManager.ChangeBundleStateForPrimaryAssets(
        BundleAssetList.Array(),
        BundleToLoad,
        {}, false, FStreamableDelegate(), FStreamableManager::AsyncLoadHighPriority
    );
	
    if (Handle.IsValid() == false || Handle->HasLoadCompleted())
    {
        // 로딩이 완료되었으면 ExecuteDelegate를 통해 OnAssetsLoadedDelegate를 호출합니다.
        // OnAssetsLoadedDelegate는 OnExperienceLoadComplete 함수를 호출합니다.
        FStreamableHandle::ExecuteDelegate(OnAssetsLoadedDelegate);
    }
    else
    {
        Handle->BindCompleteDelegate(OnAssetsLoadedDelegate);
        Handle->BindCancelDelegate(FStreamableDelegate::CreateLambda([OnAssetsLoadedDelegate]()
        {
            OnAssetsLoadedDelegate.ExecuteIfBound();
        }));
    }

    static int32 StartExperienceLoad_FrameNumber = GFrameNumber;
}

void ULyraExperienceManagerComponent::OnExperienceLoadComplete()
{
    // StartExperienceLoad 함수의 FrameNumber와 OnExperienceLoadComplete 함수의 FrameNumber가 같습니다.
    // 따라서 두 함수는 같은 프레임에 불리고, OnExperienceLoadComplete 함수는 World의 Tick 이후에 불림을 확인할 수 있습니다.
    static int32 OnExperienceLoadComplete_FrameNumber = GFrameNumber;
	
    // 해당 함수가 StreamableDelegateDelayHelper에 의해 불립니다.
    OnExperienceFullLoadCompleted();
}

void ULyraExperienceManagerComponent::OnExperienceFullLoadCompleted()
{
    check(LoadState != ELyraExperienceLoadState::Loaded);

    LoadState = ELyraExperienceLoadState::Loaded;

    // 게임모드와 플레이어스테이트에 로드된 Experience를 전달합니다.
    OnExperienceLoaded.Broadcast(CurrentExperience);
    OnExperienceLoaded.Clear();
}
```

<br/>

> [!NOTE]
> UGameFeaturesSubsystemSettings을 사용하기 위해서는 [프로젝트이름].Build.cs의 DependencyModuleNames에 "GameFeatures" 모듈을 추가해야합니다.

<br/>

> [!NOTE]
> 멀티캐스트 델리게이트를 Broadcast 했을때 함수 호출 순서는 역순입니다.

<br/>

## CDO (Class Default Object)
- 쉽게는 블루프린트의 기본 생성자라고 보면 됩니다. 정확하게는 클래스의 초기화 값으로 생성된 UObject입니다.
- UClass의 초기화 값으로 이루어진 UObject로 객체화 된 인스턴스입니다.
- 블루프린트는 C++가 아닌 스크립트이므로 기본 생성자의 동작을 구현하기 위해 CDO를 사용하고 있습니다.
- 직렬화를 지원하기 위한 목적도 있으며, 직렬화 과정에서 CDO와 다른 부분만 직렬화해서 보냄으로써 파일 또는 네트워크를 통한 메모리/네트워크 대역폭 이점이 있습니다.

<br/>

## 블루프린트

블루프린트는 클래스 생성기(Generator)이며, 언리얼의 스크립트 오브젝트라고 볼 수 있습니다. <br/>
스크립트 오브젝트로서 클래스 정의와 클래스 멤버/이벤트 함수의 정의를 담당합니다.

클래스 생성에는 아래와 같이 두 가지 방법이 있습니다.
<br/>

![02](https://github.com/LeapRealm/Study/assets/43628076/5fd0da7c-8988-4489-87a0-c2731d33d68c)
1. C++로 정의 (UHT(Unreal Header Tool)로 생성된 UClass)
2. 블루프린트로 정의 (블루프린트로 생성된 클래스는 UBlueprintGeneratedClass를 통해 알 수 있습니다)

<br/>

## PrimaryDataAsset

UPrimaryDataAsset은 GetPrimaryAssetId() 구현과 Asset Bundle에 대한 구현이 추가된 DataAsset입니다. <br/>
해당 기능은 AssetManager를 통한 load/unload 관련 메서드 호출로 활성화 가능합니다.

GetPrimaryAssetId()를 통해 아래의 두가지 패턴 구현이 가능합니다.
1. UPrimaryDataAsset을 상속 받은 Native(C++) 클래스 정의하면, 그 클래스를 PrimaryAssetType로서 사용할 수 있습니다. <br/> (ex. ULyraExperienceDefinition)
3. UPrimaryDataAsset를 상속 받은 Blueprint는 아래의 두가지 중 하나의 방식에 따라 PrimaryAssetType이 정의됩니다.
    - 부모 클래스를 거슬러 올라가며 가장 처음으로 만난 Native(C++) 클래스
    - 가장 부모인 블루프린트 클래스 (해당 블루프린트 클래스는 UPrimaryDataAsset를 바로 상속 받습니다)

이와 같은 동작 방식을 바꾸고 싶다면, GetPrimaryAssetId는 이미 virtual로 정의되어 있고, 이를 override하면 됩니다.

<br/>

> 엔진코드 - DataAsset.cpp
```cpp

FPrimaryAssetId UPrimaryDataAsset::GetPrimaryAssetId() const
{
    // 이 부분에서 해당 객체가 CDO여야 하기 때문에 앞에서 CurrentExperience 변수를 ExperienceDefinition의 UClass를 통해 CDO로 설정했습니다.
    if (HasAnyFlags(RF_ClassDefaultObject))
    {
        UClass* BestPrimaryAssetTypeClass = nullptr;

        // 기본적으로 한단계 상위 부모부터 시작합니다.
        UClass* SearchPrimaryAssetTypeClass = GetClass()->GetSuperClass();

        // 해당 클래스가 UPrimaryDataAsset를 상속받은 C++ 클래스라면 기본 FPrimaryAssetId을 리턴합니다.
        // UPrimaryDataAsset을 상속받은 클래스를 다시 한번 상속받은 구체적인 클래스가 필요합니다.
        // UPrimaryDataAsset -> NativeClassA -> NativeClassA_Instanced (여기가 this여야 합니다)

        // CLASS_Native: Native C++ 클래스 flag (UHT에서 만든 UClass)
        // CLASS_Intrinsic: UHT로 생성 안된 Native C++ 클래스
        if (GetClass()->HasAnyClassFlags(CLASS_Native | CLASS_Intrinsic) || SearchPrimaryAssetTypeClass == UPrimaryDataAsset::StaticClass())
        {
            return FPrimaryAssetId();
        }

        bool bHasIntrinsic = UObjectProperty::StaticClass()->HasAnyClassFlags(CLASS_Intrinsic);

        // 부모부터 시작해서 C++ 클래스 또는 PrimaryDataAsset 바로 아래에 있는 블루프린트 클래스를 찾습니다.
        while (SearchPrimaryAssetTypeClass)
        {
            if (SearchPrimaryAssetTypeClass->GetSuperClass() == UPrimaryDataAsset::StaticClass())
            {
                // 부모 클래스가 UPrimaryDataAsset인 경우, 이 클래스를 리턴합니다.
                BestPrimaryAssetTypeClass = SearchPrimaryAssetTypeClass;
                break;
            }
            else if (SearchPrimaryAssetTypeClass->HasAnyClassFlags(CLASS_Native | CLASS_Intrinsic))
            {
                // 부모 클래스들 중에 첫번째로 찾은 C++ 클래스를 리턴합니다.
                BestPrimaryAssetTypeClass = SearchPrimaryAssetTypeClass;
                break;
            }
            else
            {
                SearchPrimaryAssetTypeClass = SearchPrimaryAssetTypeClass->GetSuperClass();
            }
        }

        // BestPrimaryAssetType를 찾았다면
        if (BestPrimaryAssetTypeClass)
        {
            // C++ Native Class라면 FName()으로 PrimaryAssetType 만들고, 블루프린트라면 _C를 떼기 위해 GetShortFName을 호출해줍니다.
            FName PrimaryAssetType = BestPrimaryAssetTypeClass->HasAnyClassFlags(CLASS_Native | CLASS_Intrinsic) ? BestPrimaryAssetTypeClass->GetFName() : FPackageName::GetShortFName(BestPrimaryAssetTypeClass->GetOutermost()->GetFName());

            return FPrimaryAssetId(PrimaryAssetType, FPackageName::GetShortFName(GetOutermost()->GetName()));
        }

        // BestPrimaryAssetType를 찾지 못했다면, UPrimaryDataAsset이 없으므로 기본 FPrimaryAssetId를 리턴합니다.
        return FPrimaryAssetId();
    }

    // CDO가 아니라면 클래스의 이름으로 PrimaryAssetType을 선택하고, 에셋의 이름으로 GetFName()을 선택합니다.
    return FPrimaryAssetId(GetClass()->GetFName(), GetFName());
}
```

<br/>

위의 코드에서 볼 수 있듯이, GetDefault를 활용하여 CDO를 가져온 이유는 GetPrimaryAssetId()에서 UPrimaryDataAsset을 상속받는 클래스를 찾기 위함입니다.

<br/>

## PlayerState의 OnExperienceLoaded 구현

![03](https://github.com/LeapRealm/Study/assets/43628076/61be5d05-b4ed-4b1f-9158-bf1291ac0006)
<br/>
Experience 로드가 완료되면 PawnData를 세팅하고 Pawn을 스폰하는 작업을 진행해보겠습니다.

<br/>

![04](https://github.com/LeapRealm/Study/assets/43628076/567e47fc-ab92-445b-9e62-230ea51d3208)
<br/>
먼저, Pawn을 스폰하기 전에 Experience에서 PawnData를 가져와서 세팅하는 작업을 해보겠습니다.

<br/>

> LyraPlayerState.h
```cpp
template<class T>
const T* GetPawnData() const  { return Cast<T>(PawnData); }
void SetPawnData(const ULyraPawnData* InPawnData);
```

<br/>

> LyraPlayerState.cpp
```cpp
void ALyraPlayerState::OnExperienceLoaded(const ULyraExperienceDefinition* CurrentExperience)
{
    // 데디서버를 사용하는 멀티플레이는 서버와 클라이언트가 분리되어 있지만, 싱글플레이는 사실상 서버 + 클라이언트로 볼 수 있습니다.
    // 따라서, 싱글플레이에서도 GetAuthGameMode가 정상작동하여 GameMode를 가져올 수 있습니다.
    if (ALyraGameMode* GameMode = GetWorld()->GetAuthGameMode<ALyraGameMode>())
    {
        // 자신의 Controller에 해당하는 PawnData를 가져와서 세팅합니다.
        const ULyraPawnData* NewPawnData = GameMode->GetPawnDataForController(GetOwningController());
        check(NewPawnData);

        SetPawnData(NewPawnData);
    }
}

void ALyraPlayerState::SetPawnData(const ULyraPawnData* InPawnData)
{
    check(InPawnData);

    // 이미 설정된 PawnData에 덮어쓰는것은 원하지 않기때문에 PawnData가 없을 경우에만 세팅하도록 합니다.
    check(!PawnData);

    PawnData = InPawnData;
}
```

<br/>

> LyraGameMode.h
```cpp
const ULyraPawnData* GetPawnDataForController(const AController* InController) const;
```

<br/>

> LyraGameMode.cpp
```cpp
const ULyraPawnData* ALyraGameMode::GetPawnDataForController(const AController* InController) const
{
    // PlayerState에 PawnData가 있을 경우, PlayerState에 있는 PawnData를 가져와서 리턴합니다.
    if (InController)
    {
        if (const ALyraPlayerState* LyraPS = InController->GetPlayerState<ALyraPlayerState>())
        {
            if (const ULyraPawnData* PawnData = LyraPS->GetPawnData<ULyraPawnData>())
            {
                return PawnData;
            }
        }
    }

    // 아직 PlayerState에 PawnData가 세팅되지 않은 경우, ExperienceManagerComponent의 CurrentExperience에서 PawnData를 가져와서 세팅합니다.
    check(GameState);
    ULyraExperienceManagerComponent* ExperienceManagerComponent = GameState->FindComponentByClass<ULyraExperienceManagerComponent>();
    check(ExperienceManagerComponent);

    // Experience가 로딩이 되어있다면 Experience에서 DefaultPawnData를 가져와서 리턴합니다.
    if (ExperienceManagerComponent->IsExperienceLoaded())
    {
        const ULyraExperienceDefinition* Experience = ExperienceManagerComponent->GetCurrentExperienceChecked();
        if (Experience->DefaultPawnData)
        {
            return Experience->DefaultPawnData;
        }
    }

    // 아직 Experience가 로딩이 안되었다면 nullptr를 리턴합니다.
    return nullptr;
}
```

<br/>

## GameMode의 OnExperienceLoaded 구현

![05](https://github.com/LeapRealm/Study/assets/43628076/019bfd83-2491-4057-a7c1-0c2c92418eff)
<br/>
이제 실제로 Pawn을 스폰해서 Controller가 빙의하도록 만드는 구간을 작업해보겠습니다.

<br/>

> LyraGameMode.cpp
```cpp
void ALyraGameMode::OnExperienceLoaded(const ULyraExperienceDefinition* CurrentExperience)
{
	// PlayerController를 순회합니다.
	for (FConstPlayerControllerIterator Iterator = GetWorld()->GetPlayerControllerIterator(); Iterator; ++Iterator)
	{
		APlayerController* PC = Cast<APlayerController>(*Iterator);

		// 아직 Pawn에 빙의하지 않았다면, RestartPlayer 함수를 통해 Pawn을 스폰하고 빙의합니다.
		if (PC && PC->GetPawn() == nullptr)
		{
			if (PlayerCanRestart(PC))
			{
				RestartPlayer(PC);
			}
		}
	}
}
```

<br/>

이 상태로 플레이를 해보면 게임모드의 생성자에 DefaultPawnClass로 세팅해놓은 LyraCharacter가 스폰됩니다. 하지만 지금 원하는 것은 B_SimpleHeroPawn을 스폰하는 것입니다. <br>
Pawn을 스폰할때 PawnClass를 결정하는 함수인 GetDefaultPawnClassForController 함수를 오버라이드해서 스폰하길 원하는 Pawn 클래스를 반환하도록 바꿔줘야합니다.

<br/>

> LyraGameMode.h
```cpp
virtual UClass* GetDefaultPawnClassForController_Implementation(AController* InController) override;
```

<br/>

> LyraGameMode.cpp
```cpp
UClass* ALyraGameMode::GetDefaultPawnClassForController_Implementation(AController* InController)
{
    // GetPawnDataForController 함수를 사용해서 PawnData로부터 PawnClass를 가져옵니다.
    if (const ULyraPawnData* PawnData = GetPawnDataForController(InController))
    {
        if (PawnData->PawnClass)
        {
            return PawnData->PawnClass;
        }
    }
	
    return Super::GetDefaultPawnClassForController_Implementation(InController);
}
```

<br/>

이제 게임을 실행해서 `키를 눌러 콘솔창을 열어 ToggleDebugCamera를 입력하고 디버그 카메라로 확인해보면 B_SimpleHeroPawn이 스폰된 것을 확인할 수 있습니다.
<br/><br/>
![06](https://github.com/LeapRealm/Study/assets/43628076/04e6844a-7595-477c-be65-b4b4ef039994)

<br/>

## GameplayTags

> LyraGameplayTags.h
```cpp
#pragma once

#include "GameplayTagContainer.h"

struct FLyraGameplayTags
{
public:
    static const FLyraGameplayTags& Get() { return GameplayTags; }
    static void InitializeNativeTags();

    void AddTag(FGameplayTag& OutTag, const ANSICHAR* TagName, const ANSICHAR* TagComment);
    void AddAllTags(UGameplayTagsManager& Manager);

    // 아래의 GameplayTag는 초기화 과정 단계를 의미합니다.
    // - GameInstance의 초기화 과정에 UGameFrameworkComponentManager의 RegisterInitState로 등록되어 선형적으로 업데이트됩니다.
    // - 초기화 GameplayTag들은 게임의 액터들 사이에 공유되며, GameFrameworkInitStateInterface를 상속받은 클래스는 초기화 상태를 상태머신처럼 관리 가능한 인터페이스를 제공합니다.
    FGameplayTag InitState_Spawned;
    FGameplayTag InitState_DataAvailable;
    FGameplayTag InitState_DataInitialized;
    FGameplayTag InitState_GameplayReady;

private:
	static FLyraGameplayTags GameplayTags;
};
```

<br/>

> LyraGameplayTags.cpp
```cpp
#include "LyraGameplayTags.h"

#include "GameplayTagsManager.h"

FLyraGameplayTags FLyraGameplayTags::GameplayTags;

void FLyraGameplayTags::InitializeNativeTags()
{
    UGameplayTagsManager& Manager = UGameplayTagsManager::Get();
    GameplayTags.AddAllTags(Manager);
}

void FLyraGameplayTags::AddTag(FGameplayTag& OutTag, const ANSICHAR* TagName, const ANSICHAR* TagComment)
{
    OutTag = UGameplayTagsManager::Get().AddNativeGameplayTag(FName(TagName), FString(TEXT("(Native) ")) + FString(TagComment));
}

void FLyraGameplayTags::AddAllTags(UGameplayTagsManager& Manager)
{
    AddTag(InitState_Spawned, "InitState.Spawned", "1: Actor/Component has initially spawned and can be extended");
    AddTag(InitState_DataAvailable, "InitState.DataAvailable", "2: All required data has been loaded/replicated and is ready for initialization");
    AddTag(InitState_DataInitialized, "InitState.DataInitialized", "3: The available data has been initialized for this actor/component, but it is not ready for full gameplay");
    AddTag(InitState_GameplayReady, "InitState.GameplayReady", "4: The actor/component is fully ready for active gameplay");
}
```

<br/>

> LyraAssetManager.cpp
```cpp
void ULyraAssetManager::StartInitialLoading()
{
    Super::StartInitialLoading();

    // LyraGameplayTags를 초기화합니다.
    // Lyra에서는 STARTUP_JOB() 매크로를 사용하고 있지만, 지금 당장 필요하지않기 때문에 생략하겠습니다.
    FLyraGameplayTags::InitializeNativeTags();
}
```

<br/>

에디터를 실행해서 프로젝트 세팅에서 태그 목록을 확인해보면, C++에서 추가해준 초기화관련 태그 4개가 추가된것을 확인할 수 있습니다. <br><br>
![07](https://github.com/LeapRealm/Study/assets/43628076/31e60870-c9cf-4808-9f4d-f47feef50d25)

<br/>

## GameInstance
![08](https://github.com/LeapRealm/Study/assets/43628076/3595d082-e729-4fe4-8f97-a0ad5609c61e) <br>
GameInstance의 Init 함수에서 GameFrameworkComponentManager의 InitState에 GameplayTag들을 추가시켜줘야 합니다.

<br>

GameInstance는 게임 프로세스(.exe)에서 하나만 존재하는 객체로 생각하면 됩니다. <br>
게임이 켜질때만들어지고, 게임이 꺼지기 전까지 살아있습니다. <br>
에디터 상에서는 PIE로 실행 될때마다 하나씩 생성됩니다. 즉, 에디터에서는 복수개의 GameInstance가 존재할 수 있습니다. 

<br>

> LyraGameInstance.h
```cpp
#pragma once

#include "LyraGameInstance.generated.h"

UCLASS()
class ULyraGameInstance : public UGameInstance
{
    GENERATED_BODY()
	
public:
    ULyraGameInstance(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

public:
    virtual void Init() override;
    virtual void Shutdown() override;
};
```

<br>

> LyraGameInstance.cpp
```cpp
#include "LyraGameInstance.h"

#include "Components/GameFrameworkComponentManager.h"
#include "LyraClone/LyraGameplayTags.h"

#include UE_INLINE_GENERATED_CPP_BY_NAME(LyraGameInstance)

ULyraGameInstance::ULyraGameInstance(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    
}

void ULyraGameInstance::Init()
{
    Super::Init();

    // GameplayTags 섹션에서 정의한 InitState의 GameplayTag들을 등록합니다.
    UGameFrameworkComponentManager* ComponentManager = GetSubsystem<UGameFrameworkComponentManager>(this);
    if (ensure(ComponentManager))
    {
        const FLyraGameplayTags& GameplayTags = FLyraGameplayTags::Get();

        ComponentManager->RegisterInitState(GameplayTags.InitState_Spawned, false, FGameplayTag());
        ComponentManager->RegisterInitState(GameplayTags.InitState_DataAvailable, false, GameplayTags.InitState_Spawned);
        ComponentManager->RegisterInitState(GameplayTags.InitState_DataInitialized, false, GameplayTags.InitState_DataAvailable);
        ComponentManager->RegisterInitState(GameplayTags.InitState_GameplayReady, false, GameplayTags.InitState_DataInitialized);
    }
}

void ULyraGameInstance::Shutdown()
{
    Super::Shutdown();
}
```

<br>

GameFrameworkComponentManager는 Game Feature Plugin을 사용하기 위해 설계된 GameInstance Subsystem으로 생각하면 됩니다. <br>
크게 두 가지 기능을 제공합니다.
1. Extension Handlers <br>
Game Feature가 활성화될 경우, Component와 비슷한 Action을 추가/제거/수정을 가능하게 합니다.
2. Initialization States <br>
네트워크 게임의 복잡한 Component 초기화 단계를 좀 더 깔끔하게 나누어 관리할 수 있도록하는 기능을 제공합니다.

앞서 추가한 Gameplay Tag는 두번째 기능인 Initialization States에 해당하며, Initialization State를 사용자가 정의할 수 있도록 해주는 RegisterInitState 함수를 통해 등록했습니다.

<br>

> [!NOTE]
> 언리얼 에디터를 실행하고 프로젝트 세팅에서 Game Instance Class를 LyraGameInstance로 변경해줘야합니다.
