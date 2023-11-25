# Week 1

## Module
- 언리얼에서 다양한 기능들을 잘 관리하기 위해서 컴포넌트 단위로 나누어서 만들고 그것들을 합쳐서 관리하는 방법을 채택하고 있습니다.
- 그 구성요소에는 크게 3가지 프로젝트(uproject), 플러그인(uplugin), 모듈(Module)이 있습니다
- 모듈을 컴파일하면 1대1 매칭되어 dll같은 공유 라이브러리 파일이 만들어지기 떄문에, 모듈은 하나의 dll이라고 생각할 수 있습니다. 또한, .h/.cpp 를 묶는 하나의 최소 단위로 생각할 수 있습니다.
- 모듈은 단독으로 언리얼에서 구성할 수 없으며, Plugin 혹은 Project에 종속되어 활성화 되어야 합니다.
- 플러그인은 복수 개의 모듈로 구성되며, 복수 개의 플러그인을 참조할 수 있습니다. 또한, 애셋을 담을 수 있는 독립적인 Content 폴더를 가질 수 있으며 이러한 경우 모듈이 없어도 됩니다. 
- 프로젝트는 복수 개의 모듈로 구성되며, 복수 개의 플러그인을 참조할 수 있을 뿐만 아니라 포함할 수도 있습니다.

![image](https://github.com/LeapRealm/Study/assets/43628076/9615242e-ce8a-480f-b28e-96b460cfecbc)
![image](https://github.com/LeapRealm/Study/assets/43628076/de55c868-81f2-4ab2-81d8-d54363d70c03)

<br/>

LyraStarterGame의 LyraGameModule.cpp를 보면 엔진에서 제공하는 기본 모듈(FDefaultGameMoudleImpl)을 그대로 사용하는 것이 아니라 FDefaultGameMouleImpl를 상속받은 자신만의 모듈을 만들어서 사용하고 있습니다. 그 이유는 StartupModule과 ShutdownModule과 같은 가상함수들을 재정의해서 원하는대로 변경하기 위해서입니다. 

<br/>

> LyraClone.cpp
```cpp
#include "LyraClone.h"
#include "Modules/ModuleManager.h"

class FLyraGameModule : public FDefaultGameModuleImpl
{
public:
	virtual void StartupModule() override;
	virtual void ShutdownModule() override;
};

void FLyraGameModule::StartupModule()
{
	
}

void FLyraGameModule::ShutdownModule()
{
	
}

IMPLEMENT_PRIMARY_GAME_MODULE(FLyraGameModule, LyraClone, "LyraClone");
```

<br/>

## LogChannels

언리얼에서 로그를 출력할때 UE_LOG 매크로를 사용합니다. UE_LOG 매크로의 인자로 카테고리를 넣어줘야하는데 이는 수많은 로그들 중에서 원하는 로그들만 필터링해서 보기 위해 필요한 기능입니다. 

카테고리를 각각의 cpp파일에 만들 수도 있고 프로젝트 내에서 통합적으로 관리할 수도 있는데, Lyra에서는 통합적으로 관리하는 방법을 선택했습니다. 

<br/>

> LyraLogChannels.h
```cpp
#pragma once

// 로그 카테고리 이름은 LogLyra, 런타임 로그 수준은 Log, 컴파일타임 로그 수준은 All로 설정했습니다. 
// 로그 수준을 통해서 로그를 에디터에 출력할지, 파일에 출력할지, 아니면 둘 다 출력할지 등을 정할 수 있습니다. 
// 로그 수준에는 Log, Warning, Error, Fatal 등이 있으며 뒤로 갈 수록 로그 수준이 높아지며, Fatal은 크래시를 발생시킵니다.
LYRACLONE_API DECLARE_LOG_CATEGORY_EXTERN(LogLyra, Log, All)
```
<br/>

> LyraLogChannels.cpp
```cpp
#include "LyraLogChannels.h"

DEFINE_LOG_CATEGORY(LogLyra)
```

<br/>

## AssetManager

AssetManager는 애셋을 로드하고 로드된 애셋의 캐싱을 담당하는 클래스입니다. 

<br/>

> LyraAssetManager.h
```cpp
#pragma once

#include "Engine/AssetManager.h"

// 리플렉션과 관련된 코드입니다. 리플렉션은 런타임에 클래스에 대한 정보를 알아낼 수 있는 기능을 의미합니다. 
// C++는 언어차원에서 리플렉션을 지원하지 않기때문에 언리얼 엔진에서 자체적으로 리플렉션을 구축했습니다. 
// 리플렉션 코드들을 생성해서 런타임에 리플렉션이 가능하도록 도와주는데, 이 기능을 UHT(UnrealHeaderTool)이 합니다. 
// UHT이 헤더파일에 있는 UCLASS와 같은 리플렉션과 관련된 코드들을 읽으면서 generated.h와 generated.cpp에 리플렉션을 도와주는 코드들을 만들어줍니다. 
// 따라서, UCLASS()나 UPROPERTY()와 같은 리플렉션과 관련된 코드들을 사용하고 있다면, generated.h파일을 반드시 마지막에 포함하여 리플렉션이 제대로 동작해도록 해줘야합니다.
#include "LyraAssetManager.generated.h"

UCLASS(Config=Game)
class ULyraAssetManager : public UAssetManager
{
	GENERATED_BODY()
	
public:
	ULyraAssetManager();
}
```

<br/>

> LyraAssetManager.cpp
```cpp
#include "LyraAssetManager.h"
#include "LyraClone/LyraLogChannels.h"

// UE_INLINE_GENERATED_CPP_BY_NAME(LyraAssetManager)은 "LyraAssetManager.gen.cpp"로 변환됩니다. 
// 리플렉션과 컴파일 타임 단축을 위해서 헤더파일에 넣어준 generated.h와 짝을 맞춰서 gen.cpp 파일을 포함시켜줍니다.
#include UE_INLINE_GENERATED_CPP_BY_NAME(LyraAssetManager)

ULyraAssetManager::ULyraAssetManager()
	: Super()
{
    
}
```
<br/>

> [!NOTE]
> 기본적으로 엔진에서 제공하는 AssetManager를 사용하도록 설정되어 있습니다. 언리얼 에디터를 실행하고, 프로젝트 세팅에서 Asset Manager 클래스를 LyraAssetManager로 변경해줘야합니다.

<br/>

> LyraAssetManager.h
```cpp
// 애셋이 로딩되는 시간을 로깅해야할지를 반환하는 함수입니다.
static bool ShouldLogAssetLoads();
```

<br/>

> LyraAssetManager.cpp
```cpp
bool ULyraAssetManager::ShouldLogAssetLoads()
{
	// FCommandLine::Get 함수를 통해서 프로그램을 실행시킬때 같이 넘겨준 인자 값들을 받아올 수 있습니다. 
	// 예를 들어, exe파일을 cmd에서 실행시킬때 ProgramName.exe -argument 처럼 뒤에 인자값들을 넣어주는 경우가 있습니다. 
	// 이런 경우에 뒤에 넣어준 인자값들을 FCommandLine::Get 함수를 통해서 받아 올 수 있습니다.
	const TCHAR* CommandLineContent = FCommandLine::Get();

	// 앞에서 넘어온 인자값들을 편하게 파싱할 수있도록 도와주는 헬퍼함수가 FParse::Param입니다. 
	// 여기서는 LogAssetLoads라는 인자값이 존재하는지를 확인해서 bool값으로 반환해주고 있습니다.
	static bool bLogAssetLoads = FParse::Param(CommandLineContent, TEXT("LogAssetLoads"));
	return bLogAssetLoads;
}
```

<br/>

> [!NOTE]
> 콘솔 C++ 솔루션을 생성했을 때 기본적으로 Debug 모드와 Release 모드가 존재합니다. 언리얼에서 Debug모드에 해당하는 것이 Debug Editor와 DebugGame Editor이며, Release에 해당하는 것이 Development Editor입니다. <br/>
일반적으로는 디버깅을 하기 위해서 Debug모드를 사용합니다. 하지만, 언리얼 엔진은 매우 거대하기 때문에 Debug 모드로 실행을 하게 되면 에디터가 뜨는데 오랜시간이 걸려서 작업 속도가 매우 느려집니다. <br/>
이에 대한 해결방안으로 일단 Release 모드에 해당하는 Development Editor로 빌드를 하고, 매크로를 사용하여 디버깅이 필요한 부분만 최적화를 해제하는 방법을 사용하면 디버깅을 하면서 빠른 실행도 가능해집니다. <br/><br/>
PRAGMA_DISABLE_OPTIMIZATION <br/>
// 최적화를 원하는 코드 <br/>
PRAGMA_ENABLE_OPTIMIZATION

<br/>

> LyraAssetManager.h
```cpp
static UObject* SynchronousLoadAsset(const FSoftObjectPath& AssetPath);
```

<br/>

> LyraAssetManager.cpp
```cpp
// 인자로 들어온 애셋 경로가 유효한 경로인지 확인하고, 만약 유효하지 않을 경우 바로 nullptr을 반환합니다.
if (AssetPath.IsValid())
	{
		TUniquePtr<FScopeLogTime> LogTimePtr;

		// 애셋을 로드할때 로깅을 해야한다면, 로딩하는데 걸린 시간을 초단위로 로그로 남깁니다.
		if (ShouldLogAssetLoads())
		{
			// 동기 로딩은 게임스레드에서 진행할 경우 게임이 끊기는 느낌을 줄 수 있습니다. 
			// 로딩 시간이 짧은 것들은 게임 중간에 동기 로딩을 해도 불편함을 별로 못느끼지만, 로딩 시간이 긴 것들은 게임 중간에 비동기 로딩을 사용하면 게임이 끊기는 느낌을 줄일 수 있습니다. 
			// 혹시 로딩 시간이 너무 긴 애셋을 동기 로딩으로 불러오고 있지 않은지 확인하기 위해 로그를 남기게 됩니다.
			LogTimePtr = MakeUnique<FScopeLogTime>(*FString::Printf(TEXT("Synchronously loaded asset [%s]"), *AssetPath.ToString()), nullptr, FScopeLogTime::ScopeLog_Seconds);
		}

		// AssetManager가 초기화되었으면, AssetManager의 StreamableManager를 이용해서 정적 로드을 진행하고 로딩된 애셋을 반환합니다.
		if (UAssetManager::IsInitialized())
		{
			return UAssetManager::GetStreamableManager().LoadSynchronous(AssetPath);			
		}

		// AssetManager가 없다면, FSoftObjectPath에서 내장하고 있는 TryLoad 함수를 이용해서 동기 로드를 진행하고 로딩된 애셋을 반환합니다.
		return AssetPath.TryLoad();
	}

	return nullptr;
```

<br/>

> [!NOTE]
> 이미 이전에 로딩된 적이 있고 아직 GC에 의해 언로드 되지 않았다면, 다시 로드하지 않고 이미 로드된 애셋을 찾아서 반환하게 됩니다.
따라서, 애셋을 로드할때 뿐만아니라 어떤 애셋을 찾고 싶을때도 SynchronousLoadAsset 함수를 사용할 수 있고, 이런 경우에는 끊김 현상 없이 애셋을 가져올 수 있게 됩니다.

<br/>

> LyraAssetManager.h
```cpp
void AddLoadedAsset(const UObject* Asset);

// UPPROPERTY 매크로를 통해 GC에 TSet을 등록합니다. 
// AssetManager는 Engine.h에 유일하게 존재하고 엔진이 종료 될때까지 남아 있으므로 LoadedAssets도 엔진이 종료될때까지 남아있게 됩니다.
UPROPERTY()
TSet<TObjectPtr<const UObject>> LoadedAssets;

FCriticalSection SyncObject;
```
<br/>

> LyraAssetManager.cpp
```cpp
void ULyraAssetManager::AddLoadedAsset(const UObject* Asset)
{
	if (ensureAlways(Asset))
	{
		// 여러 쓰레드가 로딩을 진행하고 동시에 TSet에 추가하려고 하면 문제가 되므로, Lock을 걸어서 한 쓰레드씩만 TSet에 로드한 애셋을 추가할 수 있도록 만듭니다.
		FScopeLock Lock(&SyncObject);
		LoadedAssets.Add(Asset);
	}
}
```
<br/>

> LyraAssetManager.h
```cpp
template <typename AssetType>
static AssetType* GetAsset(const TSoftObjectPtr<AssetType>& AssetPointer, bool bKeepInMemory = true);

template <typename AssetType>
AssetType* ULyraAssetManager::GetAsset(const TSoftObjectPtr<AssetType>& AssetPointer, bool bKeepInMemory)
{
	AssetType* LoadedAsset = nullptr;

	// 이전에 구현한 동기 로딩 함수가 FSoftObjectPath를 받도록 되어 있기 때문에 먼저 FTSoftObjectPtr을 FSoftObjectPath로 변환하고 Path가 유효한지 확인합니다.
	const FSoftObjectPath& AssetPath = AssetPointer.ToSoftObjectPath();
	if (AssetPath.IsValid())
	{
		// 이미 해당 애셋이 로드된적이 있어서 캐싱되어 있다면 가져옵니다.
		LoadedAsset = AssetPointer.Get();
		if (LoadedAsset == nullptr)
		{
			// 만약 캐싱이 안되어 있음이 확인되면, 이전에 구현한 동기 로딩 함수을 통해서 애셋을 불러옵니다.
			LoadedAsset = Cast<AssetType>(SynchronousLoadAsset(AssetPath));
			ensureAlwaysMsgf(LoadedAsset, TEXT("Failed to load asset [%s]"), *AssetPointer.ToString());
		}

		// bKeepInMemory 인자값을 통해서 로드된 애셋을 무조건 계속 메모리에 유지할지, 어느 누구도 참조하지 않을때 메모리에서 날려버릴지 결정하게 됩니다.
		if (LoadedAsset && bKeepInMemory)
		{
			// 정상적으로 애셋이 로드되었고, 만약 계속 유지해야 한다면 
			// AddLoadedAsset 함수를 통해서 TSet에 로드된 추가시켜 애셋이 날아가지 않고 유지되도록 만듭니다.
			Get().AddLoadedAsset(Cast<UObject>(LoadedAsset));
		}
	}

	return LoadedAsset;
}

template <typename AssetType>
TSubclassOf<AssetType> ULyraAssetManager::GetSubclass(const TSoftClassPtr<AssetType>& AssetPointer, bool bKeepInMemory)
{
	TSubclassOf<AssetType> LoadedSubclass;
	const FSoftObjectPath& AssetPath = AssetPointer.ToSoftObjectPath();
	if (AssetPath.IsValid())
	{
		LoadedSubclass = AssetPointer.Get();
		if (LoadedSubclass == nullptr)
		{
			LoadedSubclass = Cast<UClass>(SynchronousLoadAsset(AssetPath));
			ensureAlwaysMsgf(LoadedSubclass, TEXT("Failed to load asset class [%s]"), *AssetPointer.ToString());
		}

		if (LoadedSubclass && bKeepInMemory)
		{
			Get().AddLoadedAsset(Cast<UObject>(LoadedSubclass));
		}
	}
	
	return LoadedSubclass;
}
```

<br/>

> [!NOTE]
> 액터를 스폰할때 SpawnActor 함수를 사용하게 되는데, 인자값으로 어떤 클래스를 이용해서 스폰할지를 받습니다. 인자로 넣어준 클래스를 탬플릿 삼아서 월드에 액터를 스폰하게 됩니다. <br/>
블루프린트의 경우에는 클래스 또한 하나의 애셋이기 때문에 해당 애셋을 로딩하기 위한 GetSubclass라는 함수를 따로 마련해놓습니다. <br/>
따라서 애셋을 크게 2가지 종류로 나누어보면, 스테틱메시나 텍스처와 같은 애셋과 블루프린트 클래스와 같은 애셋으로 구분할 수 있습니다. 여기에 대응하는 로딩함수가 각각 GetAsset과 GetSubclass 함수가 됩니다.

<br/>

> LyraAssetManager.h
```cpp
static UHakAssetManager& Get();
```
<br/>

> LyraAssetManager.cpp
```cpp
#include "LyraClone/LyraLogChannels.h"

ULyraAssetManager& ULyraAssetManager::Get()
{
	check(GEngine);

	// GEngine에 존재하는 AssetManager 객체를 가져와서 UHakAssetManager로 캐스팅해서 맞다면 해당 객체를 반환합니다.
	if (ULyraAssetManager* Singleton = Cast<ULyraAssetManager>(GEngine->AssetManager))
	{
		return *Singleton;
	}

	// 만약 AssetManager가 없다면 로그를 띄우는데 로그 수준이 Fatal이기 때문에 크래시를 냅니다. 
	// 따라서 아래의 반환 코드는 필요없는 코드이지만, return 문이 없으면 컴파일 에러가 나므로 단지 컴파일 에러를 통과하기 위해서 더미 AssetManager를 만든 다음 반환해줍니다.
	UE_LOG(LogLyra, Fatal, TEXT("Invalid AssetManagerClassName in DefaultEngine.ini(project settings); it must be LyraAssetManager"));
	return *NewObject<ULyraAssetManager>();
}
```
<br/>

> LyraAssetManager.h
```cpp
// AssetManager를 애셋의 로드와 캐싱의 용도뿐만 아니라 Lyra에서 사용하는 다양한 기능들의 초기화를 여기서 진행합니다.
// StartInitialLoading은 엔진의 시작 초기에 호출되기 때문에 프로젝트에서 사용하는 다양한 기능들을 초기화하는데 적절한 시점이 됩니다.
virtual void StartInitialLoading() override;
```
<br/>

> LyraAssetManager.cpp
```cpp
void ULyraAssetManager::StartInitialLoading()
{
	Super::StartInitialLoading();

	// StartInitialLoading 함수의 내용은 이후에 채울 것이기때문에 현재는 간단하게 로그만 남기도록 하겠습니다.
	UE_LOG(LogLyra, Display, TEXT("ULyraAssetManager::StartInitialLoading"));
}
```

<br/>

## Experience

게임에는 섬멸전, 깃발 뺏기 등 다양한 게임 모드들이 있을 수 있습니다. 언리얼 엔진에서 이것들을 다루기 위해 게임 모드라는 것을 만들었지만, 게임 모드가 너무 무거워지면서 경량화된 게임 모드를 만들 필요성이 생겼습니다. 
그래서 게임 모드와 비슷하지만 경량화된 Experience라는 개념이 만들어졌습니다. Experience에는 어떤 Pawn과 GameFeature를 사용할지와 같은 게임 모드에 관한 정보들이 들어가게 됩니다.

Lyra 게임을 켜보면 여러 텔레포트들 중 하나를 선택할 수 있는 맵이 제일 처음 나오게 됩니다. 텔레포트마다 하나의 Experience와 연결되어 있고, 텔레포트를 타면 연결된 맵으로 이동하게 됩니다. 
하지만 Experience에는 맵에 대한 정보가 없습니다. 그래서 Experience에 추가적으로 맵의 정보까지 합친 구조가 필요한데 이것을 UserFacingExperience라고 합니다.

<br/>

> LyraExperienceDefinition.h
```cpp
#pragma once

#include "LyraExperienceDefinition.generated.h"

class ULyraPawnData;

UCLASS(BlueprintType)
class ULyraExperienceDefinition : public UPrimaryDataAsset
{
	GENERATED_BODY()
	
public:
	ULyraExperienceDefinition(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

public:
	// Experince에는 어떤 PawnData와 GameFeature를 사용할 것인지를 담고 있습니다.
	UPROPERTY(EditDefaultsOnly, Category=Gameplay)
	TObjectPtr<ULyraPawnData> DefaultPawnData;

	UPROPERTY(EditDefaultsOnly, Category=Gameplay)
	TArray<FString> GameFeaturesToEnable;
};
```
<br/>

> LyraPawnData.h
```cpp
#pragma once

#include "LyraPawnData.generated.h"

UCLASS()
class ULyraPawnData : public UPrimaryDataAsset
{
	GENERATED_BODY()
	
public:
	ULyraPawnData(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

public:
	// PawnData에는 어떤 폰 클래스를 사용할 것인지, 입력은 어떻게 할것인지 등의 정보를 담게 되는데 지금은 간단하게 클래스의 형태만 만들어 두겠습니다.
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category="Lyra|Pawn")
	TSubclassOf<APawn> PawnClass;
};
```
<br/>

> LyraUserFacingExperienceDefinition.h
```cpp
#pragma once

#include "LyraUserFacingExperienceDefinition.generated.h"

UCLASS(BlueprintType)
class ULyraUserFacingExperienceDefinition : public UPrimaryDataAsset
{
	GENERATED_BODY()
	
public:
	ULyraUserFacingExperienceDefinition(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

public:
	UPROPERTY(BlueprintReadWrite, EditAnywhere, Category=Experience, meta=(AllowedTypes="Map"))
	FPrimaryAssetId MapID;

	UPROPERTY(BlueprintReadWrite, EditAnywhere, Category=Experience, meta=(AllowedTypes="LyraExperienceDefinition"))
	FPrimaryAssetId ExperienceID;
};
```
<br/>

> [!NOTE]
> PrimaryDataAsset는 애셋들을 FPrimaryAssetId로 관리하기 때문에 UserFacingExperience는 맵에 대한 ID와 Experience에 대한 ID를 가지고 있습니다. ID라고 해서 단순히 정수 값이 아니라 FPrimaryAssetId 구조체에는 해당 애셋에 대한 타입과 이름 정보가 들어있습니다. <br/>
FParimaryAssetId에는 어떤 ParimaryAsset의 ID라도 저장할 수 있기 때문에 AllowedTypes를 통해서 특정한 타입의 애셋만 필터링하여 나중에 데이터 애셋에서 더 쉽고 정확하게 원하는 애셋을 선택할 수 있게 해줍니다.

<br/>

## 프로젝트 세팅
언리얼 엔진에서 애셋들을 찾을 수 있도록 스캔할 Primary Asset Type들을 프로젝트 세팅에서 설정해줍니다.<br/>
애셋을 등록할떄 경로를 이용하거나 특정 애셋을 지정할 수 있습니다. 블루프린트의 경우에는 Has Blueprint Classes를 체크해야합니다.
<br/>

![image](https://github.com/LeapRealm/Study/assets/43628076/853533b3-1522-4f79-973e-5f626a20e5d0)
![image](https://github.com/LeapRealm/Study/assets/43628076/773c2d69-8fdf-4cba-87f8-be36c9e6e572)

<br/>

> [!NOTE]
> LyraUserFacingExperienceDefiniton.h에서 AllowedTypes 값과 프로젝트 세팅에서 Primary Asset Type에 정의된 값이 동일해야 정상적으로 찾을 수 있습니다.

<br/>

## 텔레포트 액터 생성 및 세팅

맵에 배치할 포탈을 만들기 위해, Actor를 상속받는 블루프린트 클래스를 만들고, 이름을 B_TeleportToUserFacingExperience로 지정합니다. <br/><br/>
![image](https://github.com/LeapRealm/Study/assets/43628076/fad565a1-bf04-4070-a52e-11d610a0d22f)

<br/>

B_TeleportToUserFacingExperience를 열어서 Capsule과 StaticMesh 컴포넌트를 추가해줍니다. <br/><br/>
![image](https://github.com/LeapRealm/Study/assets/43628076/b3af0dc8-e6c3-46b1-9935-02cddb091fec)

<br/>

Capsule 컴포넌트의 위치는 (0, 0, 90), Half Height와 Radius는 80으로 설정해줍니다. <br/><br/>
![image](https://github.com/LeapRealm/Study/assets/43628076/b6a548ef-906a-4675-8f26-c7bfb3aa5caa)

<br/>

StaticMesh 컴포넌트의 StaticMesh에는 Lyra 프로젝트에서 SM_launchpad_Round 스태틱메시를 이주해서 가져온 다음 넣어줍니다. <br/><br/>
![image](https://github.com/LeapRealm/Study/assets/43628076/d665aec4-e4eb-46a2-a67a-6bd3c39717ee)

<br/>

변수를 하나 추가해주고, 이름은 UserFacingExperienceToLoad, 타입은 LyraUserFacingExperienceDefiniton으로 지정합니다.<br/><br/>
![image](https://github.com/LeapRealm/Study/assets/43628076/d956ac98-3d15-41eb-b008-4c7d77d9b2f0)

<br/>

UserFacingExperienceToLoad 변수를 텔레포트 액터를 스폰할때 세팅해줄 수 있도록, Instance Editable과 Expose on Spawn을 체크해줍니다.<br/><br/>
![image](https://github.com/LeapRealm/Study/assets/43628076/5c6f6490-a1c3-4b04-ad84-f7bdcadae9dd)

<br/>

## 데이터 애셋 생성 및 세팅

LyraExperienceDefinition을 상속받는 블루프린트 클래스를 만들고, 이름을 B_LyraDefaultExperience로 지정합니다. <br/><br/>
![image](https://github.com/LeapRealm/Study/assets/43628076/ad3958d4-443f-4eb7-84c5-fe3b8672697e)

<br/>

LyraUserFacingExperienceDefinition을 상속받는 데이터 애셋을 만들고, 이름을 DA_ExamplePlaylist로 지정합니다. <br/><br/>
![image](https://github.com/LeapRealm/Study/assets/43628076/37b8ebf4-09d1-4dc6-bcfb-c6727def9b76)

<br/>

DA_ExamplePlaylist를 열고, Map ID에 L_DefaultEditorOverview 맵을 채워넣고, Experience ID에 B_LyraDefaultExperience를 채워넣어줍니다. <br/><br/>
![image](https://github.com/LeapRealm/Study/assets/43628076/6b00cd74-0386-4694-98bf-47b3a5dbd77c)

<br/>

## 텔레포트의 스폰을 담당하는 액터 생성 및 세팅

Actor를 상속받는 블루프린트 클래스를 만들고 이름을 B_ExperienceList3D로 지정합니다. <br/><br/>
![image](https://github.com/LeapRealm/Study/assets/43628076/98501276-8c30-4e5f-8baf-106cdf7b6c98)

<br/>

레벨에 배치하고 위치는 (0, 0, 50)으로 지정합니다. <br/><br/>
![image](https://github.com/LeapRealm/Study/assets/43628076/e70d6c38-d635-4262-b75f-f371d7dac1df)

<br/>

B_ExperienceList3D를 열어서 3개의 변수를 추가합니다. 
|변수 이름|변수 타입|컨테이너 타입|기본 값|
|------|---|---|---|
|UserFacingExperienceList|LyraUserFacingExperienceDefinition|배열|
|ExperiencePortals|B_TeleportToUserFacingExperience|배열|
|PortalSpacing|Float|단일|300|

![image](https://github.com/LeapRealm/Study/assets/43628076/cfe5ac49-5cdc-4d09-9159-4086c9960a76)

<br/>

UserFacingExperienceList 배열을 비우고, LyraUserFacingExperienceDefinition 타입의 애셋들의 PrimaryDataAssetID 목록을 가져옵니다. </br>
PrimaryDataAssetID 목록을 이용해서 애셋들의 비동기 로드를 진행합니다. <br/><br/>
![image](https://github.com/LeapRealm/Study/assets/43628076/180e4d17-fbac-4b4f-a8a7-b54e1dbd7810)

<br/>

비동기 로드가 끝나면, 비동기 로드된 애셋들을 하나씩 순회하면서 UserFacingExperienceList에 추가합니다. <br/><br/>
![image](https://github.com/LeapRealm/Study/assets/43628076/4d995148-86a2-41b3-9926-760eddeb0cdd)

<br/>

UserFacingExperienceList를 순회하면서 텔레포트를 하나씩 스폰합니다. <br/>
B_ExperienceList3D의 로컬 좌표계 기준으로 스폰 위치를 계산하고, Transform Location 노드를 통해 월드 좌표계로 변환하여 최종적인 스폰 위치를 정합니다. 
텔레포트를 스폰할때 UserFacingExperience를 세팅해줍니다. 스폰된 텔레포트는 나중에 사용하기 위해 Experience Portals 배열에 추가시켜줍니다.<br/><br/>
![19](https://github.com/LeapRealm/Study/assets/43628076/6d60249e-8e9a-47ba-adc7-5de4c90f4be5)

<br/>

게임을 실행시켜보면 텔레포트 액터가 스폰된 것을 확인할 수 있습니다. <br/><br/>
![image](https://github.com/LeapRealm/Study/assets/43628076/bc1044b6-a89c-4fef-8a21-f4f021b6eaed)

<br/>
