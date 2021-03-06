---
ID: 213
post_title: Long execution time of Enum.values()
author: Piotr Ciruk
post_excerpt: ""
layout: post
published: true
post_date: 2014-07-16 11:20:42
---
Generally it is not a good idea to implement serialization yourself, unless the code is performance critical. 
Even then there are some efficient open-source libraries available at your service (<a href="https://github.com/google/flatbuffers" target="_blank">Google FlatBuffers</a> or <a href="https://github.com/real-logic/simple-binary-encoding" target="_blank">Simple Binary Encoding</a>).
Like in case of every bad idea, I thought I'd give hand-coded serialization a try.
The subject of serialization process was a set of equity deals for a given day. Not a huge amount, just a collection of approximately 100k records. 
Each deal consists of a number of optional fields. This optionality must be somehow handled during serialization. Simply put, each field is prefixed with a flag indicating whether a value is present.
I didn't want the serialization code to contain raw calls to protcol such as:
```
if (stream.getByte() == 0) {
	value = null;
} else {
	value = new Object();
}
```

Instead, I created a simple enum to abstract this concept:
```
enum ObjectReference {
	NULL((byte) 0),
	NOT_NULL((byte) 1);
	
	ObjectReference(byte code) {
		this.code = code;
	}
	
	byte code;
	
	public byte getCode() {
		return code;
	}
	
	public static ObjectReference forCode(byte code) {
		for (ObjectReference ref : ObjectReference.values()) {
			if (ref.code == code) {
				return ref;
			}
		}
		
		throw new AssertionError();
	}
}
```

Which was used more or less in the following way:
```
if (ObjectReference.forCode(stream.getByte()) == ObjectReference.NOT_NULL) {
	value = new Object();
}
```

You might already know where this is going, I on the other hand was not aware of exisiting performance bottleneck. 
A Deal class contains almost fifty fields, so for 100k deals ObjectReference.forCode would be called nearly 5 million times. 
After quick profiling it turned out, that for 11 seconds of program execution, roughly 3 seconds were spent within `forCode()` method. What the heck is wrong with the method?
The culprit is call to `values()` method of class `Enum<T>`. This method is not a part of API (it's described in the <a href="http://docs.oracle.com/javase/specs/jls/se7/html/jls-8.html#jls-8.9.2" target="_blank">JLS</a>) but is created at compile time and embedded in enum class file.
Decompiled body of method can be found below. The interesting part is the call to `Object.clone()` for array with enumerated values. Every time you call `Enum<T>.values()` there will be implicit cloning involved.
```
public static ObjectReference[] values();
  flags: ACC_PUBLIC, ACC_STATIC
  Code:
    stack=1, locals=0, args_size=0
       0: getstatic     #1                  // Field $VALUES:[LObjectReference;
       3: invokevirtual #2                  // Method "[LObjectReference;".clone:()Ljava/lang/Object;
       6: checkcast     #3                  // class "[LObjectReference;"
       9: areturn
    LineNumberTable:
      line 1: 0
```

I took advantage of storing values in static array which can be observed in the final (?) version of my enumaration type.
```
public enum ObjectReference {
	NULL((byte) 0),
	NOT_NULL((byte) 1);
	
	static final ObjectReference[] CODES = ObjectReference.values();
	
	private byte code;
	
	public static ObjectReference fromCode(byte code) {
		if (code < CODES.length) {
			return CODES[code];
		}
		
		throw new AssertionError("No ObjectReference for given code: " + code);
	}
	
	private ObjectReference(byte code) {
		this.code = code;
	}
	
	public byte getCode() {
		return code;
	}
}
```