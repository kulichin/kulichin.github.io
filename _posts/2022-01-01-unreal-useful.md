---
title: Unreal Engine — Useful
date: 2022-03-31 11:58:47 +07:00
---

# Optimization
- [Performance guide](https://gpuopen.com/unreal-engine-performance-guide/)
- [Optimization Tips for Load Times and GC](https://ikrima.dev/ue4guide/performance-optimization/asset-size-loading/)
- [Improving Memory Usage and Load Times in UE4](https://developer.oculus.com/blog/developer-perspective-improving-memory-usage-and-load-times-in-ue4/?locale=fr_FR)
- [Merging Meshes in Unreal](https://www.manicmachinegames.com/blog/2019/8/5/merging-meshes-in-unreal-and-why-it-matters)
- [The Significance Manager](https://www.youtube.com/watch?v=u7K4qFTW608&ab_channel=MazyModz)

# Useful links:
- [Unreal Engine sizeof() Types](https://gist.github.com/rlabrecque/55b39767960194850ce543d5d5f16eab0)
- [Global Variables](https://dawnarc.com/2018/12/ue4global-variables/)
- [Core Templates](https://docs.unrealengine.com/en-US/API/Runtime/Core/Templates/index.html)
- [Core Delegates](https://docs.unrealengine.com/en-US/API/Runtime/Core/Misc/FCoreDelegates/index.html)
- [And another Core Delegates](https://ikrima.dev/ue4guide/gameplay-programming/master-engine-flow/core-eventsdelegates/)

# TInlineAllocator
In this example you avoid any dynamic allocation for the first 16 elements added to the array, because they fit in the area on the stack that is part of the TInlineAllocator. At 17 elements and beyond, all elements are moved to the secondary allocator (ie heap allocator) for storage.
```c++
TArray<Shape*> MyShapeArray;

// and easily change it to:
TArray<Shape*, TInlineAllocator<16>> MyShapeArray;
```
**Links**: 
- [TInlineAllocator](https://zhuanlan.zhihu.com/p/78351385)
- [Optimizing TArray Usage for Performance](https://www.unrealengine.com/en-US/blog/optimizing-tarray-usage-for-performance)

# Event on Close
I think the best way to do this. It is create custom game engine class and implement events you want.

You should add to `DefaultEngine.ini`:
```
[/Script/Engine.Engine]
GameEngine=/Script/QAGame.MyCustomGameEngine
```

**MyCustomGameEngine.h**:
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

**Links**:
- [Creating Latent Functions for Blueprints in C++](https://www.casualdistractiongames.com/post/2016/05/15/creating-latent-functions-for-blueprints-in-c)
- [Custom latent – delayed Blueprint function in C++](http://johnmcroberts.com/index.php/2017/11/30/custom-delayed-blueprint-function-in-c/)
