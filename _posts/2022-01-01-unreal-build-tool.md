---
title: Unreal Engine — Unreal Build Tool
date: 2022-03-31 11:58:47 +07:00
---

# UINTERFACE

# Defined UBT macroses
- https://imzlp.me/posts/5214/

# UMETA
- https://docs.unrealengine.com/en-US/Programming/UnrealArchitecture/Reference/Metadata/index.html
- https://docs.unrealengine.com/en-US/API/Runtime/CoreUObject/UObject/UM_3/index.html

## Create interface class
Links: https://docs.unrealengine.com/en-US/ProgrammingAndScripting/GameplayArchitecture/Interfaces/index.html

The "U-prefixed" class needs no constructor or any other functions, while the "I-prefixed" class will contain all interface functions and is the one that will actually be inherited by your other classes.

The Blueprintable specifier is required if you want to allow Blueprints to implement this interface.

```c++
#pragma once
#include "ReactToTriggerInterface.generated.h"

UINTERFACE(MinimalAPI, Blueprintable)
class UReactToTriggerInterface : public UInterface
{
    GENERATED_BODY()
};

class IReactToTriggerInterface
{    
    GENERATED_BODY()

public:
    /** Add interface function declarations here */
};
```

## Interface Specifiers:
- **BlueprintType** - Exposes this class as a type that can be used for variables in Blueprints.
- **DependsOn=(ClassName1, ClassName2, ...)** - 	
All classes listed will be compiled before this class. ClassName must specify a class in the same (or a previous) package. Multiple dependency classes can be specified using a single DependsOn line delimited by commas, or can be specified using a separate DependsOn line for each class. This is important when a class uses a struct or enum declared in another class as the compiler only knows what is in the classes it has already compiled.
- **MinimalAPI** - 	
Causes only the class's type information to be exported for use by other modules. The class can be cast to, but the functions of the class cannot be called (with the exception of inline methods). This improves compile times by not exporting everything for classes that do not need all of their functions accessible in other modules.

## Determining If a Class Implements Your Interface:
```c++
bool bIsImplemented = OriginalObject->GetClass()->ImplementsInterface(UReactToTriggerInterface::StaticClass()); // bIsImplemented will be true if OriginalObject implements UReactToTriggerInterface.

bIsImplemented = OriginalObject->Implements<UReactToTriggerInterface>(); // bIsImplemented will be true if OriginalObject implements UReactToTriggerInterfacce.

IReactToTriggerInterface* ReactingObjectA = Cast<IReactToTriggerInterface>(OriginalObject); // ReactingObject will be non-null if OriginalObject implements UReactToTriggerInterface.
```

## Casting To Other Unreal Types:
```c++
IReactToTriggerInterface* ReactingObject = Cast<IReactToTriggerInterface>(OriginalObject); // ReactingObject will be non-null if the interface is implemented.

ISomeOtherInterface* DifferentInterface = Cast<ISomeOtherInterface>(ReactingObject); // DifferentInterface will be non-null if ReactingObject is non-null and also implements ISomeOtherInterface.

AActor* Actor = Cast<AActor>(ReactingObject); // Actor will be non-null if ReactingObject is non-null and OriginalObject is an AActor or AActor-derived class.
```

# USTRUCT
Here are some helpful hints and things to remember when using Structs:

- UStructs can use UE's smart pointer and garbage collection systems to prevent UObjects from being removed by garbage collection.
 
- Structs are different from UObjects, and because of this Structs are best used for simple data types.
     - For more complicated interactions within your project, you may want to make a UObject or AActor subclass instead.

- UStructs ARE NOT considered for replication.
     - UProperty variables ARE considered for replication.

From: https://docs.unrealengine.com/en-US/ProgrammingAndScripting/GameplayArchitecture/Structs/UsingStructs/index.html

# UFUNCTION
## Meta specifiers can be defined as:
```c++
UFUNCTION(..., Meta = (MetaSpecifier = “Setting”, Meta2 = “Other”))
```

## Meta specifiers:
- **HidePin** - Takes parameter name(s) to Hide.
- **DefaultToSelf** - Takes parameter name(s) to default to self.
- **ExpandEnumAsExecs** - Takes parameter name of an enum ref parameter to give your node multiple execution Pins. Function must not be pure. May have to set BlueprintPure=false
- **WorldContext** - Takes parameter name to automatically pass a world content object to. Is Hidden automatically.
- **AutoCreateRefTerm** - If no argument is given in BP Graph, automatically create a default reference for the type. (AutoCreateRefTerm = "Rotation"; AutoCreateRefTerm = "AdditionalRequiredResources, AdditionalClaimedResources")

## Other specifiers:
- **Exec** - The function can be executed from the in-game console. Exec commands only function when declared within certain class. The UnrealHeaderTool code generator will not produce a execFoo thunk for this function; it is up to the user to provide one.

## Network specifiers:
- **Client** - The function is only executed on the client that owns the Object the function belongs to. Provide a body named [FunctionName]_Implementation instead of [FunctionName];
- **Server** - The function is only executed on the server. Provide a body named [FunctionName]_Implementation instead of [FunctionName];
- **NetMulticast** - This function is both executed locally on the server and replicated to all clients, regardless of the Actor's NetOwner.
- **Reliable** - The function is replicated over the network, and is guaranteed to arrive regardless of bandwidth or network errors. Only valid when used in conjunction with Client or Server.
- **Unreliable** - The function is replicated over the network but can fail due to bandwidth limitations or network errors. Only valid when used in conjunction with Client or Server.
- **BlueprintCosmetic** - Function is cosmetic and will not run on dedicated servers.
- **BlueprintAuthorityOnly** - This function will only execute with network authority.

## Useful blueprint specifiers: 
- **BlueprintSpawnableComponent** - Component Class can be spawned by a Blueprint.
- **BlueprintImplementableEvent** - Function can be declared in C++ (cannot be defined, blueprint overload), call this function in C++, blueprint overload realizes this function.
- **BlueprintNativeEvent** - You can declare and define functions in C++, call the function in C++, and implement the function by overloading the blueprint (the blueprint can overload or not overload the C++ parent class function).

From: https://programmersought.com/article/15766221673/

# Private vs Public folders
There is a reason to use public and private folders. Using public exposes it. Using private hides it. There should be no reason code files that are not used in a public manner to be in the public folder. Just like you don't make things public you don't want exposed in c++ itself.

The public/private structure is important to learn and know. Granted it doesn't change much for a project implementation. but for plugin modules, this matters immensely. In fact, allot of code people put in their game prj, should be made into a plugin module with logical private/public top level folders.

Since UE4 is an API it makes sense to make and use plugins and let your game use those plugins. It also makes sense to dictate what files go in what folder because not only does it simplify your include statements significantly, but it also is much much cleaner directory structure and helps you keep a mental model of large amounts of code. The most important part is that it will help indicate what files are interface vs implementation.

This is a very popular type of folder structure. Every place I've consulted and worked with uses this structure.

Edit i should say that this private folder are not special they are just folder name, you can use any name you want... Though, from what i learned public is a special folder name.

From: https://forums.unrealengine.com/development-discussion/c-gameplay-programming/66417-what-public-and-private-mean-on-c-wizard

# Reflection system
## To iterate over all members of a UStruct, use a TFieldIterator:

This may fetch properties from inheritance class (even from Blueprint class).
```c++
for (TFieldIterator<UProperty> PropIt(GetClass()); PropIt; ++PropIt)
{
    UProperty* Property = *PropIt;
    // Do something with the property
}
```

## Using reflection data
Most game code can ignore the property system at runtime, enjoying the benefits of the systems that it powers, but you might find it useful when writing tool code or building gameplay systems.

The type hierarchy for the property system looks like this:

- UField
	- UStruct
		- UClass (C++ class)
		- UScriptStruct (C++ struct)
		- UFunction (C++ function)

	- UEnum (C++ enumeration)
	- UProperty (C++ member variable or function parameter)
		- (Many subclasses for different types)

UStruct is the basic type of aggregate structures (anything that contains other members, such as a C++ class, struct, or function), and shouldn’t be confused with a C++ struct (that's UScriptStruct). UClass can contain functions or properties as their children, while UFunction and UScriptStruct are limited to just properties.

You can get the UClass or UScriptStruct for a reflected C++ type by writing UTypeName::StaticClass() or FTypeName::StaticStruct(), and you can get the type for a UObject instance using Instance->GetClass() (it's not possible to get the type of a struct instance since there is no common base class or required storage for structs).

From: https://www.unrealengine.com/en-US/blog/unreal-property-system-reflection?sessionInvalidated=true

## Markup
To mark a header as containing reflected types, add a special include at the top of the file. This lets UHT know that they should consider this file, and it’s also required for the implementation of the system (see the ‘A peek behind the curtain’ section for more information).

```
#include "FileName.generated.h"
```

You can now use UENUM(), UCLASS(), USTRUCT(), UFUNCTION(), and UPROPERTY() to annotate different types and member variables in the header. Each of these macros goes before the type or member declaration, and can contain additional specifier keywords.

From: https://www.unrealengine.com/en-US/blog/unreal-property-system-reflection?sessionInvalidated=true

## DECLARE_CLASS
DECLARE_CLASS, which declare 'Super' class, 'StaticClass', etc... Each class, which inheritance from UClass contains StaticClass method.
Useful: https://github.com/EpicGames/UnrealEngine/blob/2bf1a5b83a7076a0fd275887b373f8ec9e99d431/Engine/Source/Runtime/CoreUObject/Public/UObject/ObjectMacros.h#L1549

# FORCEINLINE
It most likely macro that places __forceinline which is non-standard (not part of C++ specifications) keyword exclusive to VS compiler, that forces compiler to do inline unconditionally, because with standard inline compiler might not decide to do so, for it's own optimization reasons (which might not understand what you trying to achieve)

Inline means that on call of that function compiler gonna paste function code to place of the call instead of doing standard jump to function call somewhere else juggling register values on the way, in theory it speeds up execution of function call, but in exchange you create multiple copies of function on every time you call it elsewhere in your C++ code making your machine code size bigger and it takes more memory as result. Compiler have a privilege to decide whatever it optimal to do so and have final decision to do so, force inline forces to do always inline.

https://msdn.microsoft.com/library/355f120c-2847-4608-ac04-8dda18ffe10c

To keep compatibility with other compilers (like clang for Linux and Mac), UE4 needs to have macro that gonna place other-compiler equivalent or just fallback to standard inline.

From: https://answers.unrealengine.com/questions/852234/what-is-forceinline-macro.html


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