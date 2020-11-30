# Bssom Specification

**Bssom**(**Binary search algorithm structure model object binary marshalling**) is a protocol for structured marshalling of objects using a binary search algorithm model. The marshalled data has special **metadata** information，according to these metadata information, you can efficiently **only read and change** an element in the object，In this way, when serializing and deserializing large objects, you do not have to cause complete serialization overhead because only one field is read or written.

Bssom is different from other binary protocols in that it has two characteristics:  
1. <a name="characteristic1"></a>can read an element in an object without deserializing the object in its entirety, this means that when I design Bssom, I need to allow the designed type to have a special metadata format, so that the metadata can be used to locate the specified element, and read it.
2. <a name="characteristic2"></a>can change an element in an object without serializing the entire object, this requires that some types of bssom should be designed as fixed length structures, and their formats will not change with the change of values.  

Based on these two characteristics, Bssom defines the two concepts of **type system** and **format**.

Serialization is conversion from application objects into Bssom formats via Bssom type system, Deserialization is conversion from Bssom formats into application objects via Bssom type system.

This document introduces the bssom type system and the bssom format.

## Table of contents

* Bssom specification
  * [Type system](#type-system)
      * [Native type](#native-type)
      * [Extension type](#extension-type)
      * [Blank type](#blank-type)
      * [Supplement](#supplement)
  * [Format](#format)
      * [Overview](#overview)
      * [Specific](#specific)
      * [Internal VarUInt Type format](#internal-varuint-type-format)
      * [Blank Type format](#blank-type-format)
      * [Null Type format](#null-type-format)
      * [Boolean Type format](#boolean-type-format)
      * [Number Type format](#number-type-format)
      * [Float Type format](#float-type-format)
      * [String Type format](#string-type-format)
      * [Timestamp Type format](#timestamp-type-format)
      * [Array Type format](#array-type-format)
      	* [Array1](#array1)
      	* [Array2](#array1)
      	* [Array3](#array3)
      * [Map Type format](#map-type-format)
      	* [Map1](#map1)
      	* [Map2](#map2)
      * [Native Type format](#native-type-format)
      * [Extend Type format](#extend-type-format)
  * [Implementation](#implementation)
  
## Type system
The Bssom type system defines 11 categories
  * **Null** represents null
  * **Number** integer type, is a summary of 8 integer formats, including Int8,Int16,Int32,Int64,UInt8,UInt16,UInt32,UInt64
  * **Boolean** represents true or false
  * **Float** represents the floating point type defined in the [IEEE754](https://en.wikipedia.org/wiki/IEEE_754) standard, including Float32,Float64  
  * **String** represents a string in UTF-8 format
  * **Timestamp** represents [Unix timestamp](https://en.wikipedia.org/wiki/Unix_time)
  * **Extension** to ensure the possibility of Bssom expansion in the future, the application should support it as much as possible, and its meaning is determined by the Bssom specification
  * **Native** used for the native type format created by the application itself, its implementation does not have universality and interactivity, and its meaning is determined by the application itself
  * **Array** represents a sequence of a set of elements, including three sequence formats Array1,Array2,Array3  
  	* ***Array1***: constrained format, the elements in the sequence are all the same type format, the element type is recorded in the header information, the element itself will no longer retain the type information, and the width of each element in the sequence is equal
	* ***Array2***: unconstrained format, each element has its own type information, and the elements in the sequence can be in any format
	* ***Array3***: format contains the offset information and element data of the element. The offset information is used to quickly locate the element data. Each element has its own type information. The elements in the sequence can be in any format.
  * **Map** represents key-value pairs, contains two formats Map1,Map2
	* ***Map1***: format does not contain structured metadata information, does not require preprocessing when marshalling, and is suitable for complete writing and reading
	* ***Map2***: format contains the metadata information of the binary search algorithm structure model, which can quickly locate the target value through the metadata, and is suitable for high-speed partial writing and partial reading
  * **Blank** represents a blank byte, used to fill the difference slot caused by changing the element value

### Native type:
Bssom allows applications to define application-specific types using native types, the defined native type format consists of an integer and data, the integer represents the data length of the native type. When performing cross-program interaction, the application can freely choose whether to process the unknown native type. If you choose to ignore it, you can skip its data by reading the integer.

### Extension type:
In order to ensure the type extension of the Bssom protocol, an extension type is designed. The extension type is composed of an extension type code and data. The application should support it as much as possible, but there is currently no existing extension type.

### Blank type:
When changing the Bssom format object or one of its elements, if the width of the changed value is smaller than the original object, a gap will be generated. This gap needs to be supplemented with fillers. The Blank type is used to fill blank bytes here

### Supplement：
* In bssom, in order to support the [second feature](#characteristic2), the types of Number, Float and Boolean are designed in fixed length format. In this way, the representation of values is always independent of their length in binary.
* The overall type system is divided into three parts: 1. Non-native and non-extended raw type part, 2. Native type part, 3. Extended type part . When KeyType is expressed in Map2 and ElementType is expressed in Array1, if it is a native type, the format is a byte type code; if it is a native type, the format is the type code representing the native type: 0xf2 + the size of the native type; if it is an extended type, the format is the type code representing the extended type, which is 0xf1 + the extended type code


## Format

### Overview

format name     | first byte (in binary) | first byte (in hex)
--------------- | ---------------------- | -------------------
VarBlank		| 0xxxxxxx               | 0x00 - 0x7f
UInt16Blank     | 10000000               | 0x80
UInt32Blank     | 10000001               | 0x81
Null            | 10000010               | 0x82
Int8			| 10000011               | 0x83
Int16           | 10000100               | 0x84
Int32           | 10000101               | 0x85
Int64           | 10000110               | 0x86
UInt8           | 10000111               | 0x87
UInt16          | 10001000               | 0x88
UInt32          | 10001001               | 0x89
UInt64          | 10001010               | 0x8a
Float32         | 10001011               | 0x8b
Float64         | 10001100               | 0x8c
Boolean         | 10001101               | 0x8d
Timestamp       | 10001110               | 0x8e
String          | 10001111               | 0x8f
(never used)	|						 | 0x90 - 0xc0
Map1         	| 11000001               | 0xc1
Map2            | 11000010               | 0xc2
(never used)	|						 | 0xc3 - 0xd0
Array1          | 11010001               | 0xd1
Array2          | 11010010               | 0xd2
Array3          | 11010011               | 0xd3
(never used)	|						 | 0xd4 - 0xf0
ExtendCode      | 11110001               | 0xf1
NativeCode      | 11110010               | 0xf2
(never used)	|						 | 0xf4 - 0xff

### Specific:
There is a variable-length unsigned integer format in Bssom: **VarUInt** , this format is not in the defined type system, and is only used in the internal Bssom format, such as recording the length and number of elements of Array, String, Map
VarUInt format  | first byte  |  range
----------------| ----------- | -----------------
VariableUInt8   | 0x00~0xfa   |  0~250
VariableUInt9   | 0xfb	      |  251~505
FixUInt8	    | 0xfc        |  0~255
FixUInt16	    | 0xfd		  |  0~65535
FixUInt32	    | 0xfe		  |  0~4294967295
FixUInt64	    | 0xff		  |  0~18446744073709551615
		
**The format content for each format is shown in detail next**
----------

### Symbolic representation

    one byte:
    +--------+
    |        |
    +--------+

    a set of bytes:
    +========+
    |        |
    +========+

    objects defined in the format:
    +~~~~~~~~~~~~~~~~~+
    |                 |
    +~~~~~~~~~~~~~~~~~+
	

### Internal VarUInt Type format

VarUInt has six formats, which are used to represent the length of the object inside the Bssom format

    1.VariableUInt8 : Use a byte to store 8-bit positive integers, the minimum value is 0x00, and the maximum value is 0xfa
    +--------+   +--------+
    |  0x00  | ~ |  0xfa  |
    +--------+   +--------+

    2.VariableUInt9 : Use two bytes, the second byte stores 8-bit positive integers, the minimum value is 0xfb+0x00, and the maximum value is 0xfa+0xff
    +--------+--------+   +--------+--------+
    |  0xfb  |  0x00  | ~ |  0xfb  |  0xff  |
    +--------+--------+   +--------+--------+

    3.FixUInt8 : Use two bytes, the second byte stores an 8-bit positive integer
    +--------+--------+
    |  0xfc  |XXXXXXXX|
    +--------+--------+
	
	4.FixUInt16 : Three bytes are used, and small-endian 16-bit positive integers are stored starting from the second byte
    +--------+--------+--------+
    |  0xfd  |XXXXXXXX|XXXXXXXX|
    +--------+--------+--------+
	
	5.FixUInt32 : Five bytes are used, and little-endian 32-bit positive integers are stored starting from the second byte
    +--------+--------+--------+--------+--------+
    |  0xfe  |XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|
    +--------+--------+--------+--------+--------+
	
	6.FixUInt64 : Nine bytes are used, and little-endian 64-bit positive integers are stored starting from the second byte
    +--------+--------+--------+--------+--------+--------+--------+--------+--------+
    |  0xff  |XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|
    +--------+--------+--------+--------+--------+--------+--------+--------+--------+

**Note: in the following format, varuint is represented by the following symbols**

	VarUInt:
    +*------*+
    |        |
    +*------*+

### Blank Type format

There are three formats for the blank type.

	1.variable blank :  A byte to store a 7-bit positive integer, use to represent the next blank byte with a maximum length of 0x7f
    +--------+=========+
    |0XXXXXXX| N*blank |
    +--------+=========+
	
	2.UInt16 Blank : Two bytes are used to store 16 bit positive integers to represent the next blank byte with a maximum length of 2 ^ 16-1
	+--------+--------+--------+=========+
    |  0x80	 |XXXXXXXX|XXXXXXXX| N*blank |
    +--------+--------+--------+=========+
	
	3.UInt32 Blank : Four bytes are used to store 32-bit positive integers to represent the next blank byte with a maximum length of 2 ^ 32-1
	+--------+--------+--------+--------+--------+=========+
    |  0x81	 |XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX| N*blank |
    +--------+--------+--------+--------+--------+=========+

### Null Type format

Null types have only one format.

    null:
    +--------+
    |  0x82  |
    +--------+

### Boolean Type format

The Boolean type has two formats, representing false and true respectively.

    1.false:
    +--------+--------+
    |  0x8d  |  0x00  |
    +--------+--------+

    2.true:
    +--------+--------+
    |  0x8d  |  0x01  |
    +--------+--------+

### Number Type format 

The Number type contains 8 formats Int8,Int16,Int32,Int64,UInt8,UInt16,UInt32,UInt64

    Int8 stores an 8-bit signed integer
    +--------+--------+
    |  0x83  |XXXXXXXX|
    +--------+--------+

    Int16 stores a 16-bit little-endian signed integer
    +--------+--------+--------+
    |  0x84  |XXXXXXXX|XXXXXXXX|
    +--------+--------+--------+

    Int32 stores a 32-bit little-endian signed integer
    +--------+--------+--------+--------+--------+
    |  0x85  |XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|
    +--------+--------+--------+--------+--------+

    Int64 stores a 64-bit little-endian signed integer
    +--------+--------+--------+--------+--------+--------+--------+--------+--------+
    |  0x86  |XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|
    +--------+--------+--------+--------+--------+--------+--------+--------+--------+

    UInt8 stores an 8-bit unsigned integer
    +--------+--------+
    |  0x87  |XXXXXXXX|
    +--------+--------+

    UInt16 stores a 16-bit little-endian unsigned integer
    +--------+--------+--------+
    |  0x88  |XXXXXXXX|XXXXXXXX|
    +--------+--------+--------+

    UInt32 stores a 32-bit little-endian unsigned integer
    +--------+--------+--------+--------+--------+
    |  0x89  |XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|
    +--------+--------+--------+--------+--------+

    UInt64 stores a 64-bit little-endian unsigned integer
    +--------+--------+--------+--------+--------+--------+--------+--------+--------+
    |  0x8a  |XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|
    +--------+--------+--------+--------+--------+--------+--------+--------+--------+

### Float Type format

The Float type contains two formats, Float32,Float64

    Float32 stores a floating-point number in IEEE 754 single-precision floating-point number 32-bit format
    +--------+--------+--------+--------+--------+
    |  0x8b  |XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|
    +--------+--------+--------+--------+--------+

    Float64 stores a floating-point number in IEEE 754 single-precision floating-point number 64-bit format:
    +--------+--------+--------+--------+--------+--------+--------+--------+--------+
    |  0x8c  |YYYYYYYY|YYYYYYYY|YYYYYYYY|YYYYYYYY|YYYYYYYY|YYYYYYYY|YYYYYYYY|YYYYYYYY|
    +--------+--------+--------+--------+--------+--------+--------+--------+--------+

    * When storing, floating-point numbers are stored in little-endian order
    

### String Type format

The string type has only one format, which consists of byte size and Utf8 encoded byte array

    +--------+*-------*+========+
    |  0x8f  | DataLen |  Data  |
    +--------+*-------*+========+
	
	* Data is the byte array after encoding the string with Utf8 encoding
	* DataLen is the length of the encoded byte array
	
### Timestamp Type format

The Timestamp type has only one format, the format is the number of seconds since the Unix Epoch represented by 8 bytes and the increment within a given number of seconds of 4 bytes.

    +--------+------------------------------+------------------------------------+
    |  0x8e  | seconds in 64-bit signed int | nanoseconds in 32-bit unsigned int |
    +--------+------------------------------+------------------------------------+

### Array Type format

The array type has three formats : Array1,Array2,Array3.

<a name="array1">**Array1**</a>  

The Array1 format is composed of header information and a sequence of elements. In the header information, [element type](#supplement) is included, which means that the elements in the array do not need to store type header information, therefore, there are two requirements for array elements in Array1 : **1**.Elements in the array cannot contain type headers, **2**.The width of the elements in the array are the same.  

*Because the elements of Array1 are all fixed-length, the access method of obtaining and changing elements can be obtained by: ElementsPos + index * eleSize*

	+--------+~~~~~~~~~~~~~~~+*------*+*-----*+~~~~~~~~~~~~~~~~~+
    |  0xd1	 |  ElementType  | Length | Count |    N*Element    |
    +--------+~~~~~~~~~~~~~~~+*------*+*-----*+~~~~~~~~~~~~~~~~~+

	* ElementsPos refers to the starting position of the element sequence (the end offset of the Count field)
	* ElementType : see type system
	* The value of Length is the end of the entire data minus the starting offset of Count
	* The value of Count is the number of elements
	* N represents the number of elements
	
<a name="array2">**Array2**</a>  

Array2 format is composed of length, number of elements and element sequence, unlike Array1, Array2 has no requirements for elements.

*Because there are no restrictions on the width of elements in Array2, the access method of getting elements and changing elements can be obtained by: ElementsPos + index * SkipElement*
	
	+--------+*------*+*-----*+~~~~~~~~~~~~~~~~~+
    |  0xd2	 | Length | Count |    N*Element    |
    +--------+*------*+*-----*+~~~~~~~~~~~~~~~~~+

	* ElementsPos refers to the starting position of the element sequence (the end offset of the Count field)
	* The value of Length is the end of the entire data minus the starting offset of Count
	* The value of Count is the number of elements

<a name="array3">**Array3**</a>  
  
Array3 format is composed of length, number of elements, element offset sequence, element sequence.

*Because Array3 has a sequence of element offsets, the access method of getting elements and changing elements can be obtained by: BasePos + GetElementOffset(index)*

	+--------+*------*+*-----*+~~~~~~~~~~~~~~~~~+~~~~~~~~~~~~~~~~~+
    |  0xd3	 | Length | Count |  ElementOffsets |    N*Element    |
    +--------+*------*+*-----*+~~~~~~~~~~~~~~~~~+~~~~~~~~~~~~~~~~~+

	ElementOffsets is a sequence of positive integers composed of the VarUInt format. Each VarUInt represents the offset of an element at the index based on the starting position (BasePos) of the entire data segment:
	+*--------------*+*--------------*+*--------------*+~~~~~~~~~~+~~~~~~~~~~+~~~~~~~~~~+
    | Element1Offset | Element2Offset | Element3Offset | Element1 | Element2 | Element3 |
    +*--------------*+*--------------*+*--------------*+~~~~~~~~~~+~~~~~~~~~~+~~~~~~~~~~+

	* BasePos is the starting position of the 0xd3 type code
	* The value of Length is the end of the entire data minus the starting offset of Count
	* The value of Count is the number of elements and also represents the number of formats in ElementOffsets


### Map Type format

There are two formats of Map type , Map1 and Map2.

<a name="map1">**Map1**</a>

The Map1 format is composed of header information and a sequence of key-value pairs. The header information contains the data length and the number of key-value pairs.


    +--------+*-------*+*-----*+~~~~~~~~~~~~~~~~~~+
    |  0xc1  | DataLen | Count |  N * (Key,Value) |
    +--------+*-------*+*-----*+~~~~~~~~~~~~~~~~~~+

	* The value of DataLen is the end offset of the entire data minus the starting offset of Count
	* The value of Count is the number of key-value pairs
	* N represents the number of key-value pairs

<a name="map2">**Map2**</a>

The difference between map2 format and Map1 is that the value and key are stored separately, and it has a special routing format to quickly locate the value offset.  

Map2 consists of three parts: **header information, routing information and value**.  

The header information includes data length, metadata length, number of key-value pairs, and maximum depth. The routing information mainly stores the relative offset of the value corresponding to each Key in the entire data, and the value part stores the sequence of values ​​in turn.  

In the following, the header information and routing information are collectively referred to as the metadata part.

	+--------+*-------*+*-------*+*-------*+*--------*+~~~~~~~~~~~~~~~~~~+~~~~~~~~~~~~~~~~~~+
    |  0xc2  | DataLen |  Count  |  Depth  | RouteLen |   RouteSegment   |   ValueSegment   |
    +--------+*-------*+*-------*+*-------*+*--------*+~~~~~~~~~~~~~~~~~~+~~~~~~~~~~~~~~~~~~+

	* The value of DataLen is the position where the entire data ends minus the starting offset position of RouteLen
	* The value of Count is the number of key-value pairs
	* The value of Depth is the maximum depth in RouteSegment
	* The value of RouteLen is the end position of the entire data minus the start offset position of RouteSegment
	* ValueSegment is a sequence of values ​​in key-value pairs

**The following will mainly introduce the routing segment format**:

To understand the route segment format of Map2, you need to know what a [binary search algorithm](https://en.wikipedia.org/wiki/Binary_search_algorithm) is, the metadata of Map2 is derived from the idea of ​​a binary search algorithm.

There is an array here, you need to design an algorithm to quickly find the subscript of the element in the array.

    [120,3,1,125,5]

The most primitive method is to linearly traverse the array directly, so that the complexity of the algorithm is o(n). If you can sort the array before searching, and then fold it in half each time to search by interval, the complexity will be Is reduced to logn, such an algorithm is called a binary search algorithm.

Therefore, we can write BinarySearch as follows: 

	function BinarySearch(int[] array,int index,int length){
	  int lo = index;
      int hi = index + length - 1;
      while (lo <= hi)
      {
          int i = lo + ((hi - lo) >> 1);
          int order = array[i] - value;
          if (order == 0) return i;
          if (order < 0)
              lo = i + 1;
          else
              hi = i - 1;
      }
      return ~lo;
	}

In modern compiler optimizations, switch lookups for constants are often optimized to binary search branch code in some cases.If the elements of the array are always the same, we can always find them using the following specific lookup code:
	
	function compiled_BinarySearch(int input)
	{
		if(input<=3)
		{
			if(input == 1)
				return;
			else if(input == 3)
				return;
		}
		else
		{
			if(input == 5)
				return;
			else if(input==120)
				return;
			else if(input==125)
				return;
		}
	}
	
This code is an expanded form of the above BinarySearch, and the search speed is faster.  
What the routing section of Map2 describes is what the above code represents. The corresponding Map2 routing format of the above code is as follows:

	[0]LessThen1 NextOff(3) KeyBytes(3) 
		[1]EqualNext1 NextOff(2) KeyBytes(1) KeyType ValOffset NoChildren 
		[2]EqualLast1 KeyBytes(3) KeyType ValOffset NoChildren 
	[3]LessElse 
		[4]EqualNext1 NextOff(5) KeyBytes(5) KeyType ValOffset NoChildren 
		[5]EqualNext1 NextOff(6) KeyBytes(120) KeyType ValOffset NoChildren 
		[6]EqualLast1 KeyBytes(125) KeyType ValOffset NoChildren

This is a Token expression format used to assist in observing the routing segment, which basically corresponds to the above code.  

- **LessThen** : represents an if branch, which is a binary expression that judges the content to be less than or equal to, used to calculate the value at the bottom of the current stack and the next value.  
- **NextOff** : represents the relative offset based on the starting position of the metadata. If the binary calculation is not satisfied, you can jump to the next branch by the offset, and NextOff of LessThen is used to jump to the LessElse branch.
- **EqualNext** : represents an if branch, which is a binary expression that judges that the content is equal. It is used to calculate the equality between the value of the current stack bottom and the next KeyBytes. If it is not equal, jump to the next branch through the next NextOff. EqualNext's NextOff is used to jump to the EqualLast branch.
- **EqualLast** : represents an if branch, which is different from EqualNext in that it means that the branch is the last branch in the current subject
- **KeyBytes** : represents the type data of the Key
- **KeyType** : represents the type of Key
- **ValOffset** : represents the relative offset of the value based on the starting location of the metadata
- **NoChildren** : means that the current branch has no child branches


In bssom, when the width of the key exceeds 8 bytes, it is necessary to split the key by 8 bytes during marshalling (this is because modern computers can usually compare 64 bits at a time),  When unmarshalling, you need to restore the values ​​obtained in the process from top to bottom. 

Below, I will show a complex format. Because the width of the Keys demonstrated exceeds 8, it has levels and depth, and it is more like a stack structure.

	RawValue:
		Map { 
			  {"a1234567b1":1},
			  {"a1234567":2},
			  {"c1234567d1":3},
			  {"p1":4},
			  {"e1234567r1234567":5}  
		 } 

	intermediate format:
		+ p1 : 4
		+ a1234567 : 2
			- b1 : 1
		+ c1234567 
			- d1 : 3
		+ e1234567
			- r1234567 : 5

	final:
		[1]LessThen8 NextOff(5) KeyU64(3978425819141910881) 
			[2]EqualNext2 NextOff(3) KeyBytes(112,49) KeyType(StringCode) ValOffset NoChildren 
			[3]EqualLast8 KeyU64(3978425819141910881) KeyType(StringCode) ValOffset HasChildren 
				[4]EqualLast2 KeyBytes(98,49) KeyType(StringCode) ValOffset NoChildren 
		[5]LessElse 
			[6]EqualNextN NextOff(8) KeyU64(3978425819141910883) 
				[7]EqualLast2 KeyBytes(100,49) KeyType(StringCode) ValOffset NoChildren 
			[8]EqualLastN KeyU64(3978425819141910885) EqualLast8 KeyU64(3978425819141910898) KeyType(StringCode) ValOffset NoChildren
	

Map2 defines the following tokens in the routing section:

Token	 	  |  		description																			|  Type	|  Value
--------------|-------------------------------------------------------------------------------------|-------|-------  
LessThen{1-7} | An If branch for judging less than or equal to, the number of bytes stored in the judged body (KeyBytes) is 1~7 						| byte  |	21-27 
LessThen8 	  | An If branch for judging less than or equal to, the judged body (KeyBytes) stores little-endian UInt64						| byte  |	28	
LessElse      | An else branch, always corresponding to LessThen												    | byte  |	30
EqualNext{1-7}| An If branch used to judge whether they are equal, the number of bytes stored in the main body (KeyBytes) used to judge is 1~7, and the current branch has a Key   | byte  |	1-7
EqualNext8	  | An If branch for judging whether it is equal, the main body (KeyBytes) used for judging is stored as little-endian UInt64, and the current branch has a Key    | byte  |	8
EqualNextN	  | An If branch for judging whether it is equal, the main body (KeyBytes) used for judging is stored as little-endian UInt64, and the current branch does not exist Key		| byte   |	9
EqualLast{1-7}| An If branch used to judge whether they are equal, representing the last branch in the current body, the number of bytes stored in the body (KeyBytes) used for judgment is 1~7 , and the current branch has a Key  | byte | 11-17
EqualLast8	  | An If branch for judging whether it is equal, representing the last branch in the current body, the body used for judgment (KeyBytes) stores little-endian UInt64, and the current branch has a Key  | byte | 18
EqualLastN	  | An If branch that judges whether it is equal, represents the last branch in the current main body, the KeyBytes main body used for judgment is stored as little-endian UInt64, and the current branch does not exist Key  | byte| 19
NoChildren	  | Represents the current branch has no sub-branch | byte | 32
NextOff  	  | Represents the relative offset in the data of the next branch in the current logic | VarUInt | 
KeyBytes 	  | Represents the type data of the stored Key | Bianry/UInt64 | 
ValOffset	  | Represents the relative offset of the Value corresponding to the Key in the data | VarUInt |
KeyType  	  | Represents the [type](#supplement) of Key  	|   |  

Annotation:
- **Binary**: A sequence of 1~7 bytes
- **UInt64**: Little-endian unsigned 64-bit integer, because it is an integer type, the little-endian format is used to avoid the difference in reading on different platforms
- **VarUInt**: Bssom internal defined dynamic uint type.
- **KeyType**: See type system
- **KeyBytes** : In a system with a word length of 64 bits, the cost of judging whether one byte and eight bytes are equal is basically the same. Therefore, Bssom splits the width of the Key of Map2 into 8 bytes. If the width of the Key is greater than 8 Bytes, the depth will be generated in the current branch, and the remaining bytes need to be stored by the sub-branch.
- **Relative offset**: Start from the beginning of the DataLen field


The following paragraphs describe the semantics of routing segments:

	Branch::=
		->	equalNext(1-8) + nextBranchOff + keyBytes + keyType + valoffset -> nonChildren
		                                                                       -> hasChildern + [Branch]
		->	equalNextN + nextBranchOff + keyBytes + [Branch]
			
		->	equalLast(1-8) + keyBytes + keyType + valoffset -> nonChildren
		                                                       -> hasChildern + [Branch]
		->	equalLastN + keyBytes + [Branch]
		
		->	LessThen + [Branch] + LessElse + [Branch]




### Native Type format

The native type stores a VarUInt representing the size of the data

    +--------+*------*+========+
    |  0xf2  | Length |  Data  |
    +--------+*------*+========+
	
	* Length represents the size of Data
	* Data represents the native type of data
	

### Extend Type format

The extension type stores a type code used to represent the extension type and the data of the extension type (currently there is no defined extension type)

    +--------+----------+========+
    |  0xf1  | TypeCode |  Data  |
    +--------+----------+========+

	* TypeCode represents the type code of the extended type
	* Data represents the data of the extended type
  

## Implementation

.NET platform:  
  1. [**Bssom.Net**](https://github.com/1996v/Bssom.Net)

