---
layout: page
title: Serialization and Writing Custom Serializers
uid: serialization
---

# 序列化和编写自定义序列化程序

Orleans有一个高级和可扩展的序列化框架。orleans序列化在grain请求和响应消息以及grain持久状态对象中传递的数据类型。作为这个框架的一部分，orleans自动为这些数据类型生成序列化代码。除了为已可.NET序列化的类型生成更有效的序列化/反序列化之外，Orleans还尝试为不可.NET序列化的Grain接口中使用的类型生成序列化程序。该框架还包括一组用于常用类型(列表、字典、字符串、原语、数组等)的高效内置序列化程序。

Orleans的序列化程序有两个重要的特性，使它与许多其他第三方序列化框架不同：动态类型/任意多态性和对象标识。

1.  **动态类型与任意多态性**-orleans对可以在grain调用中传递的类型没有任何限制，并保持了实际数据类型的动态特性。例如，这意味着如果grain接口中的方法被声明为接受`IDictionary`但在运行时发送者通过`SortedDictionary`，接收器确实会`SortedDictionary`(尽管“static contract/grain”接口没有指定此行为)。

2.  **维护对象标识**-如果同一个对象在一个grain调用的参数中传递了多个类型，或者从参数中间接指向了多个类型，那么orleans将只序列化它一次。在接收端，orleans将正确地恢复所有引用，以便反序列化后指向同一对象的两个指针仍然指向同一对象。在如下场景中，对象标识是很重要的。假设actor a正在向actor b发送一个包含100个条目的字典，并且字典中的10个键指向a一侧的同一个对象obj。在不保留对象标识的情况下，b将收到一个包含100个条目的字典，其中10个键指向10个不同的obj克隆。在保留对象标识的情况下，B侧的字典与A侧的字典完全相同，10个键指向一个对象obj。

以上两个行为是由标准的.NET二进制序列化程序提供的，因此，支持这一标准以及Orleans常见的行为对我们来说也很重要。

# 生成的序列化程序

Orleans使用以下规则来决定要生成哪些序列化程序。规则是：

1)扫描所有引用Core Orleans库的程序集中的所有类型。

2)在这些程序集中：为直接在grain interfaces方法签名或状态类签名中引用的类型或任何标记为`[Serializable]`属性。

3)此外，grain接口或实现项目可以通过添加`[KnownType]`或`[KnownAssembly]`程序集级属性，指示代码生成器为程序集中的特定类型或所有符合条件的类型生成序列化程序。

# 序列化提供程序

Orleans支持使用提供程序模型与第三方序列化程序集成。这需要实现`IExternalSerializer`在此文档的自定义序列化部分中描述的类型。一些常见序列化程序的集成与Orleans一起维护，例如：

-   [Protocol Buffers](https://developers.google.com/protocol-buffers/)以下内容：`Orleans.Serialization.ProtobufSerializer`来自[Microsoft.Orleans.OrleansGoogleUtils](https://www.nuget.org/packages/Microsoft.Orleans.OrleansGoogleUtils/)Nuget包。
-   [Bond](https://github.com/microsoft/bond/)以下内容：`Orleans.Serialization.BondsSerializer`来自[Microsoft.Orleans.Serialization.bond](https://www.nuget.org/packages/Microsoft.Orleans.Serialization.Bond/)Nuget包。
-   [Newtonsoft.Json AKA Json.NET](http://www.newtonsoft.com/json)以下内容：`Orleans.Serialization.OrleansJsonSerializer`来自 Orleans 核心库。

自定义实现`IExternalSerializer`在下面的编写自定义序列化程序部分中进行了描述。

### 配置

确保序列化配置在所有客户端和silos上都是相同的，这一点很重要。如果配置不一致，则可能发生序列化错误。

序列化提供程序，它实现`IExternalSerializer`，可以使用`ClientConfiguration`和`GlobalConfiguration`的`SerializationProviders`属性进行配置：

```csharp
var cfg = new ClientConfiguration();
cfg.SerializationProviders.Add(typeof(FantasticSerializer).GetTypeInfo());
```

```csharp
var cfg = new GlobalConfiguration();
cfg.SerializationProviders.Add(typeof(FantasticSerializer).GetTypeInfo());
```

或者，可以在`<serializationproviders/>`属性`<信息>`以下内容：

```xml
<Messaging>
  <SerializationProviders>
    <Provider type="GreatCompany.FantasticSerializer, GreatCompany.SerializerAssembly"/>
  </SerializationProviders>
</Messaging>
```

在这两种情况下，都可以配置多个提供程序。集合是有序的，这意味着如果在只可以序列化`b`类型的提供程序前指定同时可以序列化`a`和`b`的提供程序，则不会使用后一个提供程序。

# 编写自定义序列化程序

除了自动序列化生成之外，应用程序代码还可以为其选择的类型提供自定义序列化。Orleans建议对大多数应用程序类型使用自动序列化生成，只有在少数情况下，当您认为可以通过手动编写序列化程序来提高性能时，才编写自定义序列化程序。本说明描述了如何这样做，并确定了一些可能有用的特定情况。

应用程序可以通过三种方式自定义序列化：

1.  向类型中添加序列化方法并用适当的属性标记它们(`CopierMethod`，`SerializerMethod`，`DeserializerMethod`)中。对于应用程序拥有的类型(即可以向其添加新方法的类型)，此方法更可取。

2.  实施`IExternalSerializer`并在配置期间注册它。此方法对于集成外部序列化库非常有用。

3.  编写一个单独的静态类，用`[Serializer(typeof(yourtype))]`其中包含3个序列化方法和与上面相同的属性。此方法对于应用程序不拥有的类型非常有用，例如，在应用程序无法控制的其他库中定义的类型。

以下各节详细介绍了每种方法。

## 介绍

Orleans序列化分为三个阶段：对象立即被深度复制以确保隔离；在连接之前；对象被序列化为消息字节流；当传递到目标激活时，对象从接收的字节流重新创建(反序列化)。可以在消息中发送的数据类型(即可以作为方法参数或返回值传递的类型)必须具有执行这三个步骤的关联例程。我们将这些例程统称为数据类型的序列化程序。

类型的复制器是独立的，而序列化器和反序列化器是一起工作的一对。您可以只提供一个自定义复制器，或者只提供一个自定义序列化器和一个自定义反序列化器，也可以提供这三个的自定义实现。

序列化程序在silos启动时以及加载程序集时为每个受支持的数据类型注册。对于要使用的类型，自定义序列化程序例程需要注册。序列化程序选择基于要复制或序列化的对象的动态类型。因此，不需要为抽象类或接口创建序列化程序，因为它们永远不会被使用。

## 何时考虑编写自定义序列化程序

手工编制的序列化程序例程很少会比生成的版本执行得更好。如果您想这样做，您应该首先考虑以下选项：

如果数据类型中有不需要序列化或复制的字段或属性，可以使用`非序列化`属性。这将导致生成的代码在复制和序列化时跳过这些字段。使用`不可变<t>` & `[不变]`尽可能避免复制不可变数据。关于*优化拷贝*详情见下文。如果要避免使用标准泛型集合类型，请不要使用。Orleans运行时包含泛型集合的自定义序列化程序，这些泛型集合使用集合的语义来优化复制、序列化和反序列化。这些集合在序列化字节流中还具有特殊的“缩写”表示形式，从而带来更大的性能优势。例如，一个`字典<string，string>`会比`list<tuple<string，string>>`是的。

自定义序列化程序可以提供显著性能提高的最常见情况是，数据类型中编码了重要的语义信息，而这些信息仅通过复制字段值是不可用的。例如，通过将数组视为索引/值对的集合，即使应用程序为了提高操作速度而将数据保持为完全实现的数组，填充较少的数组通常也可以更有效地序列化。

在编写自定义序列化程序之前，要做的一个关键事情是确保生成的序列化程序确实会损害您的性能。分析在这方面有点帮助，但更重要的是使用不同的序列化负载运行应用程序的端到端压力测试，以评估系统级别的影响，而不是序列化的微观影响。例如，构建一个不向grain方法传递参数或结果的测试版本，只需在两端使用固定值，就可以放大序列化和复制对系统性能的影响。

## 方法1:向类型添加序列化方法

所有序列化程序例程都应实现为它们所操作的类或结构的静态成员。这里显示的名称不是必需的；注册基于各个属性的存在，而不是方法名。请注意，序列化程序方法不必是公共的。

除非实现所有三个序列化例程，否则应使用`可串行化`属性，以便为您生成缺少的方法。

### 复制器

复制器方法用`Orleans.CopierMethod`属性标记：

```csharp
[CopierMethod]
static private object Copy(object input, ICopyContext context)
{
    ...
}
```

复制器通常是最简单的序列化程序编写。它们接受一个对象，该对象的类型保证与复制器定义的类型相同，并且必须返回该对象在语义上等效的副本。

如果作为复制对象的一部分，需要复制子对象，则最好的方法是使用serializationmanager的deepcopyinner例程：

```csharp
var fooCopy = SerializationManager.DeepCopyInner(foo, context);
```

为了维护完整复制操作的对象标识上下文，使用deepcopyinner而不是deepcopy非常重要。

#### 维护对象标识

复制例程的一个重要职责是维护对象标识。orleans运行时为此提供了一个helper类。在“手动”复制子对象(即，不是通过调用deepcopyiner)之前，请检查是否已按如下方式引用它：

```csharp
var fooCopy = context.CheckObjectWhileCopying(foo);

if (fooCopy == null)
{
    // Actually make a copy of foo
    context.RecordObject(foo, fooCopy);
}
```

最后一行，调用`RecordObject`是必需的，以便将来可以通过`CheckObjectWhileCopying`找到对同一个对象的引用。

请注意，这只应针对类实例，而不是结构实例或.NET原语(字符串、uri、枚举)。

如果你使用`深复制内部`若要复制子对象，则会为您处理对象标识。

### 序列化程序

序列化方法标记为`SerializerMethod`属性：

```csharp
[SerializerMethod]
static private void Serialize(object input, ISerializationContext context, Type expected)
{
    ...
}
```

与copiers一样，传递给序列化程序的“input”对象保证是定义类型的实例。可以忽略“预期”类型；它基于有关数据项的编译时类型信息，并在较高级别上用于在字节流中形成类型前缀。

若要序列化子对象，请使用`SerializationManager`的`SerializeInner`例行程序：

```csharp
SerializationManager.SerializeInner(foo, context, typeof(FooType));
```

如果foo没有特定的预期类型，那么可以为预期类型传递null。

这个`BinaryTokenStreamReader`类提供了多种将数据写入字节流的方法。类的实例可以通过`context.StreamReader`属性。有关文档，请参见类。

### 反序列化程序

反序列化方法标记为`DeserializerMethod`属性：

```csharp
[DeserializerMethod]
static private object Deserialize(Type expected, IDeserializationContext context)
{
    ...
}
```

可以忽略“预期”类型；它基于有关数据项的编译时类型信息，并在较高级别上用于在字节流中形成类型前缀。要创建的对象的实际类型将始终是定义反序列化程序的类的类型。

若要反序列化子对象，请使用`SerializationManager`的`DeserializeInner`例行程序：

```csharp
var foo = SerializationManager.DeserializeInner(typeof(FooType), context);
```

或者：

```csharp
var foo = SerializationManager.DeserializeInner<FooType>(context);
```

如果foo没有特定的预期类型，请使用非泛型`DeserializeInner`变型并通过`null`对于所需的类型。

这个`BinaryTokenStreamReader`类提供了从字节流读取数据的各种方法。类的实例可以通过`context.StreamReader`属性。有关文档，请参见类。

## 方法2:编写序列化程序提供程序

在这个方法中，您可以实现`Orleans.Serialization.IexternalSerializer`并将其添加到`SerializationProviders`两者的属性`ClientConfiguration`在客户和`GlobalConfiguration`在silos里。配置在上面的序列化提供程序部分有详细说明。

实施`IExternalSerializer`遵循以下为序列化方法描述的模式`方法1`上面添加了`Initialize`方法和`IsSupportedType`Orleans用来确定序列化程序是否支持给定类型的方法。这是接口定义：

```csharp
public interface IExternalSerializer
{
    /// <summary>
    /// Initializes the external serializer. Called once when the serialization manager creates 
    /// an instance of this type
    /// </summary>
    void Initialize(Logger logger);

    /// <summary>
    /// Informs the serialization manager whether this serializer supports the type for serialization.
    /// </summary>
    /// <param name="itemType">The type of the item to be serialized</param>
    /// <returns>A value indicating whether the item can be serialized.</returns>
    bool IsSupportedType(Type itemType);

    /// <summary>
    /// Tries to create a copy of source.
    /// </summary>
    /// <param name="source">The item to create a copy of</param>
    /// <param name="context">The context in which the object is being copied.</param>
    /// <returns>The copy</returns>
    object DeepCopy(object source, ICopyContext context);

    /// <summary>
    /// Tries to serialize an item.
    /// </summary>
    /// <param name="item">The instance of the object being serialized</param>
    /// <param name="context">The context in which the object is being serialized.</param>
    /// <param name="expectedType">The type that the deserializer will expect</param>
    void Serialize(object item, ISerializationContext context, Type expectedType);

    /// <summary>
    /// Tries to deserialize an item.
    /// </summary>
    /// <param name="context">The context in which the object is being deserialized.</param>
    /// <param name="expectedType">The type that should be deserialized</param>
    /// <returns>The deserialized object</returns>
    object Deserialize(Type expectedType, IDeserializationContext context);
}
```

## 方法3:为单个类型编写序列化程序

在这个方法中，您可以编写一个新的类，并用一个属性进行注释`[SerializerAttribute(typeof(TargetType))]`，其中`TargetType`正在序列化的类型，并实现3个序列化例程。如何编写这些例程的规则与方法1相同。Orleans使用`[SerializerAttribute(typeof(TargetType))]`确定该类是`TargetType`如果这个属性能够序列化多个类型，那么可以在同一个类上多次指定它。下面是这样一个类的示例：

```csharp
public class User
{
    public User BestFriend { get; set; }
    public string NickName { get; set; }
    public int FavoriteNumber { get; set; }
    public DateTimeOffset BirthDate { get; set; }
}

[Orleans.CodeGeneration.SerializerAttribute(typeof(User))]
internal class UserSerializer
{
    [CopierMethod]
    public static object DeepCopier(object original, ICopyContext context)
    {
        var input = (User) original;
        var result = new User();

        // Record 'result' as a copy of 'input'. Doing this immediately after construction allows for
        // data structures which have cyclic references or duplicate references.
        // For example, imagine that 'input.BestFriend' is set to 'input'. In that case, failing to record
        // the copy before trying to copy the 'BestFriend' field would result in infinite recursion.
        context.RecordCopy(original, result);

        // Deep-copy each of the fields.
        result.BestFriend = (User)context.SerializationManager.DeepCopy(input.BestFriend);
        result.NickName = input.NickName; // strings in .NET are immutable, so they can be shallow-copied.
        result.FavoriteNumber = input.FavoriteNumber; // ints are primitive value types, so they can be shallow-copied.
        result.BirthDate = (DateTimeOffset)context.SerializationManager.DeepCopy(input.BirthDate);
                
        return result;
    }

    [SerializerMethod]
    public static void Serializer(object untypedInput, ISerializationContext context, Type expected)
    {
        var input = (User) untypedInput;

        // Serialize each field.
        SerializationManager.SerializeInner(input.BestFriend, context);
        SerializationManager.SerializeInner(input.NickName, context);
        SerializationManager.SerializeInner(input.FavoriteNumber, context);
        SerializationManager.SerializeInner(input.BirthDate, context);
    }

    [DeserializerMethod]
    public static object Deserializer(Type expected, IDeserializationContext context)
    {
        var result = new User();

        // Record 'result' immediately after constructing it. As with with the deep copier, this
        // allows for cyclic references and de-duplication.
        context.RecordObject(result);

        // Deserialize each field in the order that they were serialized.
        result.BestFriend = SerializationManager.DeserializeInner<User>(context);
        result.NickName = SerializationManager.DeserializeInner<string>(context);
        result.FavoriteNumber = SerializationManager.DeserializeInner<int>(context);
        result.BirthDate = SerializationManager.DeserializeInner<DateTimeOffset>(context);

        return result;
    }
}
```

### 序列化泛型类型

这个`目标类型`参数`[SerializerAttribute(typeof(TargetType))]`可以是开放泛型类型，例如，`MyGenericType<>`是的。在这种情况下，序列化程序类必须具有与目标类型相同的泛型参数。Orleans将在运行时为每个具体对象创建序列化程序的具体版本`MyGenericType<T>`序列化的类型，例如，每个`MyGenericType<int>`和`MyGenericType<string>`是的。

## 编写序列化程序和反序列化程序的提示

通常，编写序列化器/反序列化器对的最简单方法是通过构造字节数组并将数组长度写入流，然后是数组本身，然后通过反转过程进行反序列化。如果数组是固定长度的，则可以从流中忽略它。如果数据类型可以简洁地表示，并且没有可能重复的子对象(因此不必担心对象标识)，那么这种方法很好地工作。

另一种方法，即Orleans运行时对字典等集合所采用的方法，适用于具有重要而复杂的内部结构的类：使用实例方法访问对象的语义内容，序列化该内容，并通过设置语义内容而不是复杂的内部内容反序列化。国家。在这种方法中，使用SerializeInner编写内部对象，使用DeserializeInner读取内部对象。在这种情况下，编写自定义复制器也很常见。

如果您编写一个自定义序列化程序，并且它最终看起来像类中每个字段的序列化内部调用序列，则不需要该类的自定义序列化程序。

# 回退序列化

Orleans支持在运行时传输任意类型，因此内置代码生成器无法确定将提前传输的整个类型集。此外，某些类型不能为其生成序列化程序，因为它们不可访问(例如，`私有的`)或者有不可访问的字段(例如，`只读`)中。因此，需要对意外或无法提前生成序列化程序的类型进行及时序列化。负责这些类型的序列化程序称为*回退序列化程序*是的。Orleans提供了两个回退序列化程序：

-   `Orleans.Serialization.BinaryFormatterSerializer`使用.net的[BinaryFormatter](https://msdn.microsoft.com/en-us/library/system.runtime.serialization.formatters.binary.binaryformatter)；和
-   `Orleans.Serialization.ILBasedSerializer`发出[CIL](https://en.wikipedia.org/wiki/Common_Intermediate_Language)在运行时创建序列化程序的指令，该序列化程序利用Orleans的序列化框架对每个字段进行序列化。这意味着如果一个不可访问的类型`MyPrivateType`包含字段`MyType`它有一个自定义序列化程序，该自定义序列化程序将用于序列化它。

可以使用`FallbackSerializationProvider`两者的属性`ClientConfiguration`在客户和`GlobalConfiguration`在silos里。

```csharp
var cfg = new ClientConfiguration();
cfg.FallbackSerializationProvider = typeof(FantasticSerializer).GetTypeInfo();
```

```csharp
var cfg = new GlobalConfiguration();
cfg.FallbackSerializationProvider = typeof(FantasticSerializer).GetTypeInfo();
```

或者，可以在XML配置中指定回退序列化提供程序：

```xml
<Messaging>
  <FallbackSerializationProvider type="GreatCompany.FantasticFallbackSerializer, GreatCompany.SerializerAssembly"/>
</Messaging>
```

`BinaryFormatterSerializer`是默认的回退序列化程序。

# 异常序列化

使用[回退序列化程序](serialization.md#fallback-serialization)是的。使用默认配置，`BinaryFormatterSerializer`是回退序列化程序，因此[ISerializable模式](https://docs.microsoft.com/en-us/dotnet/standard/serialization/custom-serialization)必须紧跟其后才能确保异常类型中所有属性的正确序列化。

以下是具有正确实现的序列化的异常类型的示例：

```csharp
[Serializable]
public class MyCustomException : Exception
{
    public string MyProperty { get; }

    public MyCustomException(string myProperty, string message)
        : base(message)
    {
        this.MyProperty = myProperty;
    }

    public MyCustomException(string transactionId, string message, Exception innerException)
        : base(message, innerException)
    {
        this.MyProperty = transactionId;
    }

    // Note: This is the constructor called by BinaryFormatter during deserialization
    public MyCustomException(SerializationInfo info, StreamingContext context)
        : base(info, context)
    {
        this.MyProperty = info.GetString(nameof(this.MyProperty));
    }

    // Note: This method is called by BinaryFormatter during serialization
    public override void GetObjectData(SerializationInfo info, StreamingContext context)
    {
        base.GetObjectData(info, context);
        info.AddValue(nameof(this.MyProperty), this.MyProperty);
    }
}
```

# 使用不可变类型优化复制

Orleans有一个特性，可以用来避免一些与序列化包含不可变类型的消息相关的开销。本节从上下文开始介绍特性及其应用程序。

## Orleans的序列化

当一个grain方法被调用时，orleans运行时会对方法参数进行一个深度拷贝，并从拷贝中形成请求。这可以防止调用代码在数据传递给被调用的Grain之前修改参数对象。

如果被调用的Grain在不同的silos上，那么拷贝最终被序列化为字节流，并通过网络发送到目标silos，在那里它们被反序列化为对象。如果被调用的grain位于同一个silos上，那么副本将直接传递给被调用的方法。

返回值的处理方式相同：首先复制，然后可能序列化和反序列化。

请注意，所有3个进程(复制、序列化和反序列化)都尊重对象标识。换言之，如果您传递一个列表，其中包含两个相同的对象，那么在接收端，您将得到一个列表，其中包含两个相同的对象，而不是包含两个具有相同值的对象。

## 优化拷贝

在许多情况下，深度复制是不必要的。例如，一个可能的场景是一个web前端，它从客户端接收一个字节数组，并将该请求(包括字节数组)传递到一个grain进行处理。前端进程一旦将数组传递给grain，就不会对它做任何事情；特别是，它不会重用数组来接收将来的请求。在Grains内部，字节数组被解析以获取输入数据，但未被修改。grain返回它创建的另一个字节数组，并将其传递回web客户端；一旦返回该数组，它将立即丢弃该数组。web前端将结果字节数组传递回其客户端，而无需修改。

在这种情况下，不需要复制请求或响应字节数组。不幸的是，orleans运行时无法自己解决这个问题，因为它无法判断数组是由web前端修改的还是由grain修改的。在最好的情况下，我们会有某种.NET机制来指示某个值不再被修改；如果没有这种机制，我们会为此添加特定于Orleans的机制：`Immutable<T>`包装类和`[Immutable]`属性。

### 使用`Immutable<t>`

这个`Orleans.Concurrency.Immutable<T>`wrapper类用于指示一个值可能被认为是不可变的；也就是说，底层值不会被修改，因此安全共享不需要复制。注意使用`不可变<t>`意味着无论是价值的提供者还是价值的接受者都不会在未来对其进行修改；它不是单方面的承诺，而是相互的双边承诺。

使用`Immutable<T>`很简单：在grain接口中，而不是传递`T`，通过`Immutable<T>`是的。例如，在上述场景中，grain方法是：

```csharp
Task<byte[]> ProcessRequest(byte[] request);
```

变成：

```csharp
Task<Immutable<byte[]>> ProcessRequest(Immutable<byte[]> request);
```

创建`Immutable<T>`，只需使用构造函数：

```csharp
Immutable<byte[]> immutable = new Immutable<byte[]>(buffer);
```

若要获取不可变的内部值，请使用`.Value`属性：

```csharp
byte[] buffer = immutable.Value;
```

### 使用`[Immutable]`

对于用户定义的类型，`[Orleans.Concurrency.Immutable]`属性可以添加到类型。这指示Orleans的序列化程序避免复制此类型的实例。下面的代码片段演示如何使用`[Immutable]`表示不可变类型。在传输过程中不会复制此类型。

```csharp
[Immutable]
public class MyImmutableType
{
    public MyImmutableType(int value)
    {
        this.MyValue = value;
    }

    public int MyValue { get; }
}
```

## Orleans的不变性

对于orleans来说，不变性是一个相当严格的声明：数据项的内容不会以任何可能改变其语义的方式进行修改，也不会干扰另一个线程同时访问该项。确保这一点的最安全方法是根本不修改项：按位不变，而不是逻辑不变。

在某些情况下，将其放宽到逻辑不变性是安全的，但是必须注意确保变异代码是正确的线程安全的；因为处理多线程是复杂的，并且在Orleans上下文中是不常见的，所以我们强烈建议不要使用这种方法，并建议坚持按位不变性。

# 序列化最佳实践

序列化在Orleans有两个主要目的：

1.  作为在运行时在Grains和客户端之间传输数据的有线格式。
2.  作为一种存储格式，用于保存长期数据以供以后检索。

Orleans产生的串行器由于其灵活性、性能和通用性而适合于第一目的。它们不太适合用于第二个目的，因为它们没有显式的版本容忍。建议用户配置一个版本容忍序列化程序，如[协议缓冲区](https://developers.google.com/protocol-buffers/)对于持久数据。协议缓冲区通过`Orleans.Serialization.ProtobufSerializer`从[Microsoft.Orleans.OrleansGoogleUtils](https://www.nuget.org/packages/Microsoft.Orleans.OrleansGoogleUtils/)Nuget包。应使用所选特定序列化程序的最佳实践，以确保版本公差。可以使用`序列化提供程序`如上所述的配置属性。
