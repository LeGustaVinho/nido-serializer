# Nido Serializer

[Odin Serializer](https://github.com/TeamSirenix/odin-serializer "Odin Serializer") fork compatiple with Unity Package Manager

**Note:** If your project already uses [Odin Inspector](https://odininspector.com/ "Odin Inspector"), remember that it already incorporates Odin Serializer, making the use of this package unnecessary.

### How to install

#### - From OpenUPM:

- Open **Edit -> Project Settings -> Package Manager**
- Add a new Scoped Registry (or edit the existing OpenUPM entry)

| Name  | package.openupm.com  |
| ------------ | ------------ |
| URL  | https://package.openupm.com  |
| Scope(s)  | com.legustavinho  |

- Open Window -> Package Manager
- Click `+`
- Select `Add package by name...`
- Paste `com.legustavinho.legendary-tools-nido-serializer` and click `Add`

#### - From Git:
- Open **Window -> Package Manager**
- Click `+`
- Select `Add package from git URL...`
- Paste and press `Add` for all these URLs, do it in this order:
    - https://github.com/LeGustaVinho/nido-serializer.git

### Basic usage of OdinSerializer

This section will not go into great detail about how OdinSerializer works or how to configure it in advanced ways - for that, see the technical overview further down. Instead, it aims to give a simple overview of how to use OdinSerializer in a basic capacity.

There are, broadly, two different ways of using OdinSerializer:

#### Serializing regular C# objects

You can use OdinSerializer as a standalone serialization library, simply serializing or deserializing whatever data you give it, for example to be stored in a file or sent over the network. This is done using the SerializationUtility class, which contains a variety of methods that wrap OdinSerializer for straight-forward, easy use.

###### Example: Serializing regular C# objects

```csharp
using OdinSerializer;

public static class Example
{
	public static void Save(MyData data, string filePath)
	{
		byte[] bytes = SerializationUtility.SerializeValue(data, DataFormat.Binary);
		File.WriteAllBytes(bytes, filePath);
	}
	
	public static MyData Load(string filePath)
	{
		byte[] bytes = File.ReadAllBytes(filePath);
		return SerializationUtility.DeserializeValue<MyData>(bytes, DataFormat.Binary);
	}
}
```

Note that you cannot save references to Unity objects or assets down to a file in this manner. The only way to handle this is to ask for a list of all encountered UnityEngine.Object references when serializing, and then pass that list of references back into the system when deserializing data.

###### Example: Serializing regular C# objects containing Unity references

```csharp
using OdinSerializer;

public static class Example
{
	public static void Save(MyData data,  string filePath, ref List<UnityEngine.Object> unityReferences)
	{
		byte[] bytes = SerializationUtility.SerializeValue(data, DataFormat.Binary, out unityReferences);
		File.WriteAllBytes(bytes, filePath);
		
		// The unityReferences list will now be filled with all encountered UnityEngine.Object references, and the saved binary data contains index pointers into this list.
		// It is your job to ensure that the list of references stays the same between serialization and deserialization.
	}
	
	public static MyData Load(string filePath, List<UnityEngine.Object> unityReferences)
	{
		byte[] bytes = File.ReadAllBytes(filePath);
		return SerializationUtility.DeserializeValue<MyData>(bytes, DataFormat.Binary, unityReferences);
	}
}
```

#### Extending UnityEngine.Object serialization

You can also use OdinSerializer to seamlessly extend the serialization of Unity's object types, such as ScriptableObject and MonoBehaviour. There are two general ways of doing this, one of which is manual and requires a few lines of code to implement, and one of which is very easy to implement, but exhibits only the default behaviour.

The easier approach is to simply derive your type from one of many pre-created UnityEngine.Object-derived types that OdinSerializer provides, that have the above behaviour implemented already. Note that doing this will have the default behaviour of not serializing fields that Unity will serialize.

###### Example: Easily extending UnityEngine.Object serialization

```csharp
using OdinSerializer;

public class YourSpeciallySerializedScriptableObject : SerializedScriptableObject
{
	public Dictionary<string, string> iAmSerializedByOdin;
	public List<string> iAmSerializedByUnity;
}
```

The manual method requires that you implement Unity's ISerializationCallbackReceiver interface on the UnityEngine.Object-derived type you want to extend the serialization of, and then use OdinSerializer's UnitySerializationUtility class to apply Odin's serialization during the serialization callbacks that Unity invokes at the appropriate times.

###### Example: Manually extending UnityEngine.Object serialization

```csharp
using UnityEngine;
using OdinSerializer;

public class YourSpeciallySerializedScriptableObject : ScriptableObject, ISerializationCallbackReceiver
{
	[SerializeField, HideInInspector]
	private SerializationData serializationData;

	void ISerializationCallbackReceiver.OnAfterDeserialize()
	{
		// Make Odin deserialize the serializationData field's contents into this instance.
		UnitySerializationUtility.DeserializeUnityObject(this, ref this.serializationData, cachedContext.Value);
	}

	void ISerializationCallbackReceiver.OnBeforeSerialize()
	{
		// Whether to always serialize fields that Unity will also serialize. By default, this parameter is false, and OdinSerializer will only serialize fields that it thinks Unity will not handle.
		bool serializeUnityFields = false;
		
		// Make Odin serialize data from this instance into the serializationData field.
		UnitySerializationUtility.SerializeUnityObject(this, ref this.serializationData, serializeUnityFields, cachedContext.Value);
	}
}
```

NOTE: If you use OdinSerializer to extend the serialization of a Unity object, without having an inspector framework such as Odin Inspector installed, the Odin-serialized fields will not be rendered properly in Unity's inspector. You will either have to acquire such a framework, or write your own custom editor to be able to inspect and edit this data in Unity's inspector window.

Additionally, always remember that Unity doesn't strictly know that this extra serialized data exists - whenever you change it from your custom editor, remember to manually mark the relevant asset or scene dirty, so Unity knows that it needs to be re-serialized.

Finally, note that *prefab modifications will not simply work by default in specially serialized Components/Behaviours/MonoBehaviours*. Specially serialized prefab instances may explode and die if you attempt to change their custom-serialized data from the parent prefab. OdinSerializer contains a system for managing an object's specially-serialized prefab modifications and applying them, but this is an advanced use of OdinSerializer that requires heavy support from a specialised custom editor, and this is not covered in this readme.

## How to contribute

We are taking contributions under the Apache 2.0 license, so please feel free to submit pull requests. Please keep in mind the following rules when submitting contributions:

* Follow the pre-existing coding style and standards in the OdinSerializer code.
* If you work in your own fork with a modified OdinSerializer namespace, please ensure that the pull request uses the correct namespaces for this repository and otherwise compiles right away, so we don't need to clean that sort of stuff up when accepting the pull request.

We are taking any contributions that add value to OdinSerializer without also adding undue bloat or feature creep. However, these are the areas that we are particularly interested in seeing contributions in:

#### Bugfixes
* We would be very grateful if you could submit pull requests for any bugs that you encounter and fix.

#### Performance
* General overall performance: faster code is always better, as long as the increase in speed does not sacrifice any robustness.
* Json format performance: the performance of the json format (JsonDataWriter/JsonDataReader) is currently indescribably awful. The format was originally written as a testbed for use during the development of OdinSerializer, since it is human-readable and thus very useful for debugging purposes, and has remained largely untouched since.
* EnumSerializer currently allocates garbage via boxing serialized and deserialized enum values. As such, serializing enums always results in unnecessary garbage being allocated. Any approaches for fixing this would be most welcome. Some unsafe code may be required, but we haven't yet had time to really look into this properly.

#### Testing
* A thorough set of standalone unit tests. Odin Inspector has its own internal integration tests for OdinSerializer, but currently we have no decent stand-alone unit tests that work solely with OdinSerializer.

## Technical Overview

The following section is a brief technical overview of the working principles of OdinSerializer and many of its bigger features. First, however, let's have a super brief overview, to give some context before we begin.

This is how OdinSerializer works, on the highest level:

* Data to be written to or read from is passed to a data writer/reader, usually in the form of a stream.
* The data writer/reader is passed to a serializer, along with a value to be serialized, if we are serializing.
* If the value can be treated as an atomic primitive, the serializer will write or read that directly using the passed data writer/reader. If the value is "complex", IE, it is a value that consists of other values, the serializer will get and wrap the use of a formatter to read or write the value.

### "Stack-only", forward-only

OdinSerializer is a forward-only serializer, meaning that when it serializes, it writes data immediately as it inspects the object graph, and when it deserializes, it recreates the object graph immediately as it parses the data. The serializer only ever moves forward - it cannot "go back" and look at previous data, since we retain no state and are doing everything on the fly, as we move forward. Unlike some other serializers, there is no "meta-graph" data structure that is allocated containing all the data to be saved down later.

This means that we can serialize and deserialize data entirely without allocating anything on the heap, meaning that after the system has run once and all the writer, reader, formatter and serializer instances have been created, there will often be literally zero superfluous GC allocations made, depending on the data format used.

### Data writers and readers

Data writers and readers are types that implement the IDataReader and/or IDataWriter interfaces. They abstract the writing and reading of strongly typed C# data in the form of atomic primitives, from the actual raw data format that the data is written into and read from. OdinSerializer currently ships with data readers and writers that support three different formats: Json, Binary and Nodes.

Data writers and readers also contain a serialization or deserialization context, which is used to configure how serialization and deserialization operates in various ways.

### Atomic primitives

An atomic primitive (or merely a primitive), in the context of OdinSerializer, is a type that can be written or read in a single call to a data writer or reader. All other types are considered complex types, and must be handled by a formatter that translates that type into a series of atomic primitives. You can check whether something is an atomic primitive by calling FormatterUtilities.IsPrimitiveType(Type type).

The following types are considered atomic primitives:

* System.Char (char)
* System.String (string)
* System.Boolean (bool)
* System.SByte (sbyte)
* System.Byte (byte)
* System.Short (short)
* System.UShort (ushort)
* System.Int (int)
* System.UInt (uint)
* System.Long (long)
* System.ULong (ulong)
* System.Single (float)
* System.Double (double)
* System.Decimal (decimal)
* System.IntPtr
* System.UIntPtr
* System.Guid
* All enums

### Serializers and formatters

This is an important distinction - serializers are the outward "face" of the system, and are all hardcoded into the system. There is a hardcoded serializer type for each atomic primitive, and a catch-all ComplexTypeSerializer that handles all other types by wrapping the use of formatters.

Formatters are what translates an actual C# object into the primitive data that it consists of. They are the primary point of extension in OdinSerializer - they tell the system how to treat various special types. For example, there is a formatter that handles arrays, a formatter that handles multi-dimensional arrays, a formatter that handles dictionaries, and so on.

OdinSerializer ships with a large number of custom formatters for commonly serialized .NET and Unity types. An example of a custom formatter might be the following formatter for Unity's Vector3 type:

```csharp
using OdinSerializer;
using UnityEngine;

[assembly: RegisterFormatter(typeof(Vector3Formatter))]
	
public class Vector3Formatter : MinimalBaseFormatter<Vector3>
{
	private static readonly Serializer<float> FloatSerializer = Serializer.Get<float>();

	protected override void Read(ref Vector3 value, IDataReader reader)
	{
		value.x = FloatSerializer.ReadValue(reader);
		value.y = FloatSerializer.ReadValue(reader);
		value.z = FloatSerializer.ReadValue(reader);
	}
	
	protected override void Write(ref Vector3 value, IDataWriter writer)
	{
		FloatSerializer.WriteValue(value.x, writer);
		FloatSerializer.WriteValue(value.y, writer);
		FloatSerializer.WriteValue(value.z, writer);
	}
}
```

All complex types that do not have a custom formatter declared are serialized using either an on-demand emitted formatter, or if emitted formatters are not available on the current platform, using a fallback reflection-based formatter. These "default" formatters use the serialization policy set on the current context to decide which members are serialized.

### Serialization policies

Serialization policies are used by non-custom formatters (emitted formatters and the reflection formatter) to decide which members should and should not be serialized. A set of default policies are provided in the SerializationPolicies class.

### External references

External references are a very useful concept to have. In short, it's a way for serialized data to refer to objects that are not stored in the serialized data itself, but should instead be fetched externally upon deserialization.

For example, if an object graph refers to a very large asset such as a texture, stored in a central asset database, you might not want the entire texture to be serialized down along with the graph, but might instead want to serialize an external reference to the texture, which would then be resolved again upon deserialization. When working with Unity, this feature is particularly useful, as you will see in the next section.

External references in OdinSerializer can be either by index (int), by guid (System.Guid), or by string, and are used by implementing the IExternalIndexReferenceResolver, IExternalGuidReferenceResolver or IExternalStringReferenceResolver interfaces, and setting a resolver instance on the context that is set on the current data reader or writer.

All reference types (except strings, which are treated as atomic primitives) can potentially be resolved externally, and all available external reference resolvers in the current context will be queried for each encountered reference type object as to whether or not that object ought to be serialized as an external reference.

### License

All trademarks, service marks, logos, and code associated with Odin Inspector and Odin Serializer are the exclusive property of Sirenix ApS.
Odin Serializer uses Apache 2.0 license, same license as Nido Serializer.