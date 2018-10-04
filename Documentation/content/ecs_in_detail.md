# ECS features in detail

## Entity

__Entity__ is an ID. You can think of it as a super lightweight __GameObject__ that does not even have a name by default.

You can add and remove components from Entities at runtime. Entity ID's are stable. In fact they are the only stable way to store a reference to another component or Entity.

## IComponentData

Traditional Unity components (including __MonoBehaviour__) are [object-oriented](https://en.wikipedia.org/wiki/Object-oriented_programming) classes which contain data and methods for behavior. __IComponentData__ is a pure ECS-style component, meaning that it defines no behavior, only data. IComponentDatas are structs rather than classes, meaning that they are copied [by value instead of by reference](https://stackoverflow.com/questions/373419/whats-the-difference-between-passing-by-reference-vs-passing-by-value?answertab=votes#tab-top) by default. You will usually need to use the following pattern to modify data:

```C#
var transform = group.transform[index]; // Read

transform.heading = playerInput.move; // Modify
transform.position += deltaTime * playerInput.move * settings.playerMoveSpeed;

group.transform[index] = transform; // Write
```

IComponentData structs may not contain references to managed objects. Since the all component data lives in simple non-garbage-collected tracked [chunk memory](https://en.wikipedia.org/wiki/Chunking_(computing)).

## EntityArchetype

An __EntityArchetype__ is a unique array of __ComponentType__. __EntityManager__ uses EntityArchetypes to group all Entities using the same ComponentTypes in chunks.

```C#
// Using typeof to create an EntityArchetype from a set of components
EntityArchetype archetype = EntityManager.CreateArchetype(typeof(MyComponentData), typeof(MySharedComponent));

// Same API but slightly more efficient
EntityArchetype archetype = EntityManager.CreateArchetype(ComponentType.Create<MyComponentData>(), ComponentType.Create<MySharedComponent>());

// Create an Entity from an EntityArchetype
var entity = EntityManager.CreateEntity(archetype);

// Implicitly create an EntityArchetype for convenience
var entity = EntityManager.CreateEntity(typeof(MyComponentData), typeof(MySharedComponent));

```

## EntityManager

The EntityManager owns __EntityData__, EntityArchetypes, __SharedComponentData__ and __ComponentGroup__.

EntityManager is where you find APIs to create Entities, check if an Entity is still alive, instantiate Entities and add or remove components.

```cs
// Create an Entity with no components on it
var entity = EntityManager.CreateEntity();

// Adding a component at runtime
EntityManager.AddComponent(entity, new MyComponentData());

// Get the ComponentData
MyComponentData myData = EntityManager.GetComponentData<MyComponentData>(entity);

// Set the ComponentData
EntityManager.SetComponentData(entity, myData);

// Removing a component at runtime
EntityManager.RemoveComponent<MyComponentData>(entity);

// Does the Entity exist and does it have the component?
bool has = EntityManager.HasComponent<MyComponentData>(entity);

// Is the Entity still alive?
bool has = EntityManager.Exists(entity);

// Instantiate the Entity
var instance = EntityManager.Instantiate(entity);

// Destroy the created instance
EntityManager.DestroyEntity(instance);
```

```cs
// EntityManager also provides batch APIs
// to create and destroy many Entities in one call. 
// They are significantly faster 
// and should be used where ever possible
// for performance reasons.

// Instantiate 500 Entities and write the resulting Entity IDs to the instances array
var instances = new NativeArray<Entity>(500, Allocator.Temp);
EntityManager.Instantiate(entity, instances);

// Destroy all 500 entities
EntityManager.DestroyEntity(instances);
```

## Chunk implementation detail

The ComponentData for each Entity is stored in what we internally refer to as a chunk. ComponentData is laid out by stream. Meaning all components of type A, are tightly packed in an array. Followed by all components of type B etc.

A chunk is always linked to a specific EntityArchetype. Thus all Entities in one chunk follow the exact same memory layout. When iterating over components, memory access of components within a chunk is always completely linear, with no waste loaded into cache lines. This is a hard guarantee.

__ComponentDataArray__ is essentially a convenience index based iterator for a single component type;
First we iterate over all EntityArchetypes compatible with the ComponentGroup; for each EntityArchetype iterating over all chunks compatible with it and for each chunk iterating over all Entities in that chunk.

Once all Entities of a chunk have been visited, we find the next matching chunk and iterate through those Entities.

When Entities are destroyed, we move up other Entities into its place and then update the Entity table accordingly. This is required to make a hard guarantee on linear iteration of Entities. The code moving the component data into memory is highly optimized.

## World

A __World__ owns both an EntityManager and a set of __ComponentSystems__. You can create as many Worlds as you like. Commonly you would create a simulation World and rendering or presentation World.

By default we create a single World when entering Play Mode and populate it with all available ComponentSystems in the project, but you can disable the default World creation and replace it with your own code via a global define.

* [Default World creation code](../../ECSJobDemos/Packages/com.unity.entities/Unity.Entities.Hybrid/Injection/DefaultWorldInitialization.cs)
* [Automatic bootstrap entry point](../../ECSJobDemos/Packages/com.unity.entities/Unity.Entities.Hybrid/Injection/AutomaticWorldBootstrap.cs) 

> Note: We are currently working on multiplayer demos, that will show how to work in a setup with separate simulation & presentation Worlds. This is a work in progress, so right now have no clear guidelines and are likely missing features in ECS to enable it. 

## System update order

In ECS all systems are updated on the main thread. The order in which the are updated is based on a set of constraints and an optimization pass which tries to order the system in a way so that the time between scheduling a job and waiting for it is as long as possible.
The attributes to specify update order of systems are ```[UpdateBefore(typeof(OtherSystem))]``` and ```[UpdateAfter(typeof(OtherSystem))]```. In addition to update before or after other ECS systems it is possible to update before or after different phases of the Unity PlayerLoop by using typeof([UnityEngine.Experimental.PlayerLoop.FixedUpdate](https://docs.unity3d.com/2018.1/Documentation/ScriptReference/Experimental.PlayerLoop.FixedUpdate.html)) or one of the other phases in the same namespace.

The UpdateInGroup attribute will put the system in a group and the same __UpdateBefore__ and __UpdateAfter__ attributes can be specified on a group or with a group as the target of the before/after dependency.

To use __UpdateInGroup__ you need to create and empty class and pass the type of that to the __UpdateInGroup__ attribute
```cs
public class UpdateGroup
{}

[UpdateInGroup(typeof(UpdateGroup))]
class MySystem : ComponentSystem
```

## Automatic job dependency management (JobComponentSystem)

Managing dependencies is hard. This is why in __JobComponentSystem__ we are doing it automatically for you. The rules are simple: jobs from different systems can read from IComponentData of the same type in parallel. If one of the jobs is writing to the data then they can't run in parallel and will be scheduled with a dependency between the jobs.
管理依赖关系很难。 这就是__JobComponentSystem__中我们为您自动完成的原因。 规则很简单：来自不同系统的作业可以并行读取相同类型的IComponentData。 如果其中一个作业正在写入数据，那么它们将无法并行运行，并且将在作业之间进行相关性调度。

```cs
public class RotationSpeedSystem : JobComponentSystem
{
    [BurstCompile]
    struct RotationSpeedRotation : IJobProcessComponentData<Rotation, RotationSpeed>
    {
        public float dt;

        public void Execute(ref Rotation rotation, [ReadOnly]ref RotationSpeed speed)
        {
            rotation.value = math.mul(math.normalize(rotation.value), quaternion.axisAngle(math.up(), speed.speed * dt));
        }
    }

    // Any previously scheduled jobs reading/writing from Rotation or writing to RotationSpeed 
    // will automatically be included in the inputDeps dependency.
    protected override JobHandle OnUpdate(JobHandle inputDeps)
    {
        var job = new RotationSpeedRotation() { dt = Time.deltaTime };
        return job.Schedule(this, 64, inputDeps);
    } 
}
```

### How does this work?

All jobs and thus systems declare what ComponentTypes they read or write to. As a result when a JobComponentSystem returns a __JobHandle__ it is automatically registered with the EntityManager and all the types including the information about if it is reading or writing.
所有作业和系统都声明它们读取或写入的ComponentTypes。 因此，当JobComponentSystem返回__JobHandle__时，它会自动注册到EntityManager以及所有类型，包括有关它是读还是写的信息。

Thus if a system writes to component A, and another system later on reads from component A, then the JobComponentSystem looks through the list of types it is reading from and thus passes you a dependency against the job from the first system.
因此，如果系统写入组件A，而另一个系统稍后从组件A读取，则JobComponentSystem会查看它正在读取的类型列表，从而从第一个系统传递对作业的依赖关系。

So JobComponentSystem simply chains jobs as dependencies where needed and thus causes no stalls on the main thread. But what happens if a non-job ComponentSystem accesses the same data? Because all access is declared, the ComponentSystem automatically __Completes__ all jobs running against ComponentTypes that the system uses before invoking __OnUpdate__.
因此，JobComponentSystem只需将作业链接到需要的依赖项，从而不会在主线程上产生停顿。 但是，如果非作业ComponentSystem访问相同的数据会发生什么？ 因为声明了所有访问，所以ComponentSystem会自动__Completes__在调用__OnUpdate__之前对系统使用的ComponentTypes运行的所有作业。

#### Dependency management is conservative & deterministic

Dependency management is conservative. ComponentSystem simply tracks all ComponentGroups ever used and stores which types are being written or read based on that. (So if you inject ComponentDataArrays or use __IJobProcessComponentData__ once but skip using it sometimes, then we will create dependencies against all ComponentGroups that have ever been used by that ComponentSystem.)
依赖管理是保守的。 ComponentSystem只跟踪所有使用过的ComponentGroup，并根据它们存储正在写入或读取的类型。 （因此，如果您注入ComponentDataArrays或使用__IJobProcessComponentData__一次但有时跳过使用它，那么我们将创建针对该ComponentSystem曾经使用过的所有ComponentGroup的依赖项。）

Also when scheduling multiple jobs in a single system, dependencies must be passed to all jobs even though different jobs may need less dependencies. If that proves to be a performance issue the best solution is to split a system into two.
此外，在单个系统中调度多个作业时，必须将依赖关系传递给所有作业，即使不同的作业可能需要较少的依赖关系。 如果这被证明是性能问题，那么最好的解决方案是将系统分成两部分。

The dependency management approach is conservative. It allows for deterministic and correct behaviour while providing a very simple API.
依赖管理方法是保守的。 它提供了确定性和正确的行为，同时提供了一个非常简单的API。

### Sync points

All structural changes have hard sync points. __CreateEntity__, __Instantiate__, __Destroy__, __AddComponent__, __RemoveComponent__, __SetSharedComponentData__ all have a hard sync point. Meaning all jobs scheduled through JobComponentSystem will be completed before creating the Entity, for example. This happens automatically. So for instance: calling __EntityManager.CreateEntity__ in the middle of the frame might result in a large stall waiting for all previously scheduled jobs in the World to complete.
所有结构更改都有硬同步点。 __CreateEntity __，__ Onstantiate __，__ Destroy __，__ AddComponent __，__ RemoveofComponent __，_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ @ 这意味着通过JobComponentSystem安排的所有作业将在创建实体之前完成。 这会自动发生。 因此，例如：在帧的中间调用__EntityManager.CreateEntity__可能会导致一个大的停顿等待世界上所有先前安排的作业完成。

See [EntityCommandBuffer](#entitycommandbuffer) for more on avoiding sync points when creating Entities during game play.

### Multiple Worlds

Every World has its own EntityManager and thus a separate set of Job handle dependency management. A hard sync point in one world will not affect the other world. As a result, for streaming and procedural generation scenarios, it is useful to create entities in one World and then move them to another world in one transaction at the beginning of the frame. 
每个世界都有自己的EntityManager，因此有一组独立的Job句柄依赖关系管理。 一个世界中的硬同步点不会影响另一个世界。 因此，对于流和程序生成场景，在一个世界中创建实体然后在帧的开始处在一个事务中将它们移动到另一个世界是有用的。

See [ExclusiveEntityTransaction](#exclusiveentitytransaction) for more on avoiding sync points for procedural generation & streaming scenarios.


## Shared ComponentData

IComponentData is appropriate for data that varies between Entities, such as storing a world position. __ISharedComponentData__ is useful when many Entities have something in common, for example in the boid demo we instantiate many Entities from the same Prefab and thus the __MeshInstanceRenderer__ between many boid Entities is exactly the same. 
IComponentData适用于实体之间不同的数据，例如存储世界位置。 当许多实体有共同点时，__ OtherComponentData__非常有用，例如在boid演示中，我们从同一个预制实例中实例化了许多实体，因此许多boid实体之间的__MeshInstanceRenderer__完全相同。

```cs
[System.Serializable]
public struct MeshInstanceRenderer : ISharedComponentData
{
    public Mesh                 mesh;
    public Material             material;

    public ShadowCastingMode    castShadows;
    public bool                 receiveShadows;
}
```

In the boid demo we never change the MeshInstanceRenderer component, but we do move all the Entities __TransformMatrix__ every frame.
在boid演示中，我们永远不会更改MeshInstanceRenderer组件，但我们确实每帧移动所有实体__TransformMatrix__。

The great thing about ISharedComponentData is that there is literally zero memory cost on a per Entity basis.
关于ISharedComponentData的好处是，每个实体的内存成本实际上是零。、

We use ISharedComponentData to group all entities using the same InstanceRenderer data together and then efficiently extract all matrices for rendering. The resulting code is simple & efficient because the data is laid out exactly as it is accessed.
我们使用ISharedComponentData将所有实体组合在一起使用相同的InstanceRenderer数据，然后有效地提取所有矩阵以进行渲染。 生成的代码简单而有效，因为数据的布局与访问时完全相同。

* [MeshInstanceRendererSystem.cs](../../ECSJobDemos/Packages/com.unity.entities/Unity.Rendering.Hybrid/MeshInstanceRendererSystem.cs)

### Some important notes about SharedComponentData:

* Entities with the same SharedComponentData are grouped together in the same chunks. The index to the SharedComponentData is stored once per chunk, not per Entity. As a result SharedComponentData have zero memory overhead on a per Entity basis. 
* Using ComponentGroup we can iterate over all Entities with the same type.
* Additionally we can use __ComponentGroup.SetFilter()__ to iterate specifically over Entities that have a specific SharedComponentData value. Due to the data layout this iteration has low overhead.
* Using __EntityManager.GetAllUniqueSharedComponents__ we can retrieve all unique SharedComponentData that is added to any alive Entities.
* SharedComponentData are automatically [reference counted](https://en.wikipedia.org/wiki/Reference_counting).
* SharedComponentData should change rarely. Changing a SharedComponentData involves using [memcpy](https://msdn.microsoft.com/en-us/library/aa246468(v=vs.60).aspx) to copy all ComponentData for that Entity into a different chunk.

*具有相同SharedComponentData的实体在同一块中组合在一起。 SharedComponentData的索引每个块存储一次，而不是每个Entity存储一次。因此，SharedComponentData在每个实体的基础上具有零内存开销。
*使用ComponentGroup，我们可以迭代所有具有相同类型的实体。
*此外，我们可以使用__ComponentGroup.SetFilter（）__专门针对具有特定SharedComponentData值的实体进行迭代。由于数据布局，此迭代具有较低的开销。
*使用__EntityManager.GetAllUniqueSharedComponents__，我们可以检索添加到任何活动实体的所有唯一SharedComponentData。
* SharedComponentData自动[引用计数]（https://en.wikipedia.org/wiki/Reference_counting）。
* SharedComponentData应该很少更改。更改SharedComponentData涉及使用[memcpy]（https://msdn.microsoft.com/en-us/library/aa246468(v = vs.60）.aspx）将该实体的所有ComponentData复制到另一个块中。、

## ComponentSystem

## JobComponentSystem

## Iterating entities

Iterating over all Entities that have a matching set of components, is at the center of the ECS architecture.
迭代具有匹配组件集的所有实体，是ECS体系结构的核心。

## Injection

Injection allows your system to declare its dependencies, while those dependencies are then automatically injected into the injected variables before OnCreateManager, OnDestroyManager and OnUpdate.
注入允许您的系统声明其依赖项，然后在OnCreateManager，OnDestroyManager和OnUpdate之前将这些依赖项自动注入到注入的变量中。

#### Component Group Injection

Component Group injection automatically creates a ComponentGroup based on the required Component types.

This lets you iterate over all the entities matching those required Component types.
Each index refers to the same Entity on all arrays.
这使您可以迭代匹配所需组件类型的所有实体。
每个索引都指向所有阵列上的相同实体。

```cs
class MySystem : ComponentSystem
{
    public struct Group
    {
        // ComponentDataArray lets us access IComponentData 
        [ReadOnly]
        public ComponentDataArray<Position> Position;
        
        // ComponentArray lets us access any of the existing class Component                
        public ComponentArray<Rigidbody> Rigidbodies;

        // Sometimes it is necessary to not only access the components
        // but also the Entity ID.
        public EntityArray Entities;

        // The GameObject Array lets us retrieve the game object.
        // It also constrains the group to only contain GameObject based entities.              
		// GameObject数组让我们检索游戏对象。
         //它还将组限制为仅包含基于GameObject的实体。		
        public GameObjectArray GameObjects;

        // Excludes entities that contain a MeshCollider from the group
        public SubtractiveComponent<MeshCollider> MeshColliders;
        
        // The Length can be injected for convenience as well 
        public int Length;
    }
    [Inject] private Group m_Group;


    protected override void OnUpdate()
    {
        // Iterate over all entities matching the declared ComponentGroup required types
        for (int i = 0; i != m_Group.Length; i++)
        {
            m_Group.Rigidbodies[i].position = m_Group.Position[i].Value;

            Entity entity = m_Group.Entities[i];
            GameObject go = m_Group.GameObjects[i];
        }
    }
}
```

#### ComponentDataFromEntity injection

ComponentDataFromEntity<> can also be injected, this lets you get / set the component data by entity from a job. 


```cs
class PositionSystem : JobComponentSystem
{
    [Inject] ComponentDataFromEntity<Position> m_Positions;
}
```

#### Injecting other systems

```cs
class PositionSystem : JobComponentSystem
{
    [Inject] OtherSystem m_SomeOtherSystem;
}
```

Lastly you can also inject a reference to another system. This will populate the reference to the other system for you.

## ComponentGroup

The ComponentGroup is foundation class on top of which all iteration methods are built (Injection, foreach, IJobProcessComponentData etc)

Essentially a ComponentGroup is constructed with a set of required components, subtractive components. 

The ComponentGroup lets you extract individual arrays. All these arrays are guaranteed to be in sync (same length and the index of each array refers to the same Entity).

Generally speaking GetComponentGroup is used rarely, since ComponentGroup Injection and IJobProcessComponetnData is simpler and more expressive.

However the ComponentGroup API can be used for more advanced use cases like filtering a Component Group based on specific SharedComponent values.

ComponentGroup是基础类，在其基础上构建所有迭代方法（Injection，foreach，IJobProcessComponentData等）

本质上，ComponentGroup由一组必需的组件，减法组件构成。

ComponentGroup允许您提取单个数组。 所有这些数组都保证同步（长度相同，每个数组的索引指的是同一个实体）。

一般来说，很少使用GetComponentGroup，因为ComponentGroup Injection和IJobProcessComponetnData更简单，更具表现力。

但是，ComponentGroup API可用于更高级的用例，例如根据特定的SharedComponent值过滤组件组。

```cs
struct SharedGrouping : ISharedComponentData
{
    public int Group;
}

class PositionToRigidbodySystem : ComponentSystem
{
    ComponentGroup m_Group;

    protected override void OnCreateManager(int capacity)
    {
        // GetComponentGroup should always be cached from OnCreateManager, never from OnUpdate
        // - ComponentGroup allocates GC memory
        // - Relatively expensive to create
        // - Component type dependencies of systems need to be declared during OnCreateManager,
        //   in order to allow automatic ordering of systems
        m_Group = GetComponentGroup(typeof(Position), typeof(Rigidbody), typeof(SharedGrouping));
    }

    protected override void OnUpdate()
    {
        // Only iterate over entities that have the SharedGrouping data set to 1
        // (This could for example be used as a form of gamecode LOD)
        m_Group.SetFilter(new SharedGrouping { Group = 1 });
        
        var positions = m_Group.GetComponentDataArray<Position>();
        var rigidbodies = m_Group.GetComponentArray<Rigidbody>();

        for (int i = 0; i != positions.Length; i++)
            rigidbodies[i].position = positions[i].Value;
            
        // NOTE: GetAllUniqueSharedComponentDatas can be used to find all unique shared components 
        //       that are added to entities. 
        // EntityManager.GetAllUniqueSharedComponentDatas(List<T> shared);
    }
}
```

## ComponentDataFromEntity

The Entity struct identifies an Entity. If you need to access component data on another Entity, the only stable way of referencing that component data is via the Entity ID. EntityManager provides a simple get & set component data API for it.
```cs
Entity myEntity = ...;
var position = EntityManager.GetComponentData<LocalPosition>(entity);
...
EntityManager.SetComponentData(entity, position);
```

However EntityManager can't be used on a C# job. __ComponentDataFromEntity__ gives you a simple API that can also be safely used in a job.
但是，EntityManager不能用于C＃作业。 __ComponentDataFromEntity__为您提供了一个简单的API，也可以安全地在作业中使用。

```cs
// ComponentDataFromEntity can be automatically injected
[Inject]
ComponentDataFromEntity<LocalPosition> m_LocalPositions;

Entity myEntity = ...;
var position = m_LocalPositions[myEntity];
```

## ExclusiveEntityTransaction

EntityTransaction is an API to create & destroy entities from a job. The purpose is to enable procedural generation scenarios where instantiation on big scale must happen on jobs. As the name implies it is exclusive to any other access to the EntityManager.

ExclusiveEntityTransaction should be used on manually created world that acts as a staging area to construct & setup entities.
After the job has completed you can end the EntityTransaction and use ```EntityManager.MoveEntitiesFrom(EntityManager srcEntities);``` to move the entities to an active world.

EntityTransaction是用于从作业创建和销毁实体的API。 目的是启用程序生成场景，其中必须在作业上进行大规模实例化。 顾名思义，它对EntityManager的任何其他访问都是独占的。

ExclusiveEntityTransaction应该用于手动创建的世界，作为构建和设置实体的临时区域。
作业完成后，您可以结束EntityTransaction并使用```EntityManager.MoveEntitiesFrom（EntityManager srcEntities）;```将实体移动到活动世界。

## EntityCommandBuffer

The command buffer class solves two important problems:

1. When you're in a job, you can't access the entity manager
2. When you access the entity manager (to say, create an entity) you invalidate all injected arrays and component groups

1.当您在工作时，您无法访问实体管理器
2.当您访问实体管理器（例如，创建实体）时，您将使所有注入的数组和组件组无效

The command buffer abstraction allows you to queue up changes to be performed (from either a job or from the main thread) so that they can take effect later on the main thread. There are two ways to use a command buffer:

1. `ComponentSystem` subclasses which update on the main thread have one available automatically called `PostUpdateCommands`. To use it, simply reference the attribute and queue up your changes. They will be automatically applied to the world immediately after you return from your system's `Update` function.

Here's an example from the two stick shooter sample:

```cs
PostUpdateCommands.CreateEntity(TwoStickBootstrap.BasicEnemyArchetype);
PostUpdateCommands.SetComponent(new Position2D { Value = spawnPosition });
PostUpdateCommands.SetComponent(new Heading2D { Value = new float2(0.0f, -1.0f) });
PostUpdateCommands.SetComponent(default(Enemy));
PostUpdateCommands.SetComponent(new Health { Value = TwoStickBootstrap.Settings.enemyInitialHealth });
PostUpdateCommands.SetComponent(new EnemyShootState { Cooldown = 0.5f });
PostUpdateCommands.SetComponent(new MoveSpeed { speed = TwoStickBootstrap.Settings.enemySpeed });
PostUpdateCommands.AddSharedComponent(TwoStickBootstrap.EnemyLook);
```

As you can see, the API is very similar to the entity manager API. In this mode, it is helpful to think of the automatic command buffer as a convenience that allows you to prevent array invalidation inside your system while still making changes to the world.

2. For jobs, you must request command buffers from a `Barrier` on the main thread, and pass them to jobs. When the barrier system updates, the command buffers will play back on the main thread in the order they were created. This extra step is required so that memory management can be centralized and determinism of the generated entities and components can be guaranteed.

Again let's look at the two stick shooter sample to see how this works in practice.

### Barrier

First, a barrier system is declared:

```cs
public class ShotSpawnBarrier : BarrierSystem
{}
```

There's no code in a barrier system, it just serves as a synchronization point.

Next, we inject this barrier into the system that will request command buffers from it:

```cs
[Inject] private ShotSpawnBarrier m_ShotSpawnBarrier;
```

Now we can access the barrier when we're scheduling jobs and ask for command
buffers from it via `CreateCommandBuffer()`:

```cs
return new SpawnEnemyShots
{
    // ...
    CommandBuffer = m_ShotSpawnBarrier.CreateCommandBuffer(),
    // ...
}.Schedule(inputDeps);
```

In the job, we can use the command buffer normally:

```cs
CommandBuffer.CreateEntity(ShotArchetype);
CommandBuffer.SetComponent(spawn);
```

When the barrier system updates, it will automatically play back the command buffers. It's worth noting that the barrier system will take a dependency on any jobs spawned by systems that access it (so that it can now that the command buffers have been filled in fully). If you see bubbles in the frame, it may make sense to try moving the barrier later in the frame, if your game logic allows for this.

### Using command buffers from parallel for-style jobs

To record entity command buffers from parallel for jobs, you must use the `EntityCommandBuffer.Concurrent` type.

## GameObjectEntity

ECS ships with the __GameObjectEntity__ component. It is a MonoBehaviour. In __OnEnable__, the GameObjectEntity component creates an Entity with all components on the GameObject. As a result the full GameObject and all its components are now iterable by ComponentSystems.

TODO: what do you mean by "the GameObjectEntity component creates an Entity with all components on the GameObject" in the sentence above? do you mean all required components that this GameObject should have in that particular case? or is there a pre-defined set of components that will always be added? It's a little unclear.

> Note: for the time being, you must add a GameObjectEntity component on each GameObject that you want to be visible / iterable from the ComponentSystem.

## SystemStateComponentData

The purpose of SystemStateComponentData is to allow you to track resources internal to a system and have the opportunity to appropriately create and destroy those resources as needed without relying on individual callbacks.

SystemStateComponentData and SystemStateSharedComponentData are exactly like ComponentData and SharedComponentData, respectively, except in one important respect:
1. SystemState components are not deleted when an entity is destroyed.

DestroyEntity is shorthand for
1. Find all Components which reference this particular Entity ID
2. Delete those Components
3. Recycle the Entity id for reuse.

However, if SystemState components are present, they are not removed. This gives a system the opportunity to cleanup any resources or state associated with an Entity ID. The Entity id will only be reused once all SystemState components have been removed.


SystemStateComponentData的目的是允许您跟踪系统内部的资源，并有机会根据需要适当地创建和销毁这些资源，而不依赖于单个回调。

SystemStateComponentData和SystemStateSharedComponentData分别与ComponentData和SharedComponentData完全相同，除了一个重要方面：
1.销毁实体时不会删除SystemState组件。

DestroyEntity是简写
1.查找引用此特定实体ID的所有组件
2.删除这些组件
3.回收实体ID以供重用。

但是，如果存在SystemState组件，则不会删除它们。 这使系统有机会清除与实体ID相关联的任何资源或状态。 只有在删除所有SystemState组件后，才会重用Entity id。
