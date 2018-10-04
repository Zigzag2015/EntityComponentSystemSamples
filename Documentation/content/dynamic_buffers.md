# Dynamic Buffers

A `DynamicBuffer` is a type of component data that allows a variable-sized, "stretchy"
buffer to be associated with an `Entity`. It behaves as a component type that
carries an internal capacity of a certain number of elements, but can allocate
a heap memory block if the internal capacity is exhausted.

Memory management is fully automatic when using this approach. Memory associated with
`DynamicBuffers` is managed by the `EntityManager` so that when a `DynamicBuffer`
component is removed, any associated heap memory is automatically freed as well.

`DynamicBuffers` supersede fixed array support which has been removed.

`DynamicBuffer`是一种组件数据，允许可变大小的“弹性”
缓冲区与“实体”相关联。 它表现为组件类型
具有一定数量元素的内部容量，但可以分配
如果内部容量耗尽，则为堆内存块。

使用此方法时，内存管理是完全自动的。 与...相关的记忆
`DynamicBuffers`由`EntityManager`管理，以便当`DynamicBuffer`时
组件被删除，任何相关的堆内存也会自动释放。

`DynamicBuffers`取代已删除的固定阵列支持。

## Declaring Buffer Element Types

To declare a `Buffer`, you declare it with the type of element that you will be
putting into the `Buffer`:
要声明一个`Buffer`，你可以使用你将要的元素类型声明它
放入`Buffer`：

    // This describes the number of buffer elements that should be reserved
    // in chunk data for each instance of a buffer. In this case, 8 integers
    // will be reserved (32 bytes) along with the size of the buffer header
    // (currently 16 bytes on 64-bit targets)
    [InternalBufferCapacity(8)]
    public struct MyBufferElement : IBufferElementData
    {
        // These implicit conversions are optional, but can help reduce typing.
        public static implicit operator int(MyBufferElement e) { return e.Value; }
        public static implicit operator MyBufferElement(int e) { return new MyBufferElement { Value = e }; }
        
        // Actual value each buffer element will store.
        public int Value;
    }

While it seem strange to describe the element type and not the `Buffer` itself,
this design enables two key benefits in the ECS: 

1. It supports having more than one `DynamicBuffer` of type `float3`, or any
   other common value type. You can add any number of `Buffers` that leverage the
   same value types, as long as the elements are uniquely wrapped in a top-level
   struct.

2. We can include `Buffer` element types in `EntityArchetypes`, and it generally
   will behave like having a component.
   

虽然描述元素类型而不是“缓冲区”本身似乎很奇怪，
这种设计在ECS中实现了两个主要优势：

它支持多个`float3`类型的`DynamicBuffer`，或者任何
    其他常见的价值类型。 你可以添加任意数量的利用它的“缓冲区”
    相同的值类型，只要元素唯一地包装在顶级
   结构。

2.我们可以在`EntityArchetypes`中包含`Buffer`元素类型
    将表现得像一个组件。

## Adding Buffer Types To Entities

To add a buffer to an `Entity`, you can use the normal methods of adding a
component type onto an `Entity`:

要向`Entity`添加缓冲区，可以使用添加a的常规方法
组件类型到`Entity`：

### Using AddBuffer()

    entityManager.AddBuffer<MyBufferElement>(entity);

### Using an archetype

    Entity e = entityManager.CreateEntity(typeof(MyBufferElement));

## Accessing Buffers

There are several ways to access `DynamicBuffers`, which parallel access methods
to regular component data.

### Direct, main-thread only access 

    DynamicBuffer<MyBufferElement> buffer = entityManager.GetBuffer<MyBufferElement>(entity);

### Injection based access

Similar to `ComponentDataArray` you can inject a `BufferArray` which provides
the `DynamicBuffers` in a parallel array to the other injections. This example
provides a system that appends a value to every `Buffer` in an injected set:

与“ComponentDataArray”类似，您可以注入一个提供的“BufferArray”
`DynamicBuffers`在与其他注入的并行数组中。 这个例子
提供了一个系统，它为注入集合中的每个“缓冲区”附加一个值：

    public class InjectionDemo : JobComponentSystem
    {
        public struct Data
        {
            public readonly int Length;
            public BufferArray<EcsIntElement> Buffers;
        }
    
        [Inject] Data m_Data;
    
        public struct MyJob : IJobParallelFor
        {
            public BufferArray<EcsIntElement> Buffers;
    
            public void Execute(int i)
            {
                Buffers[i].Append(i * 3);
            }
        }
    
        protected override JobHandle OnUpdate(JobHandle inputDeps)
        {
            return new MyJob { Buffers = m_Data.Buffers }.Schedule(m_Data.Length, 32, inputDeps);
        }
    }

## Entity based access

You can also look up `Buffers` on a per-`Entity` basis:

        var lookup = GetBufferArrayFromEntity<EcsIntElement>();
        var buffer = lookup[myEntity];
        buffer.Append(17);
        buffer.RemoveAt(0);

## Entity based injection access

Similarly to injecting `ComponentDataFromEntity` you can inject
`BufferDataFromEntity` and look up `Buffers` indirectly. 

与注入`ComponentDataFromEntity`类似，您可以注入
`BufferDataFromEntity`并间接查找`Buffers`。

## Reinterpreting Buffers (experimental)

`Buffers` can be reinterpreted as a type of the same size. The intention is to
allow controlled type-punning and to get rid of the wrapper element types when
they get in the way. To reinterpret, simply call `Reinterpret<T>`:

“缓冲区”可以重新解释为相同大小的类型。 目的是为了
允许受控类型 - 双关语，并在何时删除包装元素类型
他们挡路了。 要重新解释，只需调用`Reinterpret <T>`：

    var intBuffer = entityManager.GetBuffer<EcsIntElement>().Reinterpret<int>();

The reinterpreted `Buffer` carries with it the safety handle of the original
`Buffer`, and is safe to use. They use the same underlying `BufferHeader`, so
modifications to one reinterpreted `Buffer` will be immediately reflected in
others.

Note that there are no type checks involved, so it is entirely possible to
alias a `uint` and `float` buffer.

重新解释的“缓冲区”带有原件的安全手柄
`Buffer`，使用安全。 他们使用相同的底层`BufferHeader`，所以
对一个重新解释的“缓冲区”的修改将立即反映出来
其他。

请注意，不涉及类型检查，因此完全可以
别名是`uint`和`float`缓冲区。
