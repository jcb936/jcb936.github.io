---
layout: post
title:  "Unreal Engine's Asset Handling Practices"
date:   2023-05-07
excerpt: "A fast overview of asset handling and common pitfalls in UE"
tag:
- blog
- c++
---

{% include toc.html %}

# Introduction

One very easy mistake during Unreal Engine’s game development is not managing asset references correctly from the get go, which can make the memory budget skyrocket very fast and make editor loading times long. Here we will investigate some common pitfalls of managing memory, some good practices for developers and a brief exploration of the different pointer types for managed objects in Unreal and when to use them. The article starts with the good practices and then go into what hard references are and how the different pointer types work.

---

# TL;DR: Good practices to avoid hard references

If you just want to have a non-exhaustive list of how to avoid hard references, you can read here.

## Use the right pointer types in code

Always use the most strict pointer type you can:

* If you don’t need ownership of an object, use `TWeakObjectPtr<T>`
* If you need ownership over objects that are not part of the object structure, prefer `TSoftObjectPtr<T>`
* For all the rest, like components and other objects that are integral part of the object structure and should be hard referenced at all time, use `TObjectPtr<T>` / raw pointers.

## Use the right reference types in blueprint

If you’re referencing object types in blueprint variables, be sure to use **Soft Object Reference** and **Soft Class Reference** when needed.

## Avoid spawning blueprint types from blueprints using the node

Avoid spawning blueprint types from blueprints setting the class directly from the node dropdown menu. It is perfectly fine to spawn C++ class types and async loaded Soft Class References.

## Avoid casting to blueprint types and improperly spawning objects in blueprints

Avoid casting objects to blueprint types. As with the above, it is perfectly fine to cast to C++ class types and to Soft Class References types. If you just need to do a boolean check on whether an actor is of a blueprint type you expect, a neat trick that **doesn’t create a hard reference** is shown below, comparing the class type of an actor to a soft class reference.

<figure>
	<img src="/assets/img/unreal_trick.png">
	<figcaption>Thank you Mark Craig from Lucid Games and his UnrealFest talk for this trick: <a href="https://www.youtube.com/watch?v=4-oRyDLfo7M" title="Asset Dependency Chains: The Hidden Danger">Asset Dependency Chains: The Hidden Danger | Unreal Fest 2022</a> on YouTube.</figcaption>
</figure>

If you need to execute some functionality, **Interfaces** are a perfect way to avoid hard references, **as long as the interface itself lacks a hard reference to another asset type as part of any function parameters or return values**. Casting and calling functions on a blueprint type implementing an interface **does not create any hard reference**.

More info on hard references on Blueprint: [Hard References & Reasons to Avoid Them ](https://raharuu.github.io//unreal/hard-references-reasons-avoid/)

---

# Hard references

Before we get into any of this, we need to understand what a **hard reference** is. Hard references refer to objects tightly coupled together,
meaning that when one is loaded, the other will be loaded as well. As an example:

{% highlight c++ %}
UCLASS()
class AMyCharacter : ACharacter
{
	[...]

	UPROPERTY()
	UStaticMesh* WeaponMesh{};
}
{% endhighlight %}

Here, `AMyCharacter` is **hard referencing** `WeaponMesh` . Which means that no matter whether the weapon mesh is actually in the scene or not, or whether the character is in view or not. When the character is loaded, the weapon mesh is loaded too. This is the same for actor references, audio assets and any other type of assets in the game. You can now see how this can go out of control relatively fast if we are not careful.

While the above is pretty straightforward, a less known type of hard referencing happens in blueprint. Say I have a BP_Test referencing a
UStaticMesh in its components.

<figure>
	<img src="/assets/img/bp_test.png">
</figure>

Now, if we try to Spawn an actor of type BP_Test from another blueprint, or we try to cast an object into a BP_Test, as in the following
image:

<figure>
	<img src="/assets/img/bp_test_spawn.png">
</figure>

We will **hard reference** BP_Test in that blueprint, meaning that BP_Test with its static mesh will need to be loaded whenever we load our blueprint. Say this blueprint is our character blueprint, it means that whenever we load our character blueprint we will also need to load a totally unnecessary static mesh. [Hard References & Reasons to Avoid Them](https://raharuu.github.io//unreal/hard-references-reasons-avoid/) offers a good explanation of hard references in blueprints specifically.

# Object pointer types

Before we go into good practices to avoid hard references, we need to understand the different object pointer types we can use in code.

## `TObjectPtr<T>` / Raw Pointers

These are our basic pointer types. `TObjectPtr<T>` or raw pointers are equivalent in the fact that they create a hard reference on other
objects. The former was introduces with UE5, and it adds some helpful features for instance tracking on editor and better null pointers debugging in non-shipping builds, but apart from that they are completely equivalent on the memory and referencing front on a shipping build. These should be the ones you should look out for, and use them only for things like component references for an actor, or other objects you are sure should be hard referenced.

{% highlight c++ %}
// These two are equivalent

UPROPERTY()
UActorComponent* Component{};

UPROPERTY()
TObjectPtr<UActorComponent> Component{};
{% endhighlight %}

> Prefer using `TObjectPtr` over raw pointers. It looks like the intention for Unreal is eventually getting rid of raw pointers in game code. Furthermore `TObjectPtr` might introduce new useful features for editor references in the future, so good practice to future-proof our code.

## `TWeakObjectPtr<T>`

This is an incredibly useful one. Say you want to reference an actor or an object you don’t want to have direct ownership over but you still need to reference. TWeakObjectPtr should be your go-to in this case. They are pretty much equivalent to C++ weak pointer types ([RAII](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization) etc.). The following is a fast example of how to use them:

{% highlight c++ %}
// MyClass.h
UPROPERTY()
TWeakObjectPtr<AOtherActor> OtherActorReference{};

// MyClass.cpp
void AMyClass::Foo()
{
	// Test if OtherActorReference is pointing to something valid
	TObjectPtr<AOtherActor> OtherActor = OtherActorReference.Get();
	if (!OtherActor)
	{
		return;
	}

	// do stuff with OtherActor...
}
{% endhighlight %}

This has also the added benefit that MyClass doesn’t contribute to the garbage collection of AOtherActor. Meaning that the AOtherActor instance might be garbage collected even if we are still referencing. This is useful and pretty desirable in a lot of cases.

## `TSoftObjectPtr<T>`

If there’s one thing you might want to take away from this article is the importance of **async loadings**. One good way to avoid hard references is soft referencing an object and async load it only when we need it. This is achieved by using a `TSoftObjectPtr<T>`. The idea is that we use a StreamableManager to async load one or a list of objects and call a delegate when the loading is complete. In the below example, we have a StreamableManager in our GameInstance, and we call it to do our async loadings:

{% highlight c++ %}
// MyClass.h
UPROPERTY()
TSoftObjectPtr<UStaticMesh> AsyncLoadedMesh{};

// MyClass.cpp
void AMyClass::LoadMesh()
{
	// Get streamable manager
	FStreamableManager& Manager = GetGameInstance<MyGameInstance>()->GetStreamableManager();
	FStreamableDelegate Delegate = FStreamableDelegate::CreateUObject(this, &ThisClass::LoadMeshDeferred);
	Manager.RequestAsyncLoad(AsyncLoadedMesh.ToSoftObjectPath(), Delegate);
}

// This will be called when the mesh has been loaded
void AMyClass::LoadMeshDeferred()
{
	if (AsyncLoadedMesh)
	{
		// DoStuff with AsyncLoadedMesh
	}
}
{% endhighlight %}

From a `TSoftObjectPtr`, you can also **sync load** the object to be used immediately using the `LoadSynchronous()` method on the pointer. This stalls the load queue, and should generally be avoided completely and used only during init operations of a class potentially (although if you need something alive for the entire lifetime of an object you probably want another type of pointer).

> Avoid sync loading objects.

# Blueprint soft object references types

There are ways to avoid hard referencing objects in blueprint too. These are **Soft Object References** and **Soft Class References**. The former is used for instanced objects, the latter is used for class assets, like a blueprint type.

<figure>
	<img src="/assets/img/bp_soft_types.png">
</figure>

Using this, you get access to the same type of async loading behaviour we saw with `TSoftObjectPtr<T>`, giving you the ability to async load an asset only when you actually need it.

<figure>
	<img src="/assets/img/bp_soft_casts.png">
</figure>