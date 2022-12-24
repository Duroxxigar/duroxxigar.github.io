---
title: Bitflags in Unreal C++
categories:
  - Unreal
---

You're in a situation where you want to combine multiple flags into one simple variable. A simple example we can look at to help illustrate this is the firemodes for a firearm. Bitflags to the rescue! 

As an example, we want to have a way to tell a firearm the firemodes that it has available for it to use. One way we can do this is having multiple boolean values such as `bHasSemi`, `bHastBurst`, `bHasAuto`. Then for each weapon, we could just flip the appropiate flag and check each value when switching firemodes. This approach will work but it can be a bit cumbersome and clunky when using the UI to create a new firearm. With bitflags, we can compact this down into one
variable and improve the UI experience as well.

To do so, we create an enum like so:

```cpp
UENUM(BlueprintType, meta=(Bitflags, UseEnumValuesAsMaskValuesInEditor="true"))
enum class EFireModes : uint8
{
  None = 0 UMETA(Hidden),
  Semi = 1,
  Burst = 2,
  Auto = 4
}

ENUM_CLASS_FLAGS(EFireModes)

// somewhere down in the actual class .h file
UPROPERTY(EditAnywhere, BlueprintReadWrite, meta=(Bitmask, BitmaskEnum=EFireModes))
int AvailableFireModes;
```

Pay attention to the `meta` part. Don't forget to add those, without them, Unreal won't know that you're trying to do some bitmask magic. Also pay attention with the way the values are set for the enum. They must be in a power of 2. An alternative way you may see it is with the bitshift operator, `1 << 2` to represent 4 as an example. I just prefer this way because it's easier to understand in my opinion. Do note, you **do** need something to represent the 0 value. I have opted to just hide it from the UI though, so it can't be selected.

Right after you declare the UENUM(), you will want the `ENUM_CLASS_FLAGS()` macro as well. This will allow you to use some convenience methods that Epic has wrote to help working with enums as bitmasks. We'll see how this comes into play later.

For the UPROPERTY, notice that you do have to tell Unreal what enum should be used for the bitmask.

If we want to assign a value to the `AvailableFireModes` variable, we would just do something like `AvailableFireModes = static_cast<uint8>(EFireModes::Auto);`. We want to do a cast here to convert to uint. We then just pass the enum value that we want in the parenthesis.

In order to actually do comparisons however, we need to define operators will work.

Just below the `EFireModes` enum declaration block, we can add the following

```cpp
inline bool operator== (int a, EFireModes b)
{
	return a == static_cast<uint8>(b);
}
```

Now, whenever we do an equality comparison, it will execute this code. Keep in mind that the comparison needs to be in the order that you define it. This means that you must do the equality comparison as `int == EFireModes`. You'll have to define the other way as well in order to allow both ways.

To check if the `int` property has any of the bitflags, we can use `EnumHasAnyFlags(static_cast<EFireModes>(AvailableFireModes), EFireModes::Semi);`. This is where the `ENUM_CLASS_FLAGS()` comes into play. There are some others that you can use as well.

You can still have a property with the type `EFireModes`. An example would be:

```cpp
EFireModes CurrentFireMode = EFireModes::Semi;

// to set it according to the bitflag
if (EnumHasAnyFlags(static_cast<EFireModes>(AvailableFireModes), EFireModes::Auto))
{
  CurrentFireMode = EFireModes::Auto;
}
```

For a more in-depth post about bitflags in general, I've found that this post is quite helpful: [Using bit flags in c++](https://tackytortoise.github.io/2020/11/26/using-bitflags-in-cpp.html)