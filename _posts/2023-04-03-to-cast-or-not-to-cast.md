---
layout: post
title:  "To Cast or not to Cast"
date:   2023-04-03
excerpt: "A little deep dive into UE's UObject Casts"
tag:
- blog
- c++
---

{% include toc.html %}

# Introduction

One constantly found code paradigm in Unreal Engine's C++ is this one:
{% highlight c++ %}
// snippet #1
void SomeClass::DoThings(AActor* SomeActor)
{
	ASomeDerivedActorClass* CastedActor = Cast<ASomeDerivedActorClass>(SomeActor);
	if (!CastedActor)
	{
		// you might want to log something here
		return; 
	}
	// do things with CastedActor...
}
{% endhighlight %}

What this code does is pretty straightforward; we have a pointer to some actor, we cast it into the type we need it and we do things with this actor. All UObjects are inherently polymorphic due to how Unreal generates reflection data for the types, thus dynamic casts of this type are very common and are pretty useful to catch errors or branch your code behaviour depending on the type of object you are working with. If you're a C++ programmer outside of UE's API, you might have heard that using *dynamic_cast*, or [RTTI](https://en.wikipedia.org/wiki/Run-time_type_information) (i.e. working with polymorphic types, which requires the compiler to generate additional metadata during compilation) in general can be pretty performance-heavy. This is debatable and it depends a lot on platform and compiler implementation and is outside the scope of this article in particular. When it comes to UE and **UObjects**, though, the code has its own metadata generated using the **UnrealHeaderTool** (UHT), and since we have this data and can look at how this is used in the context of castings, we can try to take a look inside the engine code and see what Unreal does under the hood to optimise castings, and potentially derive some good or bad practice out of this.

We will only investigate the case of *UObject* to *UObject* casting, *UInterfaces* are handled differently (we might go into them some other time).

# So, what metadata does UHT generates for us?
*A lot.*
But for our needs, we only need to care about the fact that every UCLASS marked class, when parsed by UHT, will have access to the following methods and fields (the latter is declared in those *generated.h* files you constantly forget to include):
{% highlight c++ %}
// Note: Syntax simplified for explanation purposes

UClass* ClassPrivate;

UClass* GetClass();

template<class T>  
bool IsA() const;

static UClass* StaticClass();
{% endhighlight %}
Now, these methods can be used at runtime to get the *UClass* for an object or a type, and it's pretty much what lets us being able to get RTTI for Unreal Engine *UObjects* directly in the class structure. This is also at the heart of how UE's dynamic casting works. Note that this RTTI implementation doesn't rely on virtual tables for type identification and casting; here `GetClass()` will simply return `ClassPrivate`. *UClass* is too an *UObject* type, and has access to the same type of reflection and casting we can use for *UObjects*. Finally, *UClass* have a pointer to their base class if any (*UObject* types don't support multiple inheritance, apart from when using *UInterfaces*).
# Different ways to Cast in UE

Now that we know what tools we have to understand the types of our *UObjects*, let's see the different methods of casting in UE.

## _ExactCast_
This is a simple and nice one. *ExactCast* can be used when we are sure of the type we are dealing with, but we still want to be able to check against *nullptr*. The implementation is the following:
{% highlight c++ %}
template< class T >  
FORCEINLINE T* ExactCast( UObject* Src )  
{  
	return Src && (Src->GetClass() == T::StaticClass()) ? (T*)Src : nullptr;  
}
{% endhighlight %}
If we are dealing with *UObjects* types this is a great alternative to C-like casts or C++ *static_cast*. We have the RTTI, and we have no reason to not exploit that info for a fast check even if we need no polymorphism at all.

## _Cast_
This is the meaty and most important one. *Cast* is our *dynamic_cast* equivalent, checking the type against the inheritance chain at runtime and letting us exploit polymorphism to its fullest. But how does Unreal do it, and does it do any optimisation exploiting the type info we have? Let's see:
{% highlight c++ %}
// The original Cast method will eventually expand into the one below.
// SFINAE is used to determine what type of ECastType we are dealing with

template <typename From, typename To>  
struct TCastImpl<From, To, ECastType::UObjectToUObject>  
{  
	FORCEINLINE static To* DoCast( UObject* Src )  
	{  
		return Src && Src->IsA<To>() ? (To*)Src : nullptr;  
	}

	[...]
}
{% endhighlight %}
As we could have expected, the real meat is in the *IsA* method, so let's look into that:
{% highlight c++ %}
// The template version called above will expand into this:
template <typename OtherClassType>  
FORCEINLINE bool IsA( OtherClassType SomeBase ) const  
{  
	// We have a cyclic dependency between UObjectBaseUtility and UClass,  
	// so we use a template to allow inlining of something we haven't yet seen, because it delays compilation until the function is called.  
	// 'static_assert' that this thing is actually a UClass pointer or convertible to it.  
	const UClass* SomeBaseClass = SomeBase;  
	(void)SomeBaseClass;  
	checkfSlow(SomeBaseClass, TEXT("IsA(NULL) cannot yield meaningful results"));  

	const UClass* ThisClass = GetClass();  

	// Stop the compiler doing some unnecessary branching for nullptr checks  
	UE_ASSUME(SomeBaseClass);  
	UE_ASSUME(ThisClass);  

	return IsChildOfWorkaround(ThisClass, SomeBaseClass);  
}
{% endhighlight %}
Well, this is not too helpful. This is still mostly boilerplate and checks (*void* casts are usually added to suppress compiler warnings about unused variables, although I'm not sure why it's added here). We need to go *deeper*.
{% highlight c++ %}
template <typename ClassType>  
static FORCEINLINE bool IsChildOfWorkaround(const ClassType* ObjClass, const ClassType* TestCls)  
{  
   return ObjClass->IsChildOf(TestCls);  
}
{% endhighlight %}
Ok, this makes sense, as we said we know that *UClasses* have a pointer to their base class, so it makes sense for them to have a method that checks whether a class is a child of another class. Let's see what the *IsChildOf* method does:
{% highlight c++ %}
// Code simplified for explanation purposes

/** Returns true if this struct either is SomeBase, or is a child of SomeBase. This will not crash on null structs */  
#if USTRUCT_FAST_ISCHILDOF_IMPL == USTRUCT_ISCHILDOF_OUTERWALK  
	bool IsChildOf( const UStruct* SomeBase ) const;  
#else  
	bool IsChildOf(const UStruct* SomeBase) const  
	{  
	    return (SomeBase ? IsChildOfUsingStructArray(*SomeBase) : false);  
	}  
#endif
{% endhighlight %}
Stuff is finally getting interesting, here we can see that UE defines the *IsChildOf* method depending on a preprocessor definition. UE has two implementations of the *IsChildOf* methods, one safer and used in editor builds, the other faster and used in builds. These methods are defined in the *UStruct* class, which is the base class of *UClass*; for our purposes the two are interchangeable. Let's now look at these two different implementations:

### *USTRUCT_ISCHILDOF_OUTERWALK*
This is the safer method and it's used when we are running the editor. The implementation is pretty straightforward, and we simply walk the inheritance chain of the *UClasses* until we find a match, or return false otherwise.
{% highlight c++ %}
// Method simplified for explanation purposes
bool UStruct::IsChildOf( const UStruct* SomeBase ) const  
{
	[...]
	
	if (SomeBase == nullptr)  
	{  
		return false;  
	}  

	bool bResult = false;  
	for ( const UStruct* TempStruct=this; TempStruct; TempStruct=TempStruct->GetSuperStruct() )  
	{  
		if ( TempStruct == SomeBase )  
		{  
			bResult = true;  
			break;  
		}  
	}  

	[...]
	
	return bResult ;  
}
{% endhighlight %}
Easy and nice, this is _O(n)_, where _n_ is the depth of the inheritance tree. It is pretty much as efficient as you can get with an enforced single inheritance naive dynamic cast algorithm. But can we do better? Does Unreal add any other optimisation? That's where the second *IsChildOf* comes in.
### *USTRUCT_ISCHILDOF_STRUCTARRAY*
Here is the implementation for the second method, which is used in non-editor builds:
{% highlight c++ %}
bool IsChildOf(const UStruct* SomeBase) const  
{  
	return (SomeBase ? IsChildOfUsingStructArray(*SomeBase) : false);  
}

FORCEINLINE bool IsChildOfUsingStructArray(const FStructBaseChain& Parent) const  
{  
	int32 NumParentStructBasesInChainMinusOne = Parent.NumStructBasesInChainMinusOne;  
	return NumParentStructBasesInChainMinusOne <= NumStructBasesInChainMinusOne && StructBaseChainArray[NumParentStructBasesInChainMinusOne] == &Parent;  
}
{% endhighlight %}
This is definitely more interesting. First of all, we can notice that now the check is simply one array access and one equality check, i.e. _O(1)_! _FStructBaseChain_ is a private *UStruct* superclass that keeps an array of pointers to the base classes of a class and the number of the inheritance chain. Below is a simple graph representation of how this works:

<figure>
	<img src="/assets/img/cast_array.jpg">
</figure>

Therefore, say we want to test if *UClass* D is a child of *UClass* B, we would have the following:
{% highlight c++ %}
FORCEINLINE bool IsChildOfUsingStructArray(const FStructBaseChain& Parent) const  
{  
	// this = D, Parent = B
	// NumStructBasesInChainMinusOne = 3
	// NumParentStructBasesInChainMinusOne = 1
	// StructBaseChainArray[1] = B
	// returns true
	int32 NumParentStructBasesInChainMinusOne = Parent.NumStructBasesInChainMinusOne;  
	return NumParentStructBasesInChainMinusOne <= NumStructBasesInChainMinusOne && StructBaseChainArray[NumParentStructBasesInChainMinusOne] == &Parent;  
}
{% endhighlight %}
Neat! This information is part of the metadata generated for *UClasses*, and it's serialised in non-editor builds. This is pretty much how Unreal deals with fast casting! The reason why this is not used in editor builds is that this could easily break when re-instancing blueprints or hot-patching data.
## _CastChecked_
We are almost at the end! What if you want to dynamically cast an object but you are not planning to branch in case the cast fails? In fact, you pretty much would prefer the program to crash in that case, so that you can fix the data properly. Well, that's where *CastChecked* comes in:
{% highlight c++ %}
template <typename To, typename From>  
To* CastChecked(From* Src, ECastCheckedType::Type CheckType)  
{  
	if (Src)  
	{  
		To* Result = Cast<To>(Src);  
		if (!Result)  
		{  
		   CastLogError(*GetFullNameForCastLogError(Src), *GetTypeName<To>());  
		}  

		return Result;  
	}  

	if (CheckType == ECastCheckedType::NullChecked)  
	{  
		CastLogError(TEXT("nullptr"), *GetTypeName<To>());  
	}  

	return nullptr;  
}
{% endhighlight %}
This simply will use the normal *Cast* method and throw a fatal error in case the dynamic cast fails. `ECastCheckedType` can be used to also throw a fatal error in case we pass in a *nullptr* object.

Now, this might not be super useful at first. Indeed this does exactly what the other *Cast* method does, with the only difference of crashing our engine if the cast fails. But here's the cool thing: as you might know, checks are stripped away of *Test* and *Shipping* builds, and in those builds, our implementation of *CastChecked* will instead expand to the following:
{% highlight c++ %}
// CastChecked will eventually expand into this when checks are stripped out, believe me
FORCEINLINE static To* DoCastCheckedWithoutTypeCheck( UObject* Src )  
{  
	return (To*)Src;  
}
{% endhighlight %}
Now, this is interesting; it's just a raw cast! You cannot really do it faster than this. The idea is that since you've most likely crashed (hopefully not too many times) while fixing the data in development, on *Test* and *Shipping* you can now have the guarantee and peace of mind that the data is correct and the cast will be successful.  This is a nice optimisation, but of course, it's very important for this to be used only with casts that shouldn't fail at all. 
# The bottom line
We've reached the end! Let's use this section to review the type of casts we saw and when we really should use them:

Type of cast | Speed | Usage
-------- | ----- | ------
`CastChecked<T>()` | Fastest on Test/Shipping builds. As fast as `Cast<T>()` otherwise | Perfect when we don't want to handle the failed cast and we want to make sure that the data is correct during development.
`ExactCast<T>()` | Faster than `Cast<T>()` and slower than `CastChecked<T>()` on Test/Shipping builds | Perfect replacement for static casts when we deal with UObjects. To be used when we are sure of the type we are dealing with.
`Cast<T>()` | Slowest of the three. Optimised on non-editor builds.  | For all the other scenarios, this should be your go-to.
{: rules="groups"}

Hopefully, this was informative! Feel free to get in touch with a mail if you spot any mistakes or if you have any feedback, I might be implementing a comment section to the posts... eventually.