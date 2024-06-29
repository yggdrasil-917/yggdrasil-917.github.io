---
title: "Serialization"
date: 2024-06-27T07:28:57-07:00
draft: false
toc: false
images:
tags: 
  - serialization
---

* [Pupping](#pupping)
* [Text Pupping](#text-file-pupping)
* [Binary Pupping](#binary-file-pupping)
* [Random Thoughts](#random-thoughts)

## Pupping
The method used for serialization in Tempest is referred to as pupping or pup which stands for pack-unpack. I first heard about this method close to a decade ago but the idea of it stuck with me. That idea is fairly simple. Rather than creating serilization and deserialization functions for each object type we instead create pupper objects for each type of medium bundled with a set of read and write functions for each fundamental data type. A pupper object will contain all the data and functions needed to handle both reading and writing for a specific medium like file or network serialization. Take for example, a text file pupper object might contain a file stream that each pup function would use to read and write with it.

```cpp
// Base class snippet for every pupper object
class Pupper
{
  public:
    
    enum class IoMode : uint8
    {
        Read,
        Write
    };
    
    virtual void WriteKey(const String& /*newKey*/, bool /*addNewLine */= false) {}
    
    virtual void PupPrimitive(char& value) = 0;
    virtual void PupPrimitive(String& value) = 0;
    virtual void PupPrimitive(uint8& value) = 0;
    virtual void PupPrimitive(uint16& value) = 0;
    virtual void PupPrimitive(uint32& value) = 0;
    virtual void PupPrimitive(uint64& value) = 0;
    virtual void PupPrimitive(int8& value) = 0;
    virtual void PupPrimitive(int16& value) = 0;
    virtual void PupPrimitive(int32& value) = 0;
    virtual void PupPrimitive(int64& value) = 0;
    virtual void PupPrimitive(half& value) = 0;
    virtual void PupPrimitive(float& value) = 0;
    virtual void PupPrimitive(double& value) = 0;
    virtual void PupPrimitive(bool& value) = 0;

    // Any extra functions...

  protected:
    IoMode mode;
};
```

## Text File Pupping
It is always a good idea to support text based serialization. In Tempest, the text file pupper implements the Pupper base class posted above. There are some missing functions in that snippet but are crucial functions for text file serialization. One of the biggest advantages of using text based serialization is that it is human readable and so it is also very easy to edit and version control. In the editor most things will use this text file pupper object. In the game, game settings are stored in text but everything else is stored in binary.

During the development of the format the text pupper uses I realized I was using a very verbose schema for the text serialization. It was kinda a simpler XML style and this was not really an ideal outcome in my opinion. In the end the schema was redone to look similar to what YAML does. This lead to the serialized text being easier to read and edit. Plus it was also easier to parse so it got a little faster after the reimplementation. The schema YAML uses is essentially a key value pair where the value can be a simple POD(plain old data type) or it could be an array. This is the same for the text file pupper does. Arrays and string value types need to serialize a little bit more data, namely the length of the array or string.

The text file pupper essentially boils down to using a file stream for reading and writing. When deserializing, the pupper loads all the text file's content into RAM so as to avoid reading from disk every time it reads a variables data. This means it operates on a binary blob buffer when reading data and this leads to optimal performance. For serialization it just uses the file stream object directly every time it writes data for a variable instead of writing to a binary blob in memory.

```cpp
// Same functions get used for both serialization and deserialization
Pupper* p;
uint32 version;
p->WriteKey("Version"); 
PupPrimitive(p, version);
```

## Binary File Pupping
The binary file pupper object works exactly the same as the text file pupper. The only difference is that the pup functions for reading and writing are working with binary data instead of text. This is the pupper of choice for serializing assets and several other things in the engine. This allows serialization to take less disk space and generally makes it fast to load at runtime.

## Random Thoughts
When serializing structs, it's important to also serialize data that can identify the version of that struct. This allows for versioning the data of a struct that gets serialized. For example, if a member variable is removed from serialization you will want to know if you are loading serialized data that still has that removed variable. With a simple version number on the struct being serialized you can have a more robust serialization system. In the example given, after removing the variable from serialization you would also increment the version number for that struct. Then during deserialization you can compare the serialized version number against what the latest struct version number is and then make decisions according to those differences. In Tempest 4 bytes are used to represent the version number of a struct, a uint32. This is likely a bit too much for most cases where maybe 2 or even 1 byte might be enough to represent all possible versions of a certain struct before launching a game.

Also worth mentioning is that new kinds of pupper objects can be created that are not simply writing to a local file. For instance, if you need a network serializer then this is possible to do using this framework. The same kinds of read and write functions can be overwritten to write to memory buffer and at a later point just sent over the network. I'm sure there are plenty of other use cases where this pupping framework can be applied to great success and the best part about it is that it is very simple to implement and maintain.