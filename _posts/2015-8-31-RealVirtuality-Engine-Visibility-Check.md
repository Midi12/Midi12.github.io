---
layout: post
title: RealVirtuality Engine Visibility checks (ArmA 2)
orig_link: https://www.unknowncheats.me/forum/arma-2/156279-engine-visibility-checks.html
---

Reversing visibility checks in RealVirtuality engine.

---

*All content below is for educational purpose only*

<a href="{{ page.orig_link }}" target="_blank">Original link</a>

---

Visibility checks are interesting features in ArmA series because of the kind of the game (long range shots, buildings, trees ...).

<div style='position:relative; padding-bottom:calc(62.04% + 44px)'><iframe src='https://gfycat.com/ifr/AppropriateWellmadeIndianrockpython' frameborder='0' scrolling='no' width='100%' height='100%' style='position:absolute;top:0;left:0;' allowfullscreen></iframe></div>

I am going to show you how to implement one of the multiple ways of doing this kind of stuff.

## What do we need ?

There is two major functions for doing visibility checks :

```cpp
enum class Functions
{
    Landscape__IntersectWithGroundOrSea = 0x0044F3B0,
    Landscape__ObjectCollisionLine = 0x004458E0,
};
```

And two other functions in charge of allocationg & freeing the collision buffer in charge of storing collision results :

```cpp
enum class Functions
{
    CollisionBuffer__Alloc = 0x0044FEF0,
    CollisionBuffer__Free = 0x0044FF60,
};
```

```cpp
typedef double (__thiscall *fnLandscape__IntersectWithGroundOrSea_t)(
    unsigned int this_,
    Math::Vector3 *ret,
    Math::Vector3 *from,
    Math::Vector3 *dir,
    float minDist,
    float maxDist
);
```

```cpp
typedef int (__thiscall *fnLandscape__ObjectCollisionLine_t)(
    unsigned int this_,
    float age,
    CollisionBuffer *res,
    IntersectionFilter *f,
    Math::Vector3 *beg,
    Math::Vector3 *end,
    float radius,
    ObjIntersect a8,
    ObjIntersect a9
);

typedef CollisionBuffer *(__thiscall *fnCollisionBuffer__Alloc_t)(CollisionBuffer *this_);
 
typedef void(__thiscall *fnCollisionBuffer__Free_t)(CollisionBuffer *this_);
```

So now we still need some elements in order to make this working, the Landscape class pointer, the `CollisionBuffer` struct and the IntersectionFilter struct.

As said earlier the `CollisionBuffer` struct is in charge of storing collisions results while the `IntersectionFilter` struct is in charge of filtering units we dont want to get the collisions recorded (in this case it will be our player and our target.

```cpp
struct CollisionInfo
{
    void *texture;
    void *surface;
    Math::Vector3 pos;
    Math::Vector3 dirOut;
    Math::Vector3 dirOutNotNorm;
    int hierLevel;
    Object *object;
    Object *parentObject;
    float under;
    int component;
    bool entry;
    bool exit;
    int geomLevel;
};

struct CollisionBuffer
{
    CollisionInfo *_data;
    int _n;
    BYTE gap0[1]; //alignment ??
    unsigned int _oldSP;
    void *_buffer;
    bool _memused;
    int maxN;
};
```
```cpp
struct IntersectionFilterVtbl
{
    bool(__thiscall *Filter)(void *this_, void *object);
};

struct IntersectionFilter
{
    IntersectionFilterVtbl *vfptr;
};

struct Landscape__FilterIgnoreTwo : IntersectionFilter
{
    Object *_obj1;
    Object *_obj2;
};
```

You need to initialize the vfptr field with the according filter pointer :

```cpp
enum class VirtualTables
{
    Landscape__FilterIgnoreTwo = 0x00C36BE4,
};
```

The Landscape class pointer can be grabbed in two ways :

Through his static pointer :

```cpp
enum class Statics
{
    Landscape = 0x00DBE1C0,
};

uintptr_t landscape = *reinterpret_cast<uintptr_t*>(static_cast<uintptr_t>(Offsets::Statics::Landscape));
```

Or the Scene static pointer

```cpp
enum class Statics
{
    Scene = 0x00DD32F4,
};
 
enum class Scene
{
    Landscape = 0x574,
};
 
uintptr_t landscape = *reinterpret_cast<uintptr_t*>(*reinterpret_cast<uintptr_t*>(Offsets::Statics::Scene) + static_cast<uintptr_t>(Offsets::Scene::Landscape));
```

How to check visibility ?

Quick overview :
- Get Landscape class pointer
- Get start and end world positions
- Get direction between start and end world positions
- Normalize it
- Check for terrain collisions
- Allocate CollisionBuffer
- Check for objects collisions
- Free CollisionBuffer
- Interpret results

```cpp
bool Engine::TraceLineCollision(const Object& target)
{
    if (!ObjectManager::GetInstance().IsSceneReady())
        return false;

    uintptr_t landscape = *reinterpret_cast<uintptr_t*>(*reinterpret_cast<uintptr_t*>(Offsets::Statics::Scene) + static_cast<uintptr_t>(Offsets::Scene::Landscape));
    
    if (landscape == 0)
        return false;

    Object& localPlayer = ObjectManager::GetInstance().GetLocalPlayer();
    Math::Vector3 from = ObjectManager::GetInstance().GetCamera().GetPosition();

    Math::Vector3 to = target.GetAimingPosition();

    Math::Vector3 direction = to - from;
    
    float dirSize = direction.Size();
    
    Math::Vector3 normalizedDir = direction.Normalize();

    float t = fnLandscape__IntersectWithGroundOrSea(
        landscape,
        nullptr,
        &from,
        &normalizedDir,
        0.0f,
        dirSize 
        );

    if (dirSize > t) //we hit terrain
        return true; 

            
    CollisionBuffer colResults;
    fnCollisionBuffer__Alloc(&colResults);

    Landscape__FilterIgnoreTwo filter;
    filter.vfptr = reinterpret_cast<IntersectionFilterVtbl *>(Offsets::VirtualTables::Landscape__FilterIgnoreTwo);
    filter._obj1 = &localPlayer;
    filter._obj2 = &target;

    fnLandscape__ObjectCollisionLine(
        landscape,
        0.0f,
        &colResults,
        &filter,
        &from,
        &to,
        0.0f,
        1,
        4
        );
    
    bool hit = colResults._n != 0; //we hit objects

    fnCollisionBuffer__Free(&colResults);

    return hit;
}
```

You can also get collisions positions through CollisionBuffer._data.