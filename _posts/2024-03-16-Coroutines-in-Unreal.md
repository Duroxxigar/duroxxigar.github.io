---
title: Coroutines in Unreal
excerpt: Coroutines in Unreal
categories:
  - Unreal
---

This post isn't meant to cover everything about using coroutines in general. We're just going to be getting up and going to show how you can change your approach to writting C++ code in Unreal 5. The plugin that we're going to be using is [UE5Coro](https://github.com/landelare/ue5coro). Definitely read the docs of the plugin as well - they are very good! If you want a more in-depth introduction to asynchronous code in general, [C#'s docs](https://learn.microsoft.com/en-us/dotnet/csharp/asynchronous-programming/) on it is actually pretty good.

The general gist is that you can write synchronous looking code, but it isn't synchronous. One of the advantages is that you have all of your logic in one place, instead of the alternative where you would be splitting your logic up in multiple callback functions. One disadvantage is that debugging can be more difficult because of lifetimes and all that. For the beginner, it may also be a bit more difficult to understand how the code actually flows.

To help understand the "flow" of the code, I'll lean onto the author of the library's explanation. It is quite possibly one of the clearest ways I've seen it explained.

"Regular C++ functions are **sub**routines. Meaning that you go into the function, it does stuff, and you come out of it. Basic stuff, nothing fancy here. What a coroutine lets you do is to have more control over this "in and out" thing. You can go into the function, come out of it, but then decide to go back into the same call instead of calling it again." - Laura

Put simply, when the function encounters the `co_await` keyword, it will leave the function and go off and do other things. When whatever you are `co_await`ing is finished, it will return back to the same function call and resume execution. This "leaving the function" part is what can cause some headaches though, because when you return, some objects may be gone or collected by Unreal.

Now when doing coroutines with this plugin, the coroutine can be in one of two modes. **Latent** or **Asynchronous**. Latent means that it hooks into Unreal's built-in latent system (so think of any of those nodes you see that have the little clock in the top right) and Unreal manages it. Utilizing this mode, it makes it a bit easier to dip your feet into writing coroutines. This is because Unreal will manage lifetimes of objects, as well as cancel the coroutine when necessary. On the flipside, Asynchronous mode is when it's not really hooked into Unreal's latent system. So it manages itself. This means it is up to you to clean up properly and manage lifetimes.

## Latent ##

In order to be a latent coroutine, the function needs to have either a `FLatentActionInfo` or `FForceLatentCoroutine` parameter. If you want to expose this as a blueprint node, it needs to return `FAsyncCoroutine` and then do your typical `UFUNCTION()` specifiers.

```cpp
UFUNCTION(BlueprintCallable, meta=(Latent, LatentInfo="Info"))
FAsyncCoroutine ACustomActor::ExampleLatent(FLatentActionInfo Info);

// in the .cpp
FAsyncCoroutine ACustomActor::ExampleLatent(FLatentActionInfo Info)
{
	GEngine->AddOnScreenDebugMessage(-1, 3.0f, FColor::Yellow, "Before co_await");
	co_await UE5Coro::Latent::Seconds(2.0f);
	GEngine->AddOnScreenDebugMessage(-1, 3.0f, FColor::Yellow, "After co_await");
}
```
![Node in BP](Pics/ue5-coro-started/example-latent.png)

The `FAsyncCoroutine` struct is ignored for BP and is hidden. The great thing is that the `co_await` is **not** going to block the gamethread! And yes, this is how you can do a "delay" in C++ now! Compare that to the non-coroutine way:

```cpp
// in the .h file
FTimerHandle TimeoutTimerHandle;
void OnTimeout();
void SomeOtherMethod();

// in the .cpp file
void AExampleActor::SomeOtherMethod()
{
    GEngine->AddOnScreenDebugMessage(-1, 3.0f, FColor::Yellow, "Before timer");
    GetWorld()->GetTimerManager().SetTimer(TimeoutTimerHandle, this, &ACustomActor::OnTimeout, Time, false);   
}

void AExampleActor::OnTimeout()
{
    GEngine->AddOnScreenDebugMessage(-1, 3.0f, FColor::Yellow, "After timer");
}
```

Now, admittingly, you can also wrap this in a lambda as well, so it is all in the same space. It is still more verbose though. Besides, this previous syntax is the more common approach.

If you've ever tried to do a timeline in C++, you'll know that it is quite painful. With UE5Coro, we can just do the following:

```cpp
co_await UE5Coro::Latent::Timeline(WorldContextObject, From, To, Length, [](double interpolatedValue) -> void 
{
    // write your timeline logic here - remember to do the proper captures/params for your lambda!
});
```

Even async loading objects is far more simple and straightforward.

```cpp
// .h
UPROPERTY(EditAnywhere)
TSoftObjectPtr<UStaticMesh> SoftMesh;

// .cpp
UStaticMesh* MyMesh = co_await UE5Coro::Latent::AsyncLoadObject(SoftMesh);
```

It also has a templated variant

```cpp
auto* MyMesh = co_await UE5Coro::Latent::AsyncLoadObject<UStaticMesh>(SoftMesh);
```

Now let's take a look at how you can do the same thing, but using the [classic approach](https://docs.unrealengine.com/5.3/en-US/asynchronous-asset-loading-in-unreal-engine/) with callback functions.

```cpp
void UGameCheatManager::GrantItems()
{
    // you have to set up your own UGameGlobals!
    FStreamableManager& Streamable = UGameGlobals::Get().StreamableManager;
    Streamable.RequestAsyncLoad(SoftMesh.ToSoftObjectPath(), FStreamableDelegate::CreateUObject(this, &UGameCheatManager::GrantItemsDeferred));
}

void UGameCheatManager::GrantItemsDeferred()
{
    // we have the UStaticMesh now, so we can do w/e with it
}
```

We can even load primary assets quite easily with UE5Coro. First I'm going to show the classic approach. Here is a snippet of code I'm using as an example from Tom Looman's [blog post](https://www.tomlooman.com/unreal-engine-asset-manager-async-loading/) about the asset manager for Unreal.

```cpp
// Get the Asset Manager from anywhere
if (UAssetManager* Manager = UAssetManager::GetIfValid())
{
    // Monster Id taken from a DataTable
    FPrimaryAssetId MonsterId = SelectedMonsterRow->MonsterId;

    // Optional "bundles" like "UI"
    TArray<FName> Bundles;

    // Locations array from omitted part of code (see github)
    FVector SpawnLocation = Locations[0]; 

    // Delegate with parameters we need once the asset had been loaded such as the Id we loaded and the location to spawn at. Will call function 'OnMonsterLoaded' once it's complete.
    FStreamableDelegate Delegate = FStreamableDelegate::CreateUObject(this, &ASGameModeBase::OnMonsterLoaded, MonsterId, SpawnLocation);
    
    // The actual async load request
    Manager->LoadPrimaryAsset(MonsterId, Bundles, Delegate);
}

void ASGameModeBase::OnMonsterLoaded(FPrimaryAssetId LoadedId, FVector SpawnLocation)
{
    UAssetManager* Manager = UAssetManager::GetIfValid();
    if (Manager)
    {
        USMonsterData* MonsterData = Cast<USMonsterData>(Manager->GetPrimaryAssetObject(LoadedId));

        if (MonsterData)
        {
            AActor* NewBot = GetWorld()->SpawnActor<AActor>(MonsterData->MonsterClass, SpawnLocation, FRotator::ZeroRotator);
        }
    }
}
```

Now let's rewrite this to be used with coroutines in UE5Coro. Imagine this as one function though, because there is code omitted as Tom mentioned and I did not check the github like he said to do. 

```cpp
// Monster Id taken from a DataTable
FPrimaryAssetId MonsterId = SelectedMonsterRow->MonsterId;

// Optional "bundles" like "UI"
TArray<FName> Bundles;
auto* MonsterData = co_await UE5Coro::Latent::AsyncLoadPrimaryAsset<USMonsterData>(MonsterId, Bundles);

if (MonsterData)
{
    AActor* NewBot = GetWorld()->SpawnActor<AActor>(MonsterData->MonsterClass, Locations[0], FRotator::ZeroRotator);
}
```

![Fraction of our power](Pics/ue5-coro-started/omni-man-invincible.gif)

## Asynchronous ##

In asynchronous mode, it is up to the user to manage certain aspects of the coroutine. That said, for the most part, it's pretty straightforward. Just check that something is valid after each `co_await` call.

```cpp
TCoroutine<> ABlah::Thing(AActor* otherActor)
{
    otherActor->Foo();
    co_await UE5Coro::Http::ProcessAsync(SomeRequest);
    // no latent protection - so "this" and "otherActor" may be collected by now and could cause a crash!
    otherActor->FooTwo();
}
```

The way to handle this would be:

```cpp
TCoroutine<> ABlah::Thing(AActor* otherActor)
{
    otherActor->Foo();
    TWeakObjectPtr<AActor> otherActorWeak(otherActor);
    TWeakObjectPtr<AActor> meWeak(this);
    co_await UE5Coro::Http::ProcessAsync(SomeRequest);
    // no latent protection - so "this" and "otherActor" may be collected by now and could cause a crash!

    if (!meWeak.Get() || !otherActorWeak.Get())
        co_return;
    
    otherActor->FooTwo();
}
```

However, when awaiting a latent awaiter, they generally have Latent protections. Meaning that if the object owning the coroutine is collected, the coroutine is not resumed.

```cpp
TCoroutine<> ABlah::Thing(AActor* otherActor)
{
    otherActor->Foo();
    co_await UE5Coro::Latent::Seconds(3);
    // latent protection, so "this" and "otherActor" are valid in the context of the coroutine (not counting standard IsValid() checks)

    if (IsValid(otherActor))
    {
        otherActor->FooTwo();
    }
}
```

The [docs](https://github.com/landelare/ue5coro/blob/master/Docs/Async.md#coroutines-and-uobject-lifetimes) have even more examples. Even talks about issues that can happen with Latent coroutines.

Another plugin that I can recommend is this [plugin here](https://github.com/redxdev/MAPlugins). Check out the sections about Object Referencers and UE5Coro.

That about covers some of the basics and should help get you up and going with coroutines in Unreal Engine. There is still plenty of stuff to cover and I would, again, **highly** encourage you to read the [UE5Coro docs](https://github.com/landelare/ue5coro/blob/master/Docs). Once again, my advice to get your toes dipped in gently, is to stick with doing Latent coroutines. It helps ease the transition to the asynchronous way to write code.