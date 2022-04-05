---
title: Unreal Engine — C++ Architecture
date: 2022-03-31 11:58:47 +07:00
---

# Assertions

Reporting errors and don't halt the execution. Works in all builds. Useful when you want runtime code verification but you’re handling the error case anyway (for example: handle errors through debugger or logs). In shipping work with: bUseLoggingInShipping = true;
___
- **ensure(false);** - like check(), but the program execution can be continued after the break. Notifies the crash reporter on the first time Expression is false
- **ensureMsgf(Expression, FormattedText, ...);** - Notifies the crash reporter and outputs FormattedText to the log on the first time Expression is false
- **ensureAlways(false);** - Notifies the crash reporter if Expression is false

Reporting errors and halting execution. Work only in debug, development, test builds:
___
- **check(false);** - like assert() in C, can be disabled by DO_CHECK 0.
- **verify(false);** - like check(), but the expression is still executed on DO_CHECK 0 (e.g. for variable assignments).
- **checkCode(false);** - macros which execute some code (in while do) in debug, test and development builds. it's not terminate the program.
- **checkNoRecursion();** - Halts execution if the line is hit more than once without leaving scope
- **unimplemented();** - Halts execution if the line is ever hit, similar to check(false), but intended for virtual functions that should be overridden and not called
- **checkNoReentry();** - Halts execution if the line is hit more than once
- **checkNoEntry();** - Halts execution if the line is ever hit, similar to check(false), but intended for code paths that should be unreachable

Reporting errors and halting execution exclusively in debug builds. Difference with check(), verify(): they are only active in debug builds.
Macroses which used for slow check (for example iterate large list and find cycle element).
___
- **checkSlow(false);**
- **checkfSlow(false);** 
- **verifySlow(false);**

# Delegates
Mostly referred to as "FOn_..."

- **DECLARE_DELEGATE** - C++ only, standard delegate, only one function can bind to it.
- **DECLARE_EVENT** - C++ only. Just like a multicast delegate but only the owner of this event can call a Broadcast.
- **DECLARE_MULTICAST_DELEGATE** - C++ only just like standard delegate but multiple functions can bind to it an be called all at once using Broadcast function.
- **DECLARE_DYNAMIC_MULTICAST_DELEGATE** - C++/BP. It's like a multicast delegate, but it is serializable and can be bindable from blueprints when you use BlueprintAssignable keyword in UPROPERTY.

**Links: ** 
https://stackoverflow.com/questions/62165120/c-equivalent-blueprint-event-dispatcher-vs-blueprint-events
https://www.programmersought.com/article/8873853019/
https://docs.unrealengine.com/en-US/API/Runtime/Core/Misc/FCoreDelegates/index.html
https://ikrima.dev/ue4guide/gameplay-programming/master-engine-flow/core-eventsdelegates/

# World context object
If you want to use a static BP node anywhere in the game code, but your C++ function wants to modify the state of the game world by creating objects or actors, or performing an action on any instanced data within the game world, then you can pass along a World Context Object.

**From:** https://michaeljcole.github.io/wiki.unrealengine.com/Blueprints,_Creating_C++_Functions_as_new_Blueprint_Nodes/

# Class Default Object
CDO is Class Default Object, it's a master copy of object for specific class contained in reflection system which in this case it's contained in class representing the class which is UClass. CDO contains object defaults and make them accessible very easily since C++ don't have such a feature. It is also used to hold default values of variables in virtual classes created from blueprints for example.

CDO is created when engine is initialized when it generates UClass objects for each class, then it naturally executes constructor setting default variables. Thats also reason why you should not use objects or gameplay code in constructor, because constructor is executed early in engine initiation stage where most objects don't exist, not to mention level is most likely won't be even loaded at that point. You should use different start up events which both UObject and AActor contains for many stages of it's creation process, you can read about those here:

https://docs.unrealengine.com/en-us/Programming/UnrealArchitecture/Actors/ActorLifecycle

You can access CDO of each class via this function in UClass of specific class, starting 4.9 you will also able to access defaults from CDO in blueprint too (in more protected way of corse):

https://docs.unrealengine.com/latest/INT/API/Runtime/CoreUObject/UObject/UClass/GetDefaultObject/1/index.html

So in short, because of this setup, constructor can be only used for defaults that will be contained in CDO.

**From:** https://answers.unrealengine.com/questions/191353/view.html#:~:text=CDO%20is%20Class%20Default%20Object,t%20have%20such%20a%20feature.


# Touch events
By Default, two joysticks are provided. You can override that to use none or only one in the Engine input settings

**Add touch handlers:**
- **void AMyCharacter::BeginTouch(const ETouchIndex::Type FingerIndex, const FVector Location);**
- **void AMyCharacter::EndTouch(const ETouchIndex::Type FingerIndex, const FVector Location);**
- **void AMyCharacter::TouchUpdate(const ETouchIndex::Type FingerIndex, const FVector Location);**

**Bind to touch events in SetupPlayerInputComponent:**
```c++
bool AMyCharacter::EnableTouchscreenMovement(class UInputComponent* InputComponent)
{
	bool bResult = false;
	if(FPlatformMisc::GetUseVirtualJoysticks() || GetDefault<UInputSettings>()->bUseMouseForTouch)
	{
		bResult = true;
		InputComponent->BindTouch(EInputEvent::IE_Pressed, this, &AMyCharacter::BeginTouch);
		InputComponent->BindTouch(EInputEvent::IE_Released, this, &AMyCharacter::EndTouch);
		InputComponent->BindTouch(EInputEvent::IE_Repeat, this, &AMyCharacter::TouchUpdate);
	}
	return bResult;
}
```
# FObjectInitializer

They are both valid unreal engine class constructor, FObjectInitializer has some very useful function to initialize the object properties, but some simple classes don't really need that functionalities, and that is why, I think, Epic made an alternative constructor that has no parameter, just to make things simple and tidy.

https://docs.unrealengine.com/latest/INT/Programming/UnrealArchitecture/Reference/Classes/#classconstructor

# BeginDeferredActorSpawnFromClass and FinishSpawningActor
'BeginDeferredActorSpawnFromClass' allows you initialize object before begin play call.

```c++
FTransform SpawnTransform(Rotation, Origin);
auto MyDeferredActor = Cast<ADeferredActor>(UGameplayStatics::BeginDeferredActorSpawnFromClass(this, DeferredActorClass, SpawnTransform));
if (MyDeferredActor != nullptr)
{
	MyDeferredActor->Init(ShootDir);
	UGameplayStatics::FinishSpawningActor(MyDeferredActor, SpawnTransform);
}
```

**From:** https://answers.unrealengine.com/questions/284744/spawning-an-actor-with-parameters.html

# AActor VS UObject
AActor: 
- [x] Support replication 
- [x] Can be placed in a level
- Size: about 1016 bytes

UObject: 
- [ ] Support replication 
- [ ] Can be placed in a level
- Size: about 56 bytes

Beside this Actor and Object are garbage collected and support blueprint. Objects are smaller than actor in memory, even if you should be concerned since textures are much bigger than those.

From: https://answers.unrealengine.com/questions/296674/noob-question-difference-between-object-actor.html

An actor currently takes up about 1016 bytes while an object takes up about 56 bytes. At least on this version with this compiler/os but that should give you an indication of the intended uses.

From: https://answers.unrealengine.com/questions/296674/noob-question-difference-between-object-actor.html

# Strings
## FString
- FString is 16 bytes ~ (2x size of FName, 4x the size of a float/int32)

FString is the easiest type to manipulate, modify, search and replace, concatenate, etc etc.
FString is a mutable string, analogous to std::string. FString has a large suite of methods for making it easy to work with strings. To create a new FString, use the TEXT() macro:
```c++
FString MyStr = TEXT("Hello, Unreal 4!").
```

### Printf
The FString function, Printf, can create FString objects the format argument specifiers as the C++ printf function. Similarly, the UE_LOG macro prints a printf formatted string to the screen, log output, and log files, depending on what type of UE4 build is running.

## FText
- FText is 40 bytes ~ (more than 2x size of FString, more than 4x the size of an FName)
FText is similar to FString, but it is meant for localized text. To create a new FText, use the NSLOCTEXT macro. This macro takes a namespace, key, and a value for the default language:

```c++
FText MyText = NSLOCTEXT("Game UI", “Health Warning Message”, “Low Health!”)

You could also use the LOCTEXT macro, so you only have to define a namespace once per file. Make sure to undefine it at the bottom of your file.
// In GameUI.cpp

define LOCTEXT_NAMESPACE "Game UI"
//…
FText MyText = LOCTEXT("Health Warning Message", “Low Health!”)
//…

undef LOCTEXT_NAMESPACE
// End of file
```

https://docs.unrealengine.com/latest/INT/API/Runtime/Core/Internationalization/FText/index.html

## FName
- FName is 8 bytes ~ (smallest! Twice the size of a float/int32)
So whenever you dont need the extra bytes you can use FName, and you should only use FText for visible displayed text that actually matters to be different, region to region, thus requiring Localization.

Internally FName uses an index into a string table. Two FNames made with the same string (ignoring case-variations as FName is case-insensitive) will generate the same string table index, which is what's used by the FName == operator to compare them.

Note that you should not store FNames on disk or serialize them over the network. They are guaranteed to be unique only within the local process (not even on the same machine). When persisting or serializing FName fields, convert them from and to FString instead.

FNames are usually going to be an ID for something. They're case-insensitive, which is probably the most important thing to remember about them.
- Asset names
- Material Parameters
- Tagging Actors
- Skeleton bone names

## FCString
Set of basic string utility functions operating on plain C strings. In addition to the wrapped C string API,this struct also contains a set of widely used utility functions that operate on c strings. There is a specialized implementation for ANSICHAR and WIDECHAR strings provided. To access these functionality, the convenience typedefs FCString and FCStringAnsi are provided.

## Conversion Routines
We have a number of macros to convert strings to and from various encodings. These macros use a class instance declared in local scope and allocate space on the stack, so it is very important you do not retain pointers to them! They are intended only for passing strings to function calls.
- TCHAR_TO_ANSI(str)
- TCHAR_TO_OEM(str)
- ANSI_TO_TCHAR(str)
- TCHAR_TO_UTF8(str)
- UTF8_TO_TCHAR(str)

These use the following helper classes from UnStringConv.h:
- typedef TStringConversion<TCHAR,ANSICHAR,FANSIToTCHAR_Convert> FANSIToTCHAR;
- typedef TStringConversion<ANSICHAR,TCHAR,FTCHARToANSI_Convert> FTCHARToANSI;
- typedef TStringConversion<ANSICHAR,TCHAR,FTCHARToOEM_Convert> FTCHARToOEM;
- typedef TStringConversion<ANSICHAR,TCHAR,FTCHARToUTF8_Convert> FTCHARToUTF8;
- typedef TStringConversion<TCHAR,ANSICHAR,FUTF8ToTCHAR_Convert> FUTF8ToTCHAR;

It is also critical that when using TCHAR_TO_ANSI you do not assume the number of bytes will be the same as the TCHAR string length. Multiple byte character sets could require multiple bytes per TCHAR character. If you need to know the length of the resulting string in bytes, you can use the helper class instead of the macros. For example:
```c++
FString String;
...
FTCHARToANSI Convert(*String);
Ar->Serialize((ANSICHAR*)Convert, Convert.Length()); // FTCHARToANSI::Length() returns the number of bytes for the encoded string, excluding the null terminator.
```
The objects that these macros declare have very short lifetimes. The intended use case for them is as function parameters, and they are suited to this situation. Do not assign a variable to the contents of the converted string, as the object will go out of scope and the string will be released. This can cause crashes if your code continues to access the pointer to released memory.
