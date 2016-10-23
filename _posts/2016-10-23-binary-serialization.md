---
layout: post
title: Binary serialization - how not to lose backward compatibility
---

### How to get more control over the serialization process and what for ###

This small example shows how you can lose backward compatibility in your programs, if you do not take care in advance about this.
And ways to get more control of serialization process

At first, we will write an example of the first version of the program:

Version 1

```csharp
[Serializable]
class Data
{
    [OptionalField]
    private int _version;
    
    public int Version
    {
        get { return _version; }
        set { _version = value; }
    }
}
```

And now, let us assume that in the second version of the program added a new class. And we need to store it in an array.

Now code will look like this:

Version 2

```csharp
[Serializable]
class NewItem
{
    [OptionalField]
    private string _name;

    public string Name
    {
        get { return _name; }
        set { _name = value; }
    }
}

[Serializable]
class Data
{
    [OptionalField]
    private int _version;

    public int Version
    {
        get { return _version; }
        set { _version = value; }
    }

    [OptionalField]
    private List<NewItem> _newItems;

    public List<NewItem> NewItems
    {
        get { return _newItems; }
        set { _newItems = value; }
    }
}
```

And code for serialize and deserialize

```csharp
private static byte[] SerializeData(object obj)
{
    var binaryFormatter = new BinaryFormatter();
    using (var memoryStream = new MemoryStream())
    {
        binaryFormatter.Serialize(memoryStream, obj);
        return memoryStream.ToArray();
    }
}

private static object DeserializeData(byte[] bytes)
{
    var binaryFormatter = new BinaryFormatter();
    using (var memoryStream = new MemoryStream(bytes))
        return binaryFormatter.Deserialize(memoryStream);
}
```

_And so, what would happen when you serialize the data in the program of v2 and will try to deserialize them in the program of v1?_

You get an exception:

```csharp
System.Runtime.Serialization.SerializationException was unhandled
Message=The ObjectManager found an invalid number of fixups. This usually indicates a problem in the Formatter.Source=mscorlib
StackTrace:
   at System.Runtime.Serialization.ObjectManager.DoFixups()
   at System.Runtime.Serialization.Formatters.Binary.ObjectReader.Deserialize(HeaderHandler handler, __BinaryParser serParser, Boolean fCheck, Boolean isCrossAppDomain, IMethodCallMessage methodCallMessage)
   at System.Runtime.Serialization.Formatters.Binary.BinaryFormatter.Deserialize(Stream serializationStream, HeaderHandler handler, Boolean fCheck, Boolean isCrossAppDomain, IMethodCallMessage methodCallMessage)
   at System.Runtime.Serialization.Formatters.Binary.BinaryFormatter.Deserialize(Stream serializationStream)
   at Microsoft.Samples.TestV1.Main(String[] args) in c:\Users\andrew\Documents\Visual Studio 2013\Projects\vts\CS\V1 Application\TestV1Part2\TestV1Part2.cs:line 29
   at System.AppDomain._nExecuteAssembly(Assembly assembly, String[] args)
   at Microsoft.VisualStudio.HostingProcess.HostProc.RunUsersAssembly()
   at System.Threading.ExecutionContext.Run(ExecutionContext executionContext, ContextCallback callback, Object state)
   at System.Threading.ThreadHelper.ThreadStart()
```

**Why?**

The ObjectManager has a different logic to resolve dependencies for arrays and for reference and value types.
We added an array of new the reference type which is absent in our assembly.


When ObjectManager attempts to resolve dependencies it builds the graph.
When it sees the array, it can not fix it immediately, so that it creates a dummy reference and then fixes the array later.


And since this type is not in the assembly and dependencies can't be fixed. For some reason, it does not remove the array from the list of elements for the fixes and at the end it throws an exception "IncorrectNumberOfFixups".


It is some 'gotchas' in the process of serialization.
For some reason, it does not work correctly only for arrays of new reference types.

```
A Note:
Similar code will work correctly, if you do not use arrays with new classes
```

_And the first way to fix it and maintain compatibility?_

_Use a collection of new structures rather than classes or use a dictionary(possible classes), because a dictionary it's a collection of keyvaluepair(it's structure)_


### Get more control over the serialization ###

And now ways to get more control over the serialization process:

**1. SerializationBinder**

The gives you an opportunity to inspect what types are being loaded in your application domain.
SerializationBinder can also be used for security.There might be some security exploits when you are trying to deserialize some data from an untrusted source:

```csharp
class MyBinder : SerializationBinder
{
    public override Type BindToType(string assemblyName, string typeName)
    {
        if (typeName.Equals("BinarySerializationExample.Item"))
            return typeof(Item);
        return null;
    }
}
```

Now we can check what types are loading and on this basis to decide what we really want to receive

For using a binder, you must add it to the BinaryFormatter

```csharp
object DeserializeData(byte[] bytes)
{
    var binaryFormatter = new BinaryFormatter();
    binaryFormatter.Binder = new MyBinder();

    using (var memoryStream = new MemoryStream(bytes))
        return binaryFormatter.Deserialize(memoryStream);
}
```

**2. SerializationSurrogates**

Serialization surrogate selector that allows one object to perform serialization and deserialization of another object  and can transform the serialized data if necessary.
As well allows to properly serialize or deserialize a class that is not itself \[Serializable\].

```csharp
public class ItemSurrogate : ISerializationSurrogate
{
    public void GetObjectData(object obj, SerializationInfo info, StreamingContext context)
    {
        var item = (Item)obj;
        info.AddValue("_name", item.Name);
    }

    public object SetObjectData(object obj, SerializationInfo info, StreamingContext context, ISurrogateSelector selector)
    {
        var item = (Item)obj;
        item.Name = (string)info.GetValue("_name", typeof(string));
        return item;
    }
}
```

Then you need to let your IFormatter know about the surrogates by defining and initializing a SurrogateSelector and assigning it to your BinaryFormatter

```csharp
var surrogateSelector = new SurrogateSelector();
surrogateSelector.AddSurrogate(typeof(Item), new StreamingContext(StreamingContextStates.All), new ItemSurrogate());   
var binaryFormatter = new BinaryFormatter{
    SurrogateSelector = surrogateSelector
};
```

Even if the class is not marked Serializable.

```csharp
//this class is not serializable
public class Item
{
    private string _name;

    public string Name
    {
        get { return _name; }
        set { _name = value; }
    }
}
```

**3. ISerialization**

Allows an object to control its own serialization and deserialization

```csharp
[Serializable]
public class Item : ISerializable
{
    private string _name;

    public string Name
    {
        get { return _name; }
        set { _name = value; }
    }

    public Item ()
    {

    }

    protected Item (SerializationInfo info, StreamingContext context)
    {
        _name = (string)info.GetValue("_name", typeof(string));
    }

    public void GetObjectData(SerializationInfo info, StreamingContext context)
    {
        info.AddValue("_name", _name, typeof(string));
    }
}
```

_Now, we can control how the data will be serialized._


_And the second solution to the problem, which I described in the beginning, how to maintain backward compatibility, if we do not taken care of this before_


_Just specify that an array of new classes should not serialize as an array, but merely as a sequence of elements, without declaring an array._


```csharp
[Serializable]
class NewItem
{
    public string Name;
}

[Serializable]
class Data : ISerializable
{
    [OptionalField]
    private int _version;

    public int Version
    {
        get { return _version; }
        set { _version = value; }
    }

    [OptionalField]
    private List<NewItem> _newItems;
    
    public List<NewItem> NewItems
    {
        get { return _newItems; }
        set { _newItems = value; }
    }

    public Data()
    {

    }

    protected Data(SerializationInfo info, StreamingContext context)
    {
        var newItemsSize = (int)info.GetValue("_newItemsSize", typeof(int));
        _newItems = new List<NewItem>(newItemsSize);
        for (int i = 0; i < newItemsSize; i++)
        {
            var item = (NewItem)info.GetValue($"_newItem{i}", typeof(NewItem));
            _newItems.Add(item);
        }
    }
    
    public void GetObjectData(SerializationInfo info, StreamingContext context)
    {
        info.AddValue("_newItemsSize", _newItems.Count, typeof(int));
        for (int i = 0; i < _newItems.Count; i++)
            info.AddValue($"_newItem{i}", _newItems[i], typeof(NewItem));
    }
}
```

                 
Of course, it's not very nice solution, but so we can solve the problem.

And best of all, of course not to use a binary serialization for this task


