---
title: Unreal Engine - Useful
date: 2022-03-31 11:58:47 +07:00
comments: false
---

# Optimization
- [Performance guide](https://gpuopen.com/unreal-engine-performance-guide/)
- [Optimization Tips for Load Times and GC](https://ikrima.dev/ue4guide/performance-optimization/asset-size-loading/)
- [Improving Memory Usage and Load Times in UE4](https://developer.oculus.com/blog/developer-perspective-improving-memory-usage-and-load-times-in-ue4/?locale=fr_FR)
- [Merging Meshes in Unreal](https://www.manicmachinegames.com/blog/2019/8/5/merging-meshes-in-unreal-and-why-it-matters)
- [The Significance Manager](https://www.youtube.com/watch?v=u7K4qFTW608&ab_channel=MazyModz)

# sizeof types
```
https://gist.github.com/rlabrecque/55b39767960194850ce543d5d5f16eab
```

# Global Variables
```
https://dawnarc.com/2018/12/ue4global-variables/
```

# Core templates
```
https://docs.unrealengine.com/en-US/API/Runtime/Core/Templates/index.html
```

# Core delegates
```
https://docs.unrealengine.com/en-US/API/Runtime/Core/Misc/FCoreDelegates/index.html
https://ikrima.dev/ue4guide/gameplay-programming/master-engine-flow/core-eventsdelegates/
```

# UE_ARRAY_COUNT
```c++
const FVector PawnLocation = GetPawn()->GetActorLocation();
FRotator ViewDir = GetControlRotation();
ViewDir.Pitch = -45.0f;

const float YawOffsets[] = { 0.0f, -180.0f, 90.0f, -90.0f, 45.0f, -45.0f, 135.0f, -135.0f };
const float CameraOffset = 600.0f;
FCollisionQueryParams TraceParams(SCENE_QUERY_STAT(DeathCamera), true, GetPawn());

FHitResult HitResult;
for (int32 i = 0; i < UE_ARRAY_COUNT(YawOffsets); i++)
{
	FRotator CameraDir = ViewDir;
	CameraDir.Yaw += YawOffsets[i];
	CameraDir.Normalize();

	const FVector TestLocation = PawnLocation - CameraDir.Vector() * CameraOffset;
	
	const bool bBlocked = GetWorld()->LineTraceSingleByChannel(HitResult, PawnLocation, TestLocation, ECC_Camera, TraceParams);
	if (!bBlocked)
	{
		CameraLocation = TestLocation;
		CameraRotation = CameraDir;
		return true;
	}
}
```

# TInlineAllocator
In this example you avoid any dynamic allocation for the first 16 elements added to the array, because they fit in the area on the stack that is part of the TInlineAllocator. At 17 elements and beyond, all elements are moved to the secondary allocator (ie heap allocator) for storage.
```c++
TArray<Shape*> MyShapeArray;

// and easily change it to:
TArray<Shape*, TInlineAllocator<16>> MyShapeArray;
```
Links: 
- https://zhuanlan.zhihu.com/p/78351385
- https://www.unrealengine.com/en-US/blog/optimizing-tarray-usage-for-performance

# Event on Close
I think the best way to do this. It is create custom game engine class and implement events you want.

You should add to DefaultEngine.ini:
```
[/Script/Engine.Engine]
GameEngine=/Script/QAGame.MyCustomGameEngine
```

MyCustomGameEngine.h:
```c++
#pragma once

#include "Engine/GameEngine.h"
#include "MyCustomGameEngine.generated.h"

UCLASS()
class UMyCustomGameEngine : public UGameEngine
{
	GENERATED_UCLASS_BODY()

	// When engine starts
	void Init(class IEngineLoop* InEngineLoop);
	// Close engine
	void PreExit();
};
```

MyCustomGameEngine.cpp
```c++
#include "QAGame.h"
#include "MyCustomGameEngine.h"

UMyCustomGameEngine::UMyCustomGameEngine(const FPostConstructInitializeProperties& PCIP)
: Super(PCIP)
{
}

void UMyCustomGameEngine::Init(class IEngineLoop* InEngineLoop)
{
	Super::Init(InEngineLoop);
}

void UMyCustomGameEngine::PreExit()
{
	Super::PreExit();
}
```
# Property helpers

## FindArrayPropertyInObject
```c++
FORCEINLINE void FindArrayPropertyInObject(UObject* Object, FName FieldName)
{
	if (IsValid(Object))
	{
		/** TPropertyValueIterator
		* Class for iterating through all fields in UStruct.
		* Return field with value data
		*/
		for (TPropertyValueIterator<UProperty> It(Object->GetClass(), Object); It; ++It)
		{
			if (Cast<UArrayProperty>(It.Key()) && IsValid(It.Key()) && It.Key()->GetFName() == FieldName)
			{
				void* ArrayPtr = (void*)It.Value();
				TArray<UObject*>* Array = (TArray<UObject*>*)ArrayPtr;

				if (Array)
				{
					for (UObject* Element : *Array)
					{
						GEngine->AddOnScreenDebugMessage(-1, 10000.0f, FColor::Red, Element->GetFName().ToString());
					}
				}
			}
		}
	}
}
```

## IsBlueprintableClass
```c++
FORCEINLINE bool IsBlueprintableClass(UStruct* Struct)
{
	if (IsValid(Struct) && Cast<UBlueprintGeneratedClass>(Struct))
	{
		return true;
	}

	return false;
}
```

## PropertyValueIterator
```c++
FORCEINLINE void PropertyValueIterator(UObject* Object)
{
	if (IsValid(Object))
	{
		/** TPropertyValueIterator
		* Class for iterating through all fields in UStruct.
		* Return field with value data
		*/
		for (TPropertyValueIterator<UProperty> It(Object->GetClass(), Object); It; ++It)
		{
		    GEngine->AddOnScreenDebugMessage(-1, 10000.0f, FColor::Red, It.Key()->GetFName().ToString());
		}
	}
}
```

## ObjectIterator
```c++
template<typename T>
FORCEINLINE void ObjectIterator()
{
	/** TObjectIterator
	* Class for iterating through all objects created in world
	* which inherit from a specified base class.  
	* Does not include any class default objects.
	* Note that when Playing In Editor, this will find objects in the
	* editor as well as the PIE world, in an indeterminate order.
	*/
	for (TObjectIterator<AActor> It; It; ++It)
	{
        	GEngine->AddOnScreenDebugMessage(-1, 10000.0f, FColor::Red, It->GetFName().ToString());
	}
}
```

## FieldIterator
```c++
FORCEINLINE void FieldIterator(UStruct* Struct)
{
	if (IsValid(Struct))
	{
        	/** TFieldIterator
		* Class for iterating through all fields in UStruct.
		* Return field data
        	*/
    		for (TFieldIterator<UProperty> It(Struct); It; ++It)
    		{
    			GEngine->AddOnScreenDebugMessage(-1, 10000.0f, FColor::Red, It->GetFName().ToString());
		}
	}
}
```

# Latent action
A Latent Action is a node that can be called in a blueprint that will return at some point in the future. The best example is the Delay node.
```c++
class FDelayAction : public FPendingLatentAction
{
public:
	float TimeRemaining;
	FName ExecutionFunction;
	int32 OutputLink;
	FWeakObjectPtr CallbackTarget;

	FUpgradeDelayAction(float Duration, const FLatentActionInfo& LatentInfo)
		: TimeRemaining(Duration)
		, ExecutionFunction(LatentInfo.ExecutionFunction)
		, OutputLink(LatentInfo.Linkage)
		, CallbackTarget(LatentInfo.CallbackTarget)
	{
	}

	virtual void UpdateOperation(FLatentResponse& Response) override
	{
		TimeRemaining -= Response.ElapsedTime();
		Response.FinishAndTriggerIf(TimeRemaining <= 0.0f, ExecutionFunction, OutputLink, CallbackTarget);
	}
};
```

Some articles:
- http://unktomi.github.io/Latent-Actions-Cont/Cont.html
- https://www.casualdistractiongames.com/post/2016/05/15/creating-latent-functions-for-blueprints-in-c
- http://johnmcroberts.com/index.php/2017/11/30/custom-delayed-blueprint-function-in-c/
