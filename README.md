Utf8Json - Fast JSON Serializer for C#
===
 [![Releases](https://img.shields.io/github/release/neuecc/Utf8Json.svg)](https://github.com/neuecc/Utf8Json/releases)

Definitely Fastest and Zero Allocation JSON Serializer for C#(.NET, .NET Core, Unity and Xamarin), this serializer write/read directly to UTF8 binary so boostup performance. And I adopt the same architecture as the fastest binary serializer, [MessagePack for C#](https://github.com/neuecc/MessagePack-CSharp) that I've developed.

![image](https://user-images.githubusercontent.com/46207/30883721-11e0526e-a348-11e7-86f8-efff85a9afe0.png)

Utf8Json does not beat MessagePack for C#(binary), but shows a similar memory consumption(there is no additional memory allocation) and achieves higher performance than other JSON serializers.

The crucial difference is that read and write directly to UTF8 binaries means that there is no overhead. Normaly serialization requires serialize to `Stream` or `byte[]`, it requires additional UTF8.GetBytes cost or StreamReader/Writer overhead(it is very slow!).

```csharp
TargetClass obj1;

// Object to UTF8 byte[]
[Benchmark]
public byte[] Utf8JsonSerializer()
{
    return Utf8Json.JsonSerializer.Serialize(obj1, jsonresolver);
}

// Object to String to UTF8 byte[]
[Benchmark]
public byte[] Jil()
{
    return utf8.GetBytes(global::Jil.JSON.Serialize(obj1));
}

// Object to Stream with StreamWriter
[Benchmark]
public void JilTextWriter()
{
    using (var ms = new MemoryStream())
    using (var sw = new StreamWriter(ms, utf8))
    {
        global::Jil.JSON.Serialize(obj1, sw);
    }
}
```

For example, the `OutputFormatter` of [ASP.NET Core](https://github.com/aspnet/Home) needs to write to Body(`Stream`), but using [Jil](https://github.com/kevin-montrose/Jil)'s TextWriter overload is slow. (This not means Jil is slow, for example `StreamWriter` allocate many memory(`char[1024]` and `byte[3075]`) on constructor ([streamwriter.cs#L203-L204](https://github.com/Microsoft/referencesource/blob/master/mscorlib/system/io/streamwriter.cs#L203-L204)) and other slow features unfortunately).

```csharp
// ASP.NET Core, OutputFormatter
public class JsonOutputFormatter : IOutputFormatter //, IApiResponseTypeMetadataProvider
{
    const string ContentType = "application/json";
    static readonly string[] SupportedContentTypes = new[] { ContentType };

    public Task WriteAsync(OutputFormatterWriteContext context)
    {
        context.HttpContext.Response.ContentType = ContentType;

        // Jil, normaly JSON Serializer requires serialize to Stream or byte[].
        using (var writer = new StreamWriter(context.HttpContext.Response.Body))
        {
            Jil.JSON.Serialize(context.Object, writer, _options);
            writer.Flush();
            return Task.CompletedTask;
        }

        // Utf8Json
        // Utf8Json.JsonSerializer.NonGeneric.Serialize(context.ObjectType, context.HttpContext.Response.Body, context.Object, resolver);
    }
}
```

The approach of directly write/read from JSON binary is similar to [corefxlab/System.Text.Json](https://github.com/dotnet/corefxlab/tree/master/src/System.Text.Json/System/Text/Json) and [corefxlab/System.Text.Formatting](https://github.com/dotnet/corefxlab/wiki/System.Text.Formatting). But it is not yet finished and not be general serializer.

> Benchmark code is in [sandbox/PerfBenchmark](https://github.com/neuecc/Utf8Json/tree/master/sandbox/PerfBenchmark) by [BenchmarkDotNet](https://github.com/dotnet/BenchmarkDotNet).

Install and QuickStart
---
The library provides in NuGet except for Unity. Standard library availables for .NET Framework 4.5 and .NET Standard 2.0.

```
Install-Package Utf8Json
```

And official Extension Packages for support other library(ImmutableCollection) or binding for framework(ASP.NET Core MVC).

```
Install-Package Utf8Json.ImmutableCollection
Install-Package Utf8Json.UnityShims
Install-Package Utf8Json.AspNetCoreMvcFormatter
```

NuGet page links - [Utf8Json](https://www.nuget.org/packages/Utf8Json), [Utf8Json.ImmutableCollection](https://www.nuget.org/packages/Utf8Json.ImmutableCollection), [Utf8Json.UnityShims](https://www.nuget.org/packages/Utf8Json.UnityShims), [Utf8Json.AspNetCoreMvcFormatter](https://www.nuget.org/packages/Utf8Json.AspNetCoreMvcFormatter)

for Unity, you can download from [releases](https://github.com/neuecc/Utf8Json/releases) page. There providing .unitypackage. Unity support details, see [Unity section](https://github.com/neuecc/Utf8Json#for-unity).

QuickStart, you can call `Utf8Json.JsonSerializer`.`Serialize/Deserialize`.

```csharp
var p = new Person { Age = 99, Name = "foobar" };

// Object -> byte[] (UTF8)
byte[] result = JsonSerializer.Serialize(p);

// byte[] -> Object
var p2 = JsonSerializer.Deserialize<Person>(result);

// Object -> String
var json = JsonSerializer.ToJsonString(p2);

// Write to Stream
JsonSerializer.Serialize(stream, p2);
```

In default, you can serialize all public members. You can customize serialize to private, exclude null, change DateTime format(default is ISO8601), enum handling, etc. see the [Resolver](https://github.com/neuecc/Utf8Json#resolver) section.

Performance of Serialize
---
This image is what code is generated when object serializing.

![image](https://user-images.githubusercontent.com/46207/30877807-c7f264d8-a335-11e7-91d8-ad1029d4ae86.png)

```csharp
// Disassemble generated serializer code.
public sealed class PersonFormatter : IJsonFormatter<Person>
{
    private readonly byte[][] stringByteKeys;
    
    public PersonFormatter()
    {
        // pre-encoded escaped string byte with "{", ":" and ",".
        this.stringByteKeys = new byte[][]
        {
            JsonWriter.GetEncodedPropertyNameWithBeginObject("Age"), // {\"Age\":
            JsonWriter.GetEncodedPropertyNameWithPrefixValueSeparator("Name") // ,\"Name\":
        };
    }

    public sealed Serialize(ref JsonWriter writer, Person person, IJsonFormatterResolver jsonFormatterResolver)
    {
        if (person == null) { writer.WriteNull(); return; }

        // WriteRawX is optimize byte->byte copy when we know src size.
        UnsafeMemory64.WriteRaw7(ref writer, this.stringByteKeys[0]);
        writer.WriteInt32(person.Age); // itoa write directly to avoid ToString + UTF8 encode
        UnsafeMemory64.WriteRaw8(ref writer, this.stringByteKeys[1]);
        writer.WriteString(person.Name);

        writer.WriteEndObject();
    }

    // public unsafe Person Deserialize(ref JsonReader reader, IJsonFormatterResolver jsonFormatterResolver)
}
```

Object to JSON's main serialization cost is write property name. Utf8Json create cache at first and after that only do memory copy. Optimize part1, concatenate "{", ":" and "." to cached  propertyname. Optimize part2, use optimized custom memory copy method(see: [UnsafeMemory.cs](https://github.com/neuecc/Utf8Json/blob/f724c83986d7c919a336c63e55f5a5886cca3575/src/Utf8Json/Internal/UnsafeMemory.cs)). Normally memory copy is used `Buffer.BlockCopy` but it has some overhead when target binary is small enough, releated to [dotnet/coreclr - issue #9786 Optimize Buffer.MemoryCopy](https://github.com/dotnet/coreclr/pull/9786) and [dotnet/coreclr - Add a fast path for byte[] to Buffer.BlockCopy #3118](https://github.com/dotnet/coreclr/pull/3118). Utf8Json don't use `Buffer.BlockCopy` and generates length specialized copy code that can reduce branch cost.

Number conversion is often high cost. If target encoding is UTF8 only, we can use `itoa` algorithm so avoid `int.ToString` and UTF8 encode cost. Especialy double-conversion, Utf8Json ported [ google/double-conversion](https://github.com/google/double-conversion) algorithm, it is fast `dtoa` and `atod` works.

Other optimize techniques.

* High-level API uses internal memory pool, don't allocate working memory under 64K
* Struct JsonWriter does not allocate any more and write underlying byte[] directly, don't use TextWriter
* Avoid boxing all codes, all platforms(include Unity/IL2CPP)
* Heavyly tuned dynamic IL code generation, it generates per option so reduce option check: see:[DynamicObjectResolver.cs](https://github.com/neuecc/Utf8Json/blob/f724c83986d7c919a336c63e55f5a5886cca3575/src/Utf8Json/Resolvers/DynamicObjectResolver.cs#L729-L1187)
* Call Primitive API directly when IL code generation knows target is primitive
* Getting cached generated formatter on static generic field(don't use dictionary-cache because lookup is overhead)
* Don't use `IEnumerable<T>` abstraction on iterate collection, specialized each collection types, see:[CollectionFormatter.cs](https://github.com/neuecc/Utf8Json/blob/f724c83986d7c919a336c63e55f5a5886cca3575/src/Utf8Json/Formatters/CollectionFormatters.cs)
* Uses optimized type key dictionary for non-generic methods, see: [ThreadsafeTypeKeyHashTable.cs](https://github.com/neuecc/Utf8Json/blob/f724c83986d7c919a336c63e55f5a5886cca3575/src/Utf8Json/Internal/ThreadsafeTypeKeyHashTable.cs)

Performance of Deserialize
---
When deserializing, requires property name to target member name matching. Utf8Json avoid string key decode for matching, generate [automata](https://en.wikipedia.org/wiki/Automata_theory) based IL inlining code.

![image](https://user-images.githubusercontent.com/46207/29754771-216b40e2-8bc7-11e7-8310-1c3602e80a08.png)

use raw byte[] slice and try to match each `ulong type` (per 8 character, if it is not enough, pad with 0).

```csharp
// Disassemble generated serializer code.
public sealed class PersonFormatter : IJsonFormatter<Person>
{
    // public sealed Serialize(ref JsonWriter writer, Person person, IJsonFormatterResolver jsonFormatterResolver)

    public unsafe Person Deserialize(ref JsonReader reader, IJsonFormatterResolver jsonFormatterResolver)
    {
        if (reader.ReadIsNull()) return null;
        
        reader.ReadIsBeginObjectWithVerify(); // "{"
        
        byte[] bufferUnsafe = reader.GetBufferUnsafe();
        int age;
        string name;
        fixed (byte* ptr2 = &bufferUnsafe[0])
        {
            int num;
            while (!reader.ReadIsEndObjectWithSkipValueSeparator(ref num)) // "}" or ","
            {
                // don't decode string, get raw slice
                ArraySegment<byte> arraySegment = reader.ReadPropertyNameSegmentUnescaped();
                byte* ptr3 = ptr2 + arraySegment.Offset;
                int count = arraySegment.Count;
                if (count != 0)
                {
                    // match automata per long
                    ulong key = AutomataKeyGen.GetKey(ref ptr3, ref count);
                    if (count == 0)
                    {
                        if (key == 6645569uL)
                        {
                            age = reader.ReadInt32(); // atoi read directly to avoid GetString + int.Parse
                            continue;
                        }
                        if (key == 1701667150uL)
                        {
                            name = reader.ReadString();
                            continue;
                        }
                    }
                }
                reader.ReadNextBlock();
            }
        }

        return new PersonSample
        {
            Age = age,
            Name = name
        };
    }
}
```

Of course number conversion(decode to string -> try parse) is high cost. Utf8Json directly convert byte[] to number by atoi/atod algorithm.

Built-in support types
---
These types can serialize by default.

Primitives(`int`, `string`, etc...), `Enum`, `Nullable<>`,  `TimeSpan`,  `DateTime`, `DateTimeOffset`, `Guid`, `Uri`, `Version`, `StringBuilder`, `BitArray`, `Type`, `ArraySegment<>`, `BigInteger`, `Complext`, `Task`, `Array[]`, `Array[,]`, `Array[,,]`, `Array[,,,]`, `KeyValuePair<,>`, `Tuple<,...>`, `ValueTuple<,...>`, `List<>`, `LinkedList<>`, `Queue<>`, `Stack<>`, `HashSet<>`, `ReadOnlyCollection<>`, `IList<>`, `ICollection<>`, `IEnumerable<>`, `Dictionary<,>`, `IDictionary<,>`, `SortedDictionary<,>`, `SortedList<,>`, `ILookup<,>`, `IGrouping<,>`, `ObservableCollection<>`, `ReadOnlyOnservableCollection<>`, `IReadOnlyList<>`, `IReadOnlyCollection<>`, `ISet<>`, `ConcurrentBag<>`, `ConcurrentQueue<>`, `ConcurrentStack<>`, `ReadOnlyDictionary<,>`, `IReadOnlyDictionary<,>`, `ConcurrentDictionary<,>`, `Lazy<>`, `Task<>`, custom inherited `ICollection<>` or `IDictionary<,>` with paramterless constructor, `IEnumerable`, `ICollection`, `IList`, `IDictionary` and custom inherited `ICollection` or `IDictionary` with paramterless constructor(includes `ArrayList` and `Hashtable`) and your own class or struct(includes anonymous type).

Utf8Json has sufficient extensiblity. You can add custom type support and has some official/third-party extension package. for ImmutableCollections(`ImmutableList<>`, etc). Please see [extensions section](https://github.com/neuecc/Utf8Json#extensions).

Object Serialization
---
Utf8Json can serialze your own public `Class` or `Struct`. In default, serializer search all public instance member(field or property) and uses there member name as json property name. If you want to avoid serialization target, you can use `[IgnoreDataMember]` attribute of `System.Runtime.Serialization` to target member. If you want to change property name, you can use `[DataMember(Name = string)]` attribute of `System.Runtime.Serialization`.

Utf8Json has other option, allows private/internal member serialization, convert property name to camelCalse/snake_case, if value is null does not create property. Or you can use a different DateTime format(default is ISO8601). The details, please read [Resolver](https://github.com/neuecc/Utf8Json#resolver) section. Here is sample.

```csharp
// default serializer change to allow private/exclude null/snake_case serializer.
JsonSerializer.SetDefaultResolver(StandardResolver.AllowPrivateExcludeNullSnakeCase);

var json = JsonSerializer.ToJsonString(new Person { Age = 23, FirstName = null, LastName = "Foo" });

// {"age":23,"last_name":"Foo"}
Console.WriteLine(json);
```

Serialize ImmutableObject(SerializationConstructor)
---
Utf8Json can deserialize immutable object like this.

```
public struct CustomPoint
{
    public readonly int X;
    public readonly int Y;

    public CustomPoint(int x, int y)
    {
        this.X = x;
        this.Y = y;
    }
}
```

Utf8Json choose constructor with the most matched argument by name(ignore case).

> MessagePack for C# choose `least` matched argument, please be aware of the opposite. This is design miss of MessagePack for C#.

If can not match automatically, you can specify to use constructor manually by `[SerializationConstructorAttribute]`.

```csharp
public class CustomPoint
{
    public readonly int X;
    public readonly int Y;

    public CustomPoint(int x, int y)
    {
        this.X = x;
        this.Y = y;
    }

    // used this constructor.
    [SerializationConstructor]
    public CustomPoint(int x)
    {
        this.X = x;
    }
}
```

Dynamic Deserialization
---
If use JsonSerializer.Deserialize<object> or JsonSerializer.Deserialize<dynamic>, convert json to `bool`, `double`, `string`, `IDictionary<string, object>`, `List<object>`.

```csharp
// dynamic json deserialize
var json = JsonSerializer.Deserialize<dynamic>(@"{""foo"":""json"",""bar"":100,""nest"":{""foobar"":true}}");

var r1 = json["foo"]; // "json" - dynamic(string)
var r2 = json["bar"]; // 100 - dynamic(double), it can cast to int or other number.
var r3 = json["nest"]["foobar"]; // true
```

If target is object, you access by string indexer.

Which serializer should be used
---
The performance of binary(protobuf, msgpack, avro, etc...) vs text(json, xml, yaml, etc...) depends on the implementation. However, binary has advantage basically. Utf8Json write directly to `byte[]` it is close to the binary serializer. But especialy `double` is still slower than binary write(Utf8Json uses [google/double-conversion](https://github.com/google/double-conversion/) algorithm, it is good but there are many processes, it can not be the fastest), write `string` requires escape and large payload must pay copy cost.

I recommend use [MessagePack for C#](https://github.com/neuecc/MessagePack-CSharp/) for general use serializer, C# to C#, C# to NoSQL, Save to File, communicate internal cross platform(multi-language), etc. MessagePack for C# has many options(Union ,Typeless, Compression) and definitely fastest.

But JSON is still better on web, for public Web API, send for JavaScript and  easy to integrate between multi-language communication. For example, use Utf8Json for Web API formatter and use MessagePack for C# for Redis. It is perfect.

For that reason Utf8Json is focusing performance and cross-platform compatibility. I don't implement original format(like Union, Typeless, Cyclic-Reference) if you want to use it should be use binary serializer. But customizability for serialize/deserialize JSON is important for cross-platform communication. IJsonFormatterResolver can serialize/deserialize all patterns and you can create own pattern.

High-Level API(JsonSerializer)
---
`JsonSerializer` is the entry point of Utf8Json. Its static methods are main API of Utf8Json.

| API | Description |
| --- | --- |
| `DefaultResolver` | FormatterResolver that used resolver less overloads. If does not set it, used StandardResolver.Default. |
| `SetDefaultResolver` | Set default resolver of JsonSerializer APIs. |
| `Serialize<T>` | Convert object to byte[] or write to stream. There has IJsonFormatterResolver overload, used specified resolver. |
| `SerializeUnsafe<T>` | Same as `Serialize<T>` but return `ArraySegement<byte>`. The result of ArraySegment is contains internal buffer pool, it can not share across thread and can not hold, so use quickly. |
| `ToJsonString<T>` | Convert object to string. |
| `Deserialize<T>` | Convert byte[] or `ArraySegment<byte>` or stream to object. There has IFormatterResolver overload, used specified resolver. |
| `NonGeneric.*` | NonGeneric APIs of Serialize/Deserialize. There accept type parameter at first argument. This API is bit slower than generic API but useful for framework integration such as ASP.NET formatter. |

Utf8Json operates at the byte[] level, so `Deserialize<T>(Stream)` read to end at first, it is not truly streaming deserialize.

High-Level API uses memory pool internaly to avoid unnecessary memory allocation. If result size is under 64K, allocates GC memory only for the return bytes.

Low-Level API(IJsonFormatter)
---
IJsonFormatter is serializer by each type. For example `Int32Formatter : IJsonFormatter<Int32>` represents Int32 JSON serializer.

```csharp
public interface IJsonFormatter<T> : IJsonFormatter
{
    void Serialize(ref JsonWriter writer, T value, IJsonFormatterResolver formatterResolver);
    T Deserialize(ref JsonReader reader, IJsonFormatterResolver formatterResolver);
}
```

Many builtin formatters exists under Utf8Json.Formatters. You can get sub type serializer by `formatterResolver.GetFormatter<T>`. Here is sample of write own formatter.

```csharp
// serialize fileinfo as string fullpath.
public class FileInfoFormatter<T> : IJsonFormatter<FileInfo>
{
    public void Serialize(ref JsonWriter writer, FileInfo value, IJsonFormatterResolver formatterResolver)
    {
        if (value == null) { writer.WriteNull(); return; }

        // if target type is primitive, you can also use writer.Write***.
        formatterResolver.GetFormatterWithVerify<string>().Serialize(ref writer, value.FullName, formatterResolver);
    }

    public FileInfo Deserialize(ref JsonReader reader, IJsonFormatterResolver formatterResolver)
    {
        if (reader.ReadIsNull()) return null;

        // if target type is primitive, you can also use reader.Read***.
        var path = formatterResolver.GetFormatterWithVerify<string>().Deserialize(ref reader, formatterResolver);
        return new FileInfo(path);
    }
}
```

Created formatter needs to register to IFormatterResolver. Please see [Resolver](https://github.com/neuecc/Utf8Json#resolver) section.

You can see many other samples from [builtin formatters](https://github.com/neuecc/Utf8Json/tree/master/src/Utf8Json/Formatters).

> If target type requires support dictionary key, you need to implements `IObjectPropertyNameFormatter<T>`, too.

Primitive API(JsonReader/JsonWriter)
---
`JsonReader` and `JsonWriter` is most low-level API. It is mutable struct so it must pass by ref. C# 7.2 supports ref-like types(see: [csharp-7.2/span-safety.md](https://github.com/dotnet/csharplang/blob/master/proposals/csharp-7.2/span-safety.md)) but not yet implements in C#, be careful to use.

`JsonReader` and `JsonWriter` is too primitive(performance reason), slightly odd. Internal state manages only int offset. You should manage other state(in array, in object...) manualy in outer.

**JsonReader**

| Method | Description |
| --- | --- |
| AdvanceOffset | Advance offset manually. |
| SkipWhiteSpace | Skip whitespace. |
| ReadNext | Skip JSON token. |
| ReadNextBlock | Skip JSON token with sub structures(array/object). This is useful for create deserializer. |
| ReadIsNull | If is null return true. |
| ReadIsBeginArray | If is '[' return true. |
| ReadIsBeginArrayWithVerify | If is '[' return true. |
| ReadIsEndArray | If is ']' return true. |
| ReadIsEndArrayWithVerify | If is not ']' throws exception. |
| ReadIsEndArrayWithSkipValueSeparator | check reached ']' or advance ',' when (ref int count) is not zero. |
| ReadIsBeginObject | If is '{' return true. |
| ReadIsBeginObjectWithVerify | If is not '{' throws exception. |
| ReadIsEndObject | If is '}' return true. |
| ReadIsEndObjectWithVerify | If is not '}' throws exception. |
| ReadIsEndObjectWithSkipValueSeparator | check reached '}' or advance ',' when (ref int count) is not zero. |
| ReadIsValueSeparator |  If is ',' return true. |
| ReadIsValueSeparatorWithVerify | If is not ',' throws exception. |
| ReadIsNameSeparator |  If is ':' return true. |
| ReadIsNameSeparatorWithVerify | If is not ':' throws exception. |
| ReadString | ReadString, unescaped. |
| ReadStringSegmentUnsafe | ReadString block but does not decode string. Return buffer is in internal buffer pool, be careful to use.  |
| ReadPropertyName | ReadString + ReadIsNameSeparatorWithVerify. |
| ReadPropertyNameSegmentRaw | Get raw string-span(do not unescape) + ReadIsNameSeparatorWithVerify. |
| ReadBoolean | ReadBoolean. |
| ReadSByte | atoi.  |
| ReadInt16 | atoi. |
| ReadInt32 | atoi.  |
| ReadInt64 | atoi.  |
| ReadByte | atoi. |
| ReadUInt16 | atoi. |
| ReadUInt32 | atoi. |
| ReadUInt64 | atoi. |
| ReadUInt16 | atoi. |
| ReadSingle | atod. |
| ReadDouble | atod. |
| GetBufferUnsafe | return underlying buffer. |
| GetCurrentOffsetUnsafe | return underlying offset. |
| GetCurrentJsonToken | Get current token(skip whitespace), do not advance. |

`Read***` methods advance next token when called. JsonReader reads utf8 byte[] to primitive directly.

How to use, see the List formatter.

```csharp
// JsonReader is struct, always pass ref and do not set local variable.
public List<T> Deserialize(ref JsonReader reader, IJsonFormatterResolver formatterResolver)
{
    // check current state is null. when null, advanced offset. if not, do not.
    if (reader.ReadIsNull()) return null;

    var formatter = formatterResolver.GetFormatterWithVerify<T>();
    var list = new List<T>();

    var count = 0; // managing array-count state in outer.
    reader.ReadIsBeginArrayWithVerify(); // read '['
    while (!reader.ReadIsEndArrayWithSkipValueSeparator(ref count)) // loop when reached ']' or advance when ','
    {
        list.Add(formatter.Deserialize(ref reader, formatterResolver));
    }

    return list;
}
```

**JsonWriter**

| Method | Description |
| --- | --- |
| static GetEncodedPropertyName | Get JSON Encoded byte[]. |
| static GetEncodedPropertyNameWithPrefixValueSeparator | Get JSON Encoded byte[] with ',' on prefix. |
| static GetEncodedPropertyNameWithBeginObject | Get JSON Encoded byte[] with '{' on prefix. |
| static GetEncodedPropertyNameWithoutQuotation | Get JSON Encoded byte[] without pre/post '"'. |
| CurrentOffset | Get current offset. |
| AdvanceOffset | Advance offset manually. |
| GetBuffer | Get current buffer. |
| ToUtf8ByteArray | Finish current buffer to byte[]. |
| ToString | Finish current buffer to json stirng. |
| EnsureCapacity | Ensure inner buffer capacity. |
| WriteRaw | Write byte/byte[] directly. |
| WriteRawUnsafe | WriteRaw but don't check and ensure capacity. |
| WriteBeginArray | Write '['. |
| WriteEndArray | Write ']'. |
| WriteBeginObject | Write '{'. |
| WriteEndObject | Write '}'. |
| WriteValueSeparator | Write ','. |
| WriteNameSeparator | Write ':'. |
| WritePropertyName | WriteString + WriteNameSeparator. |
| WriteQuotation | Write '"'. |
| WriteNull | Write 'null'. |
| WriteBoolean | Write 'true' or 'false'. |
| WriteTrue | Write 'true'. |
| WriteFalse | Write 'false'. |
| WriteSByte | itoa.  |
| WriteInt16 | itoa. |
| WriteInt32 | itoa.  |
| WriteInt64 | itoa.  |
| WriteByte | itoa. |
| WriteUInt16 | itoa. |
| WriteUInt32 | itoa. |
| WriteUInt64 | itoa. |
| WriteUInt16 | itoa. |
| WriteSingle | dtoa. |
| WriteDouble | dtoa. |

`GetBuffer`, `ToUtf8ByteArray`, `ToString` get the wrote result. JsonWriter writes primitive to utf8 bytes directly.

How to use, see the List formatter.

```csharp
// JsonWriter is struct, always pass ref and do not set local variable.
public void Serialize(ref JsonWriter writer, List<T> value, IJsonFormatterResolver formatterResolver)
{
    if (value == null) { writer.WriteNull(); return; }

    var formatter = formatterResolver.GetFormatterWithVerify<T>();

    writer.WriteBeginArray(); // "["
    if (value.Count != 0)
    {
        formatter.Serialize(ref writer, value[0], formatterResolver);
    }
    for (int i = 1; i < value.Count; i++)
    {
        writer.WriteValueSeparator(); // write "," manually
        formatter.Serialize(ref writer, value[i], formatterResolver);
    }
    writer.WriteEndArray(); // "]"
}
```

Resolver
---
`IJsonFormatterResolver` is storage of typed serializers. Serializer api accepts resolver and can customize serialization.

| Resovler Name | Description |
| --- | --- |
| BuiltinResolver | Builtin primitive and standard classes resolver. It includes primitive(int, bool, string...) and there nullable, array and list. and some extra builtin types(Guid, Uri, BigInteger, etc...). |
| DynamicGenericResolver | Resolver of generic type(`Tuple<>`, `List<>`, `Dictionary<,>`, `Array`, etc). It uses reflection call for resolve generic argument at first time. |
| AttributeFormatterResolver | Get formatter from `[MessagePackFormatter]` attribute. |
| EnumResolver | `EnumResolver.Default` serialize as name, `EnumResolver.UnderlyingValue` serialize as underlying value. Deserialize, can be both. |
| StandardResolver | Composited resolver. It resolves in the following order `object fallback -> (builtin -> enum -> dynamic generic -> attribute ->  dynamic object)`. `StandardResolver.Default` is default resolver of JsonSerialzier and has many option resolvers, see below. |
| CompositeResolver | Singleton custom composite resolver.  |

StandardResolver has 12 option resolvers it combinate

* AllowPrivate = true/false
* ExcludeNull = true/false
* NameMutate = Original/CamelCase/SnakeCase.

for example `StandardResolver.SnakeCase`, `StandardResolver.ExcludeNullCamelCase`, `StandardResolver.AllowPrivateExcludeNullSnakeCase`. `StandardResolver.Default` is AllowPrivate:False, ExcludeNull:False, NameMutate:Original.

Assemble the resolver's priority is the only configuration point of Utf8Json. It is too simple but well works. In most cases, it is sufficient to have one custom resolver globally. CompositeResolver will be its helper. It is also necessary to use extension resolver like `Utf8Json.ImmutableCollection` that add support for for System.Collections.Immutable library. It adds `ImmutableArray<>`, `ImmutableList<>`, `ImmutableDictionary<,>`, `ImmutableHashSet<>`, `ImmutableSortedDictionary<,>`, `ImmutableSortedSet<>`, `ImmutableQueue<>`, `ImmutableStack<>`, `IImmutableList<>`, `IImmutableDictionary<,>`, `IImmutableQueue<>`, `IImmutableSet<>`, `IImmutableStack<>` serialization support.

```csharp
// use global-singleton CompositeResolver.
// This method initialize CompositeResolver and set to default MessagePackSerializer
CompositeResolver.RegisterAndSetAsDefault(new IJsonFormatter[] {
    // add custome formatters, use other DateTime format.
    // if target type is struct, requires add nullable formatter too(use NullableXxxFormatter or StaticNullableFormatter(innerFormatter))
    new DateTimeFormatter("yyyy-MM-dd HH:mm:ss"),
    new NullableDateTimeFormatter("yyyy-MM-dd HH:mm:ss")
}, new[] {
    // resolver custom types first
    ImmutableCollectionResolver.Instance,
    EnumResolver.UnderlyingValue,

    // finaly choose standard resolver
    StandardResolver.AllowPrivateExcludeNullSnakeCase
});
```

```csharp
// select resolver per invoke.
JsonSerializer.Serialize(a, StandardResolver.Default);
JsonSerializer.Serialize(a, StandardResolver.SnakeCase);
JsonSerializer.Serialize(b, CompositeResolver.Instance);
```

You can also build own custom composite resolver.

```csharp
// create custom composite resolver per project is recommended way.
// let's try to copy and paste:)
public class ProjectDefaultResolver : IJsonFormatterResolver
{
    public static IJsonFormatterResolver Instance = new ProjectDefaultResolver();

    // configure your resolver and formatters.
    static IJsonFormatter[] formatters = new IJsonFormatter[]{
        new DateTimeFormatter("yyyy-MM-dd HH:mm:ss"),
        new NullableDateTimeFormatter("yyyy-MM-dd HH:mm:ss")
    };

    static readonly IJsonFormatterResolver[] resolvers = new[]
    {
        ImmutableCollectionResolver.Instance,
        EnumResolver.UnderlyingValue,
        StandardResolver.AllowPrivateExcludeNullSnakeCase
    };

    ProjectDefaultResolver()
    {
    }

    public IJsonFormatter<T> GetFormatter<T>()
    {
        return FormatterCache<T>.formatter;
    }

    static class FormatterCache<T>
    {
        public static readonly IJsonFormatter<T> formatter;

        static FormatterCache()
        {
            foreach (var item in formatters)
            {
                foreach (var implInterface in item.GetType().GetTypeInfo().ImplementedInterfaces)
                {
                    var ti = implInterface.GetTypeInfo();
                    if (ti.IsGenericType && ti.GenericTypeArguments[0] == typeof(T))
                    {
                        formatter = (IJsonFormatter<T>)item;
                        return;
                    }
                }
            }

            foreach (var item in resolvers)
            {
                var f = item.GetFormatter<T>();
                if (f != null)
                {
                    formatter = f;
                    return;
                }
            }
        }
    }
}
```

JsonFormatterAttribute
---
JsonFormatterAttribute is lightweight extension point. This is like JSON.NET's JsonConverterAttribute. You can change to use formatter per type and member.

```csharp
// if serializing, choosed CustomObjectFormatter.
[JsonFormatter(typeof(CustomObjectFormatter))]
public class CustomObject
{
    string internalId;

    public CustomObject()
    {
        this.internalId = Guid.NewGuid().ToString();
    }

    class CustomObjectFormatter : IJsonFormatter<CustomObject>
    {
        public void Serialize(ref JsonWriter writer, CustomObject value, IJsonFormatterResolver formatterResolver)
        {
            formatterResolver.GetFormatterWithVerify<string>().Serialize(ref writer, value.internalId, formatterResolver);
        }

        public CustomObject Deserialize(ref JsonReader reader, IJsonFormatterResolver formatterResolver)
        {
            var id = formatterResolver.GetFormatterWithVerify<string>().Deserialize(ref reader, formatterResolver);
            return new CustomObject { internalId = id };
        }
    }
}
```

JsonFormatter can receive parameter and can attach to member. For example, configure DateTime format.

```csharp
public class Person
{
    public int Age { get; set; }
    public string Name { get; set; }

    [JsonFormatter(typeof(DateTimeFormatter), "yyyy-MM-dd")]
    public DateTime Birth { get; set; }
}
```

`DateTime`, `DateTimeOffset`, `TimeSpan` is used ISO8601 format in default by `ISO8601DateTimeFormatter`, `ISO8601DateTimeOffsetFormatter`, `ISO8601TimeSpanFormatter` but if you want to configure format, you can use `DateTimeFormatter`, `DateTimeOffsetFormatter`, `TimeSpanFormatter` with format string argument.

Text Protocol Foundation
---
Utf8Json implements fast itoa/atoi, dtoa/atod. It can be useful for text protocol serialization. For example I'm implementing [MySqlSharp](https://github.com/neuecc/MySqlSharp/) that aims fastest MySQL Driver on C#(work in progress yet), MySQL protocol is noramlly text so requires fast parser for text protocol.

`Utf8Json.Internal.NumberConverter` is Read/Write primitive to bytes. It is `public` API so you can use if requires itoa/atoi, dtoa/atod algorithm.

```csharp
byte[] buffer = null; // buffer is automatically ensure.
var offset = 0;
var writeSize = NumberConverter.WriteInt64(ref buffer, offset, 99999);

int readCount;
var value = NumberConverter.ReadInt64(buffer, 0, out readCount);
```

for Unity
---
Unity has the [JsonUtility](https://docs.unity3d.com/2017.2/Documentation/Manual/JSONSerialization.html). It is well fast but has many limitations, can not serialize/deserialize dictionary or other collections and nullable, can not root array, can not handle null correctly, etc... Utf8Json has no limitation and performance is same or better especialy convert to/from byte[], Utf8Json achieves true no zero-allocation.

In Unity version, added `UnityResolver` to StandardResolver in default. It enables serialize `Vector2`, `Vector3`, `Vector4`, `Quaternion`, `Color`, `Bounds`, `Rect`.

`.unitypackage` is exists in [releases](https://github.com/neuecc/Utf8Json/releases) page. Zip archives includes unitypackage and code-generator.

Pre Code Generation(Unity/Xamarin Supports)
---
Utf8Json generates object formatter dynamically by [ILGenerator](https://msdn.microsoft.com/en-us/library/system.reflection.emit.ilgenerator.aspx). It is fast and transparently generated at run time. But it does not work on AOT environment(Xamarin, Unity IL2CPP, etc.).

If you want to run on IL2CPP(or other AOT env), you need pre-code generation. `Utf8Json.UniversalCodeGenerator.exe` is code generator of Utf8Json. It is exists in releases page's for unity.zip. It is using [Roslyn](https://github.com/dotnet/roslyn) so analyze source code and created by [.NET Core](https://www.microsoft.com/net/) for cross platform application.

```
arguments help:
  -i, --inputFiles=VALUE        [optional]Input path of cs files(',' separated)
  -d, --inputDirs=VALUE         [optional]Input path of dirs(',' separated)
  -o, --output=VALUE            [required]Output file path
  -f, --allowInternal           [optional, default=false]Allow generate internal(friend)
  -c, --conditionalsymbol=VALUE [optional, default=empty]conditional compiler symbol
  -r, --resolvername=VALUE      [optional, default=GeneratedResolver]Set resolver name
  -n, --namespace=VALUE         [optional, default=MessagePack]Set namespace root name
```

```csharp
// Simple usage(directory)
Utf8Json.UniversalCodeGenerator.exe -d "..\src\Shared\Request,..\src\Shared\Response" -o "Utf8JsonGenerated.cs"
```

If you create DLL by msbuild project, you can use Pre/Post build event or hook your Unity's post/pre process.

```xml
<PropertyGroup>
    <PreBuildEvent>
        Utf8Json.UniversalCodeGenerator.exe, here is useful for analyze/generate target is self project.
    </PreBuildEvent>
    <PostBuildEvent>
        Utf8Json.UniversalCodeGenerator.exe, here is useful for analyze target is another project.
    </PostBuildEvent>
</PropertyGroup>
```

In default, generates resolver to Utf8Json.Resolvers.GeneratedResolver and formatters generates to Utf8Json.Formatters.***. And application launch, you need to set Resolver at first.

```csharp
// CompositeResolver is singleton helper for use custom resolver.
// Ofcourse you can also make custom resolver.
Utf8Json.Resolvers.CompositeResolver.RegisterAndSetAsDefault(
    // use generated resolver first, and combine many other generated/custom resolvers
    Utf8Json.Resolvers.GeneratedResolver.Instance,

    // set StandardResolver or your use resolver chain
    Utf8Json.StandardResolver.Default,
);
```

How to Build
---
Open `Utf8Json.sln` on Visual Studio 2017(latest) and install .NET Core 2.0 SDK.

Unity Project is using symbolic link. At first, run `make_unity_symlink.bat` so linked under Unity project. You can open `src\Utf8Json.UnityClient` on Unity Editor.

Author Info
---
Yoshifumi Kawai(a.k.a. neuecc) is a software developer in Japan.  
He is the Director/CTO at Grani, Inc.  
Grani is a mobile game developer company in Japan and well known for using C#.  
He is awarding Microsoft MVP for Visual C# since 2011.  
He is known as the creator of [UniRx](http://github.com/neuecc/UniRx/)(Reactive Extensions for Unity)  

Blog: [https://medium.com/@neuecc](https://medium.com/@neuecc) (English)  
Blog: [http://neue.cc/](http://neue.cc/) (Japanese)  
Twitter: [https://twitter.com/neuecc](https://twitter.com/neuecc) (Japanese)  

License
---
This library is under the MIT License.
