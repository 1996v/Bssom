# Bssom 规范

**Bssom**(**Binary search algorithm structure model object binary marshalling**)是一个使用二分查找算法模型对对象进行结构化编组的协议，被编组后的数据具有特殊的**元数据**信息，根据这些元数据信息可以高效的**仅读取和更改**对象中的某个元素，这样可以在对大对象进行序列化和反序列化的过程中不必因为只读取或只写入一个字段而造成完整的序列化开销。

Bssom区别于其它二进制协议在于他有两个特征:  
1. <a name="characteristic1"></a>可以读取对象中的某个元素而不用完整的反序列化对象，这要求Bssom在设计时需要让被设计的类型能够拥有一种特殊的元数据格式，进而可以通过元数据定位到指定的元素的位置来读取它。
2. <a name="characteristic2"></a>可以更改对象中的某个元素而不用完整的序列化整个对象,这要求Bssom的某些类型被设计成定长结构，其格式不会因为值的改变而改变。

基于这两个特征，Bssom定义了**类型系统**和**格式**这两个概念.

序列化是通过Bssom类型系统从应用程序对象转换为Bssom格式,反序列化是通过Bssom类型系统从Bssom格式转换为应用程序对象.

本文档介绍了Bssom类型系统及Bssom格式

## 目录

* Bssom 规范
  * [类型系统](#类型系统)
      * [Native type](#native-type)
      * [Extension type](#extension-type)
      * [Blank type](#blank-type)
      * [补充](#补充)
  * [格式](#格式)
      * [概述](#概述)
      * [特殊的](#特殊的)
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
  * [实现](#实现)
  
## 类型系统
Bssom类型系统定义了11个种类
  * **Null** 表示空
  * **Number** 整数类型,是对8个整数格式的归纳,包含Int8,Int16,Int32,Int64,UInt8,UInt16,UInt32,UInt64
  * **Boolean** 表示 true 或 false
  * **Float** 表示[IEEE754](https://en.wikipedia.org/wiki/IEEE_754)标准中所定义的浮点类型,包含Float32,Float64  
  * **String** 表示UTF-8格式的字符串
  * **Timestamp** 表示[Unix时间戳](https://en.wikipedia.org/wiki/Unix_time)
  * **Extension** 保证了Bssom在未来拓展的可能性，应用程序应尽可能的对其支持，其含义由Bssom规范决定
  * **Native** 用于应用程序自己创建的本地类型格式，其实现不具备通用性和交互性，其含义由应用程序自己决定
  * **Array** 表示一组元素的序列,包含了三种序列格式 Array1,Array2,Array3  
  	* *Array1*: 有约束的格式,该序列中的元素都是同一个类型格式,元素类型被单独记录在头部信息中,元素本身将不再保留类型信息,且序列中的每个元素的宽度都是相等的
	* *Array2*: 没有约束的格式,每个元素正常拥有自己的类型信息,该序列中的元素可以是任意格式
	* *Array3*: 格式包含元素的偏移量信息和元素数据,通过数组下标定位到偏移量后直接访问和写入元素,适合需要快速访问和读取非定长元素的结构
  * **Map** 表示键值对,包含了两种键值对格式 Map1,Map2
	* *Map1*: 格式不包含结构化的元数据信息,编组时不需要预处理,适合快速写入和读取
	* *Map2*: 格式包含了结构化的元数据信息,将适用于Key的二分查找算法的查找结构给编组成用来路由定位的元数据,通过自动机来进行查找,适合高速的不完整写和不完整读
  * **Blank** 表示空白字节,用于填充由更改元素值导致的差槽

### Native type:
Bssom允许应用程序使用本地类型定义特定于应用程序的类型,所定义的本地类型格式为一个整数和数据组成,整数代表了该本地类型的数据长度,当进行跨程序交互时,应用程序可以自由选择是否处理未知的本地类型,如果选择忽略,则可以通过读取该整数来对其的数据进行跳过.

### Extension type:
为了保证Bssom协议的类型扩展,设计了扩展类型,扩展类型是由扩展类型码和数据组成,应用程序应尽可能的对其支持,但目前并没有已存在的扩展的类型

### Blank type:
当更改Bssom格式对象或其的某个元素时,若更改后的值的宽度小于被更改的对象,则会产生空隙,此空隙需要用填充符来补充,Blank类型就是用于在此处填充空白字节

### 补充：
* 在Bssom中,为了支持[第二点](#characteristic2)特征,Number,Float,Boolean的类型都被设计成定长的格式,这样,值的表现始终与其在二进制中的长度无关.
* 类型系统总体分为三部分: 1.非本地非扩展的原生类型部分, 2.本地类型部分, 3.扩展类型部分 , 在Map2表达KeyType和在Array1中表达ElementType时,如果是原生类型,则格式为一个字节的类型码,如果是本地类型则格式为代表本地类型的类型码0xf2+本地类型大小,如果是扩展类型则格式为代表扩展类型的类型码0xf1+扩展的类型码


## 格式

### 概述

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

### 特殊的:
在Bssom中有一个变长的无符号整数格式,**VarUInt**,该格式不在被定义的类型系统中,仅在Bssom格式的内部中使用,如 记录Array,String,Map的长度和元素数量
VarUInt format  | first byte  |  range
----------------| ----------- | -----------------
VariableUInt8   | 0x00~0xfa   |  0~250
VariableUInt9   | 0xfb	      |  251~505
FixUInt8	    | 0xfc        |  0~255
FixUInt16	    | 0xfd		  |  0~65535
FixUInt32	    | 0xfe		  |  0~4294967295
FixUInt64	    | 0xff		  |  0~18446744073709551615
		
**接下来将具体展示每个格式**
----------

### 符号表示

    一个字节:
    +--------+
    |        |
    +--------+

    一组字节:
    +========+
    |        |
    +========+

    在格式中定义的对象:
    +~~~~~~~~~~~~~~~~~+
    |                 |
    +~~~~~~~~~~~~~~~~~+
	

### Internal VarUInt Type format

VarUInt有六种格式,用于在Bssom格式内部表示对象的长度

    1.VariableUInt8 使用一个byte,存储了8位正整数,最小值为0x00,最大值为0xfa
    +--------+   +--------+
    |  0x00  | ~ |  0xfa  |
    +--------+   +--------+

    2.VariableUInt9 使用两个byte,第二个byte存储了8位正整数,最小值为0xfb+0x00,最大值为0xfa+0xff
    +--------+--------+   +--------+--------+
    |  0xfb  |  0x00  | ~ |  0xfb  |  0xff  |
    +--------+--------+   +--------+--------+

    3.FixUInt8 使用两个byte,第二个byte存储了8位正整数
    +--------+--------+
    |  0xfc  |XXXXXXXX|
    +--------+--------+
	
	4.FixUInt16 使用了三个byte,从第二个byte开始存储了小端序的16位正整数
    +--------+--------+--------+
    |  0xfd  |XXXXXXXX|XXXXXXXX|
    +--------+--------+--------+
	
	5.FixUInt32 使用了五个byte,从第二个byte开始存储了小端序的32位正整数
    +--------+--------+--------+--------+--------+
    |  0xfe  |XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|
    +--------+--------+--------+--------+--------+
	
	6.FixUInt64 使用了九个byte,从第二个byte开始存储了小端序的64位正整数
    +--------+--------+--------+--------+--------+--------+--------+--------+--------+
    |  0xff  |XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|
    +--------+--------+--------+--------+--------+--------+--------+--------+--------+

**注: 在接下来的格式中,VarUInt用如下符号来表示**

	VarUInt:
    +*------*+
    |        |
    +*------*+

### Blank Type format

空白类型有三种格式.

	1.variable blank ,使用一个byte存储7位正整数,用于表示接下来的最大长度为0x7f的空白字节
    +--------+=========+
    |0XXXXXXX| N*blank |
    +--------+=========+
	
	2.UInt16 Blank ,使用了两个byte存储16位正整数,用于表示接下来的最大长度为2^16-1的空白字节
	+--------+--------+--------+=========+
    |  0x80	 |XXXXXXXX|XXXXXXXX| N*blank |
    +--------+--------+--------+=========+
	
	3.UInt32 Blank ,使用了四个byte存储32位正整数,用于表示接下来的最大长度为2^32-1的空白字节
	+--------+--------+--------+--------+--------+=========+
    |  0x81	 |XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX| N*blank |
    +--------+--------+--------+--------+--------+=========+

### Null Type format

Null类型只有一种格式.

    null:
    +--------+
    |  0x82  |
    +--------+

### Boolean Type format

Boolean类型拥有两种格式,分别代表false和true.

    1.false:
    +--------+--------+
    |  0x8d  |  0x00  |
    +--------+--------+

    2.true:
    +--------+--------+
    |  0x8d  |  0x01  |
    +--------+--------+

### Number Type format 

Number类型包含8种格式Int8,Int16,Int32,Int64,UInt8,UInt16,UInt32,UInt64

    Int8存储了一个8位的有符号整数
    +--------+--------+
    |  0x83  |XXXXXXXX|
    +--------+--------+

    Int16存储了一个16位的小端序的有符号整数
    +--------+--------+--------+
    |  0x84  |XXXXXXXX|XXXXXXXX|
    +--------+--------+--------+

    Int32存储了一个32位的小端序的有符号整数
    +--------+--------+--------+--------+--------+
    |  0x85  |XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|
    +--------+--------+--------+--------+--------+

    Int64存储了一个32位的小端序的有符号整数
    +--------+--------+--------+--------+--------+--------+--------+--------+--------+
    |  0x86  |XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|
    +--------+--------+--------+--------+--------+--------+--------+--------+--------+

    UInt8存储了一个8位的无符号整数
    +--------+--------+
    |  0x87  |XXXXXXXX|
    +--------+--------+

    UInt16存储了一个16位的小端序的无符号整数
    +--------+--------+--------+
    |  0x88  |XXXXXXXX|XXXXXXXX|
    +--------+--------+--------+

    UInt32存储了一个32位的小端序的无符号整数
    +--------+--------+--------+--------+--------+
    |  0x89  |XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|
    +--------+--------+--------+--------+--------+

    UInt64存储了一个32位的小端序的无符号整数
    +--------+--------+--------+--------+--------+--------+--------+--------+--------+
    |  0x8a  |XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|
    +--------+--------+--------+--------+--------+--------+--------+--------+--------+

### Float Type format

Float类型包含两种格式,Float32,Float64

    float32以IEEE 754单精度浮点数32位格式存储一个浮点数:
    +--------+--------+--------+--------+--------+
    |  0x8b  |XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|
    +--------+--------+--------+--------+--------+

    float64以IEEE 754双精度浮点数64位格式存储一个浮点数:
    +--------+--------+--------+--------+--------+--------+--------+--------+--------+
    |  0x8c  |YYYYYYYY|YYYYYYYY|YYYYYYYY|YYYYYYYY|YYYYYYYY|YYYYYYYY|YYYYYYYY|YYYYYYYY|
    +--------+--------+--------+--------+--------+--------+--------+--------+--------+

    * 在存储时,浮点数以小端序来存储
    

### String Type format

字符串类型只有一种格式,组成为字节大小和Utf8编码的字节数组

    +--------+*-------*+========+
    |  0x8f  | DataLen |  Data  |
    +--------+*-------*+========+
	
	* Data是用Utf8编码对字符串进行编码后的字节数组
	* DataLen是被编码后的字节数组的长度
	
### Timestamp Type format

Timestamp类型只有一个格式,该格式是由8个字节表示的自Unix Epoch以来的秒数和4个字节的给定秒数内的增量.

    +--------+------------------------------+------------------------------------+
    |  0x8e  | seconds in 64-bit signed int | nanoseconds in 32-bit unsigned int |
    +--------+------------------------------+------------------------------------+

### Array Type format

数组类型拥有三种格式: Array1,Array2,Array3.

<a name="array1">**Array1**</a>  

Array1格式是由头部信息和元素序列组成,在头部信息中,包含了[元素类型](#补充),这意味着数组中的元素不需要存储类型头信息,因此数组元素在Array1中有两点要求: **1**.数组中的元素不能包含类型头, **2**.数组中的元素的宽度都是相同的.  

*因为Array1的元素都是定长的,因此获取元素和更改元素的访问方式可以通过: ElementsPos + index * eleSize 来获得*

	+--------+~~~~~~~~~~~~~~~+*------*+*-----*+~~~~~~~~~~~~~~~~~+
    |  0xd1	 |  ElementType  | Length | Count |    N*Element    |
    +--------+~~~~~~~~~~~~~~~+*------*+*-----*+~~~~~~~~~~~~~~~~~+

	* ElementsPos是指元素序列的起始位置(Count字段的末尾偏移量)
	* ElementType:参见类型系统
	* Length的值是整个数据结束的位置减去Count的起始偏移位置
	* Count的值是元素的数量
	* N代表元素的数量
	
<a name="array2">**Array2**</a>  

Array2格式是由长度,元素数量,和元素序列组成,与Array1不同,Array2对元素没有任何要求.  

*因为Array2中的元素没有任何限制,因此获取元素和更改元素的访问方式可以通过: ElementsPos + index * SkipElement 来获得*
	
	+--------+*------*+*-----*+~~~~~~~~~~~~~~~~~+
    |  0xd2	 | Length | Count |    N*Element    |
    +--------+*------*+*-----*+~~~~~~~~~~~~~~~~~+

	* ElementsPos是指元素序列的起始位置(Count字段的末尾偏移量)
	* Length的值是整个数据结束的位置减去Count的起始偏移位置
	* Count的值是元素的数量

<a name="array3">**Array3**</a>  
  
Array3格式是由长度,元素数量,元素偏移量序列,元素序列组成.

*因为Array3中拥有元素偏移量序列,因此获取元素和更改元素的访问方式可以通过: ElementsPos + GetElementOffset(index) 来获得*

	+--------+*------*+*-----*+~~~~~~~~~~~~~~~~~+~~~~~~~~~~~~~~~~~+
    |  0xd3	 | Length | Count |  ElementOffsets |    N*Element    |
    +--------+*------*+*-----*+~~~~~~~~~~~~~~~~~+~~~~~~~~~~~~~~~~~+

	ElementOffsets是由VarUInt格式组成的正整数序列,每个VarUInt对应此下标元素基于ElementsPos的偏移量:
	+*-------*+*-------*+*-------*+
    | Offset1 | Offset2 | Offset3 |
    +*-------*+*-------*+*-------*+	...

	* ElementsPos是指元素序列的起始位置(ElementOffsets段落的末尾偏移量)
	* Length的值是整个数据结束的位置减去Count的起始偏移位置
	* Count的值是元素的数量,也代表ElementOffsets中的格式数量


### Map Type format

Map类型有两种格式,Map1和Map2.

<a name="map1">**Map1**</a>

Map1格式是由头部信息和键值对序列组成,头部信息包含数据长度和键值对数量


    +--------+*-------*+*-----*+~~~~~~~~~~~~~~~~~~+
    |  0xc1  | DataLen | Count |  N * (Key,Value) |
    +--------+*-------*+*-----*+~~~~~~~~~~~~~~~~~~+

	* DataLen的值是整个数据结束的位置减去Count的起始偏移位置
	* Count的值是键值对的数量
	* N代表了键值对的数量

<a name="map2">**Map2**</a>

map2格式与Map1不同处在于值和键分开存储,且拥有特殊的路由格式用来快速定位值偏移量.  

Map2由头部信息和路由信息和值三部分组成.头部信息包含数据长度,元数据长度,键值对数量,最大深度.路由信息主要存储了每个Key对应的值在整个数据中的相对偏移量,值部分则是依次存储了值序列.在下文,头部信息和路由信息都统称为元数据部分.

	+--------+*-------*+*-------*+*-------*+*--------*+~~~~~~~~~~~~~~~~~~+~~~~~~~~~~~~~~~~~~+
    |  0xc2  | DataLen |  Count  |  Depth  | RouteLen |   RouteSegment   |   ValueSegment   |
    +--------+*-------*+*-------*+*-------*+*--------*+~~~~~~~~~~~~~~~~~~+~~~~~~~~~~~~~~~~~~+

	* DataLen的值是整个数据结束的位置减去RouteLen的起始偏移位置
	* Count的值是键值对的数量
	* Depth的值是RouteSegment中的最大深度
	* RouteLen的值是整个数据结束的位置减去RouteSegment的起始偏移位置
	* ValueSegment是键值对中的值的序列

**下文将主要介绍路由段格式**:

要想了解Map2的路由段格式,得先了解什么是[二分算法](https://en.wikipedia.org/wiki/Binary_search_algorithm).Map2的元数据就是通过二分算法思想来衍生出来的.

这里有一个数组,你需要设计一个算法,以用来快速查找数组中的元素所在的下标.

    [120,3,1,125,5]

最原始的方法是直接线性遍历该数组,这样,该算法的复杂度则为o(n).如果你能在搜索前先将数组进行排序,然后每次折半来按区间查找,这样复杂度将被降低到logn,这样的算法就被叫做二分查找算法.

因此,我们可以编写BinarySearch,如下所示

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

在现代编译器的优化中,通常会对常量的switch查找在某种情况下给优化成二分查找分支代码.如果该数组的元素始终不变的话,那么我们始终能够通过如下的特定的查找代码来查找:
	
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
	
此代码是上述BinarySearch的展开形式,查找的速度更快.  
Map2的路由段所描述的就是上面这段代码所表述的内容,上面的代码对应的格式如下:

	[0]LessThen1 NextOff(3) KeyBytes(3) 
		[1]EqualNext1 NextOff(2) KeyBytes(1) KeyType ValOffset NoChildren 
		[2]EqualLast1 KeyBytes(3) KeyType ValOffset NoChildren 
	[3]LessElse 
		[4]EqualNext1 NextOff(5) KeyBytes(5) KeyType ValOffset NoChildren 
		[5]EqualNext1 NextOff(6) KeyBytes(120) KeyType ValOffset NoChildren 
		[6]EqualLast1 KeyBytes(125) KeyType ValOffset NoChildren

这是一段用于辅助观察路由段的Token表述格式,与上述代码基本相照应.  

- **LessThen**代表了一个if分支,判断内容为小于等于的二元表达式,用于将当前栈底的值与接下来的KeyBytes进行小于等于计算.  
- **NextOff**代表了基于元数据起始位置的相对偏移量,若二元计算不满足,则可以通过偏移量跳跃到下一个分支, LessThen的NextOff用于跳转到LessElse分支.
- **EqualNext**代表了一个if分支,判断内容为相等二元表达式,用于将当前栈底的值与接下来的KeyBytes做相等计算,若不等则通过接下来的NextOff跳转到下一个分支,EqualNext的NextOff用于跳转到EqualLast分支.
- **EqualLast**代表了一个if分支,与EqualNext区别在于,它意味着该分支是当前主体中的最后分支
- **KeyBytes**代表Key的类型数据
- **KeyType**代表了Key的类型
- **ValOffset**代表了值基于元数据起始位置的相对偏移量
- **NoChildren**意味着当前分支没有子分支


在Bssom中,当Key的宽度超出8个字节后,在编组时需要对Key进行以8字节为单位的拆分(这是因为现代计算机中通常可以一次比较64位),在解组的时候,需要对过程中所获取的值自上而下的进行还原.在这段格式中并不能完全体现出Map2路由段的全部组成,因为Int类型的Key始终有4个字节的宽度. 

下面,我将展示一个复杂的格式,该格式因为所演示的Keys中有的宽度超出8个,所以有了层级和深度,它更像一个堆栈结构

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
	

Map2在路由段中共定义了如下Token:

Token	 	  |  		含义																			|  格式	|  值
--------------|-------------------------------------------------------------------------------------|-------|-------  
LessThen{1-7} | 一个判断小于等于的If分支,用于判断的KeyBytes主体存储的字节数量为{1-7} 						| byte  |	21-27 
LessThen8 	  | 一个判断小于等于的If分支,用于判断的KeyBytes主体存储的是小端的UInt64  						| byte  |	28	
LessElse      | 一个else分支,始终与LessThen相对应													    | byte  |	30
EqualNext{1-7}| 一个判断是否相等的If分支,用于判断的KeyBytes主体存储的字节数量为{1-7},且当前分支拥有一个Key   | byte  |	1-7
EqualNext8	  | 一个判断是否相等的If分支,用于判断的KeyBytes主体存储的为小端的UInt64,且当前分支拥有一个Key    | byte  |	8
EqualNextN	  | 一个判断是否相等的If分支,用于判断的KeyBytes主体存储的为小端的UInt64,当前分支不存在Key		| byte   |	9
EqualLast{1-7}| 一个判断是否相等的If分支,代表当前主体中的最后分支,用于判断的KeyBytes主体存储的字节数量为{1-7},且当前分支拥有一个Key  | byte | 11-17
EqualLast8	  | 一个判断是否相等的If分支,代表当前主体中的最后分支,用于判断的KeyBytes主体存储的为小端的UInt64,且当前分支拥有一个Key   | byte | 18
EqualLastN	  | 一个判断是否相等的If分支,代表当前主体中的最后分支,用于判断的KeyBytes主体存储的为小端的UInt64,当前分支不存在Key  | byte| 19
NoChildren	  | 代表了当前分支没有子分支 | byte | 32
NextOff  	  | 代表了当前逻辑中下一个分支的在数据中的相对偏移量 | VarUInt | 
KeyBytes 	  | 代表了存储的Key的类型数据 | Bianry/UInt64 | 
ValOffset	  | 代表了Key对应的Value在数据中的相对偏移量 | VarUInt |
KeyType  	  | 代表了Key的[类型](#补充)  	|   |  

注解:
- **Binary**: 一串1-7个字节序列
- **UInt64**: 小端序的无符号64位整数,因为是整数类型,所以采用小端格式来避免不同平台读取的不同表现
- **VarInt**: Bssom内部的定义的动长类型.
- **KeyType**: 参见类型系统
- **KeyBytes** : 在字长为64位的系统中,判断一个字节和八个字节是否相等的开销基本一致,因此Bssom将Map2的Key的宽度按照8个字节进行拆分,若Key的宽度大于8个字节,则在当前分支中将产生深度,需要由子分支来储存剩余的字节.
- **相对偏移量**:从DataLen字段的起始位置开始


如下段落描述了路由段的语义:

	Branch::=
		->	equalNext(1-8) + nextBranchOff + keyBytes + keyType + valoffset -> nonChildren
		                                                                       -> hasChildern + [Branch]
		->	equalNextN + nextBranchOff + keyBytes + [Branch]
			
		->	equalLast(1-8) + keyBytes + keyType + valoffset -> nonChildren
		                                                       -> hasChildern + [Branch]
		->	equalLastN + keyBytes + [Branch]
		
		->	LessThen + [Branch] + LessElse + [Branch]




### Native Type format

本地类型存储一个表示数据大小的VarInt

    +--------+*------*+========+
    |  0xf2  | Length |  Data  |
    +--------+*------*+========+
	
	* Length代表了Data的大小
	* Data代表了该本地类型的数据
	

### Extend Type format

扩展类型存储一个用于表示扩展类型的类型码和该扩展类型的数据(目前没有已定义的扩展类型)

    +--------+----------+========+
    |  0xf1  | TypeCode |  Data  |
    +--------+----------+========+

	* TypeCode代表扩展类型的类型码
	* Data代表了该扩展类型的数据
  

## 实现

.NET平台:  
  1. [**Bssom.Net**](https://github.com/1996v/Bssom.Net)

