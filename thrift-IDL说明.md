# Thrift IDL

标签（空格分隔）： Thrift

---
Thrift interface definition language（IDL）可以用来定义Thrift Types。一个IDL文件可以通过Thrift代码生成器用来产生不同语言的代码从而支持IDL文件中声明的结构体和服务。

## thrift IDL

 一. Document

每个thrift idl包含0个或者更多地headers，headers后面包含0个或者更多地定义。
```
[1]  Document        ::=  Header* Definition*
```
 二. Header

每个header要么是一个Thrift include，要么是一个C++ include，或者是一个namespace声明。
```
[2]  Header          ::=  Include | CppInclude | Namespace
```
1. Thrift Include
thrift include可以使得另一个thrift文件中的符号可以在当前文件中可用（使用一个前缀），并且把对应的include声明语句加入到当前thrift文件生成的代码中。
```
[3]  Include         ::=  'include' Literal
```
2. C++ Include
C++ Include把特定的C++包含到C++代码输出里面。
```
[4]  CppInclude      ::=  'cpp_include' Literal
```
3. Namespace
命名空间申明了namespaces/package/module/etc，“*”表示namespace适用所有目标语言。
```
[5]  Namespace       ::=  ( 'namespace' ( NamespaceScope Identifier ) |
                                        ( 'smalltalk.category' STIdentifier ) |
                                        ( 'smalltalk.prefix' Identifier ) ) |
                          ( 'php_namespace' Literal ) |
                          ( 'xsd_namespace' Literal )

[6]  NamespaceScope  ::=  '*' | 'cpp' | 'java' | 'py' | 'perl' | 'rb' | 'cocoa' | 'csharp'
```
举个栗子：
```thrift
namespace java com.ffy.thrift.test
namespace cpp com.ffy.thrift.test
//namespace * com.ffy.thrift.test
```
三. Definition
```
[7]  Definition      ::=  Const | Typedef | Enum | Senum | Struct | Union | Exception | Service
```
1. Const
```
[8]  Const           ::=  'const' FieldType Identifier '=' ConstValue ListSeparator?
```
例如：
```thrift
const i32 DEFAULT_ID = 32
```
2. Typedef
Typedef用于给某个类型创建一个别名
```
[9]  Typedef         ::=  'typedef' DefinitionType Identifier
```
例如：
```
typedef i32 MyInteger
```
3. Enum
枚举类型，包含名称和值；如果常量的值没有声明，则要么是0（第一个位置），要么是一个比前面枚举值大的值。任何常量值都必须是非负数。
```
[10] Enum            ::=  'enum' Identifier '{' (Identifier ('=' IntConstant)? ListSeparator?)* '}'
```
例如：
```thrift
enum StatusType{
    SUCCESS = 1 
    FAILURE = 2
}
```
4. Senum
已废弃，统一使用String代替
5. Struct
struct是Thrift中的基础组合类型。struct中的每个field的名称必须唯一。
```
[12] Struct          ::=  'struct' Identifier 'xsd_all'? '{' Field* '}'
```
备注：xsd_all关键词在Facebook内部有一些特殊意义，但thrift本身没有，强烈建议大家不用用这个关键词。
栗子来了：
```thrift
struct Book{
    1:required i32 id;
    2:required string sn;
    3:required float64 price;
    4:optional string writer;
    5:i32 year;
}
```
6. Union
Union和struct类似，除了他们提供一种方式在传输的时候只传输多个field中的一个field，和C++中的union{}类似。简单的说，unio的成员隐式的定义为optional。
```
[13] Union          ::=  'union' Identifier 'xsd_all'? '{' Field* '}'
```
7. Exception
Exceptions 和Structs类似，除了他主要是用了对接目标语言的异常处理逻辑。exception中的每个field都必须是唯一的。
```thrift
[14] Exception       ::=  'exception' Identifier '{' Field* '}'
```
8. Service
service提供了Thrift server对外暴露的interface。而interface这是简单的function列表。一个service可以继承另一个service，表示它除了提供它自身特点的function外，还扩展了被继承service的functions。
```thrift
[15] Service         ::=  'service' Identifier ( 'extends' Identifier )? '{' Function* '}'
```
四. Field
```thrift
[16] Field           ::=  FieldID? FieldReq? FieldType Identifier ('= ConstValue)? XsdFieldOptions ListSeparator?
```
1. Field ID
```thrift
[17] FieldID         ::=  IntConstant ':'
```
2. Field Requiredness
field的requiredness具有两个显示值，以及如果required和optional都没有显示指定，设置一个default requiredness。
```thrift
[18] FieldReq        ::=  'required' | 'optional'
```
requiredness 规则如下：
+ required
  - Write：Required fields总是被写入并且期望被设置。
  - Read：Required fields总是可读并且期望被包含在在输入流中。
  - Default values：总是写入

如果一个required field在读的过程中缺失，期望的行为是反馈给调用者读取操作不成功，例如：抛出一个exception或者返回一个error。
正因为这个约定存在，required fields彻底地限定了soft versioning的极限。读取的时候field必须存在，所以该field永远不能被设置为过期。如果一个required field被移除，或者修改为optional，不同版本之间的数据就不再兼容。
+ optional
  - Write: Optional field 只有在他们被设置了之后才会写入。
  - Read：Optional field 可以是，也可以不是输入流的一部分。
  - Default values：只有isset flag被设置之后才会写入。

大部分语言都采用一种被称为“isset” flag的推荐方案，来表名一个特定的optional field是否被设置。只有该标记被设置，field才会被写入；相反的，只有field value从输入流中读取到，改标记才会被设置。

+ default requiredness(implicit)
   - Write: 理论上，field总是被写入，当然有一些例外，见下文。
   - Read: 和optional一样，field可以是，也可以不是输入流的一部分。
   - Default values: 可以不被写入（见下一章）

Default requiredness 是一个不错的起点。期望的行为是optional和required的综合，因此也被称为“opt-in,req-out”。尽管理论上这些field期望的是被写入（"req-out"），现实中unset的fields总是不被写入。特别是一些特殊情况下，一些field的值是无法通过thrift传输的。达到这种效果的唯一途径是根本就不去写改field，而这也是绝大多数语言所做的。
+ Default Values的语义
> There are ongoing discussions about that topic, see JIRA for details. Not all implementations treat default values in the very same way, but the current status quo is more or less that default fields are typically set at initialization time. Therefore, a value that equals the default may not be written, because the read end will set the value implicitly. On the other hand, an implementation is free to write the default value anyways, as there is no hard restriction that prevents this.
> The major point to keep in mind here is the fact, that any unwritten default value implicitly becomes part of the interface version. If that default is changed, the interface changes. If, in contrast, the default value is written into the output data, the default in the IDL can change at any time without affecting serialized data.

3. XSD Options
N.B.: These have some internal purpose at Facebook but serve no current purpose in Thrift. Use of these options is strongly discouraged.
```thrift
[19] XsdFieldOptions ::=  'xsd_optional'? 'xsd_nillable'? XsdAttrs?
[20] XsdAttrs        ::=  'xsd_attrs' '{' Field* '}'
```
五. Functions
```thrift
[21] Function        ::=  'oneway'? FunctionType Identifier '(' Field* ')' Throws? ListSeparator?

[22] FunctionType    ::=  FieldType | 'void'

[23] Throws          ::=  'throws' '(' Field* ')'
```
六. Types
``` thrift
[24] FieldType       ::=  Identifier | BaseType | ContainerType

[25] DefinitionType  ::=  BaseType | ContainerType

[26] BaseType        ::=  'bool' | 'byte' | 'i8' | 'i16' | 'i32' | 'i64' | 'double' | 'string' | 'binary' | 'slist'

[27] ContainerType   ::=  MapType | SetType | ListType

[28] MapType         ::=  'map' CppType? '<' FieldType ',' FieldType '>'

[29] SetType         ::=  'set' CppType? '<' FieldType '>'

[30] ListType        ::=  'list' '<' FieldType '>' CppType?

[31] CppType         ::=  'cpp_type' Literal
```
七. Constant Values
```thrift
[32] ConstValue      ::=  IntConstant | DoubleConstant | Literal | Identifier | ConstList | ConstMap

[33] IntConstant     ::=  ('+' | '-')? Digit+

[34] DoubleConstant  ::=  ('+' | '-')? Digit* ('.' Digit+)? ( ('E' | 'e') IntConstant )?

[35] ConstList       ::=  '[' (ConstValue ListSeparator?)* ']'

[36] ConstMap        ::=  '{' (ConstValue ':' ConstValue ListSeparator?)* '}'
```
八. Basic Definitions
1. Literal
```thrift
[37] Literal         ::=  ('"' [^"]* '"') | ("'" [^']* "'")
```
2. Identifier
```thrift
[38] Identifier      ::=  ( Letter | '_' ) ( Letter | Digit | '.' | '_' )*

[39] STIdentifier    ::=  ( Letter | '_' ) ( Letter | Digit | '.' | '_' | '-' )*
```
3. List Separator
```thrift
[40] ListSeparator   ::=  ',' | ';'
```
4. Letters and Digits
```thrift
[41] Letter          ::=  ['A'-'Z'] | ['a'-'z']

[42] Digit           ::=  ['0'-'9']
```







