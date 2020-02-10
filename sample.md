---
author: '黄某某'
title: 'EDA数据转换标准化探索'
date: '2018-03-18'
...

摘要 {-}
====

Liberty格式可以描述电路单元的特性，是EDA流程中必不可少的内容。它已经作为开放的工业标准，被使用了超过20年。

json是一种简单灵活的轻量级数据交换格式，它独立于语言，并具有易于人阅读和编写的优点。json schema是基于json格式定义的用于规范json文件内容的规范。根据schema文件，可以检查目标json文件是否符合该文件所描述的内容规范。

Liberty虽是目前的标准，但它由于发明时间早，经过多次修改，格式上存在不合理之处。如果使用新格式指定新标准，将对EDA软件编写带来更多的方便。

本文研究了Liberty格式标准，设计实现了通用的将Liberty转成json的程序；并针对实际的情况，介绍json schema的应用。

**关键词：**Liberty，json，json schema，格式转换。

Abstract {-}
========

Liberty format can describe characteristics of circuit units, which is indespensable in the EDA flow. It has been an open industry standard for more than 20 years.

json is a simple, lightweight data-interchange format. It is language independent, easy for humans to read and write and easy for machines to pars and generate[1]. json schema is a json-based format to define the structure of json data for validation[6].

Liberty is current standard, but because is has been invented for many years. After several revisions, there are some loopholes in the format. Using a new format will bring convenience for EDA tool development.

This paper studies the Liberty format standard, designed and implemented a generic liberty-to-json converter, and introduce json schema in a particular application environment.

**Keywords:**Liberty, json, json schema, format converter.

第一章 绪论 {#sec:sec1}
==========

§ 1.1 研究意义和目的
-------------------

集成电路设计中，无论是定制电路设计，基于标准单元的半定制设计，还是基于FPGA的半定制设计等，基本单元的特性描述都是EDA流程中必不可少的信息。它使各个EDA工具能够得到基本单元的输入负载、延迟、功耗等信息。使得EDA流程能够准确地进行。

Liberty格式作为工业通用标准，从不同层次和类别表征了这些单元的这些信息，被广泛采用。它从1987年被发明至今，经历了多次修改。形成了内容繁多，形式复杂的情况。这将对EDA软件的开发造成一定程度的障碍。

综上所述，如何使用一种新的文件格式来表示Liberty库的内容是本文的主要研究内容。

§1.2 json和json schema简介
-------------------------

json是一种轻量级数据交换格式，全称JavaScript Object Notation。它是JavaScript的一个子集，但是它有独立于语言的特性。并且同时方便程序解析以及易于人阅读。

json schema是用json格式来书写的，用于定义json数据文件的结构和内容规范的规范。它用于描述和规范现有的数据格式，方便于由人编写，并方便于自动化测试验证目标json数据是否符合规范。

目前json在各个领域有很多应用，因此目前已经存在很多开源工具支持json的编写、解析、格式检查以及与其他数据格式的相互转换。因此使用json格式具有更灵活方便的优势。并且，通过json schema检查验证json数据规范也已经有完整方便的开源工具可以使用。因此对库文件内容的规范可以由一个文件来描述，这将方便EDA软件的开发，也方便EDA软件在库文件格式内容变化时修改扩展。

本文的主要工作是，提出使用json文件格式来表示Liberty库文件，并使用json schema验证其内容的思路和方法；并且设计实现了Liberty到json的转换器程序。

§1.3 论文结构
-------------

本文的内容安排如下：

第二章为背景知识介绍。首先介绍json的格式标准；参照json格式总结Liberty的格式标准；再介绍json schema。
第三章介绍Liberty到json转换器。包括需求分析、具体设计以及测试和分析。
第四章介绍使用json schema描述Liberty内容规范的方法。
第五章将对工作做出总结，指出工作存在的不足以及进一步研究的方向和思路。

第二章 基础知识介绍
==========================

§2.1 json格式介绍
----------------

###§2.1.1 json基本格式总览

[@lst:jsonExample]展示了json的基本格式。例子中可以看出json以两种基本结构组织起来：

* 对象。对象是一对花括号括起来的内容。其中可以包含多组键值对。其中键是名称，值是其内容。值可以为字符串、数字、对象、数组、布尔值（true, false）或空（null）。
* 数组。数组是一对方括号括起来的内容。其中包含一系列的值。值可以是不同类型。

```{#lst:jsonExample .json caption="json样例"}
{
    "key1" : "value1",
    "key2" : 3,
    "key3" : {
        "subkey3" : "value of key3 is an object"
    },
    "key4" : ["I'm", "an", "array"]
}
```


###§2.1.2 格式范式

json的格式范式如下：

**object:**

    object ::= '{' string ':' value ( ',' string ':' value )* '}'

![BNF_object](E:\库\documents\大四下\PJ\doc\output_sources\diagram_json\object.png)

**array:**

    array ::= '[' ( value ( ',' value )* )? ']'

![BNF_array](E:\库\documents\大四下\PJ\doc\output_sources\diagram_json\array.png)

**value:**

    value    ::= string
           | number
           | object
           | array
           | true
           | false
           | null

![BNF_value](E:\库\documents\大四下\PJ\doc\output_sources\diagram_json\value.png)

**string:**

    string ::= '"' ( 
                        (   ( "any UNICODE character except double_quote or back_slash or control character" )
                            |   
                            ( 
                                '\' ( '"' | '\' | '/' | 'b' | 'f' | 'n' | 'r' | 't' | ('u' "4 hexadecimal digits") )
                            ) 
                        )+
                    )? '"'

![BNF_string](E:\库\documents\大四下\PJ\doc\output_sources\diagram_json\string.png)

**number:**

    number ::= ('-')? ( (digit1-9 (digit)*) | '0' ) ( '.' (digit)+ )? ('e' | 'E') ('+' | '-')? (digit)+

![BNF_number](E:\库\documents\大四下\PJ\doc\output_sources\diagram_json\number.png)

其中：

**digit:**

    digit    ::= digit1-9
           | '0'

![BNF_digit](E:\库\documents\大四下\PJ\doc\output_sources\diagram_json\digit.png)

**digit1-9:**

    digit1-9 ::= '1'
           | '2'
           | '3'
           | '4'
           | '5'
           | '6'
           | '7'
           | '8'
           | '9'

![BNF_digit1-9](E:\库\documents\大四下\PJ\doc\output_sources\diagram_json\digit1-9.png)

§2.2 Liberty格式介绍
-------------------

###§2.2.1 基本格式总览

例展示了Liberty的基本格式。从[@lst:libExample]中可以看出其基本特征：

* 层次结构采用花括号组织
* 基本条目用分号结束
* 基本条目有冒号分隔和括号分隔两种情况




```{#lst:libExample .c caption="Liberty样例"}
library (name) {
    technology (name) ; /* library-level attributes */
	delay_model : generic_cmos | table_lookup | cmos2 |
	piecewise_cmos | dcm | polynomial ;
	bus_naming_style : string;
	routing_layers (string) ;
	time_unit : unit ;
	voltage_unit : unit ;
	current_unit : unit ;
	pulling_resistance_unit : unit ;
	capacitive_load_unit (value, unit) ;
	leakage_power_unit : unit ;
	piece_type : type ;
	piece_define ("list") ;
	define_cell_area (area_name, resource_type) ;
    /* default values for environment definitions */
	operating_conditions (name) {
    /* operating conditions */
	}
	timing_range (name) {
    /* timing information */
	}
	wire_load (name) {
    /* wire load information */
	}
	wire_load_selection () {
    /* area/group selections */
	}
	power_lut_template (name) {
    /* power lookup table template information */
	}
    cell (name1) { /* cell definitions */
    /* cell information */
    }
    cell (name2) {
    /* cell information */
    }
    scaled_cell (name1) {
    /* alternate scaled cell information */
    }
    ...
    type (name) {
    /* bus type name
    }
    input_voltage (name) {
    /* input voltage information */
    }
    output_voltage (name) {
    /* output voltage information */
    }
}
```

###§2.2.2 声明(Statement)

Liberty有三种声明，分别为：

* 组(Group)声明
* 属性(Attribute)声明
* 定义(Define)声明

声明可以跨行书写，但需要在跨行出以反斜杠(\\)标记。下面详述三种声明的格式

1. 组声明

组声明格式如下：

```c
group_type (name) {

... statements ...

}
```

其中：

* group_type为组的类别，可以是library, cell, pin, timing等
* name为组的名称，写于圆括号内
* 一对花括号标记组的范围，其中书写组的内容。

[@lst:libExampleGroup]中声明了pin组A：


```{#lst:libExampleGroup .c caption=}
pin(A) {

... pin group statements ...

}
```


2. 属性声明

属性声明为Liberty中的基本单位之一，包含在组中。其分为简单属性和复杂属性。

**简单属性**

简单属性声明格式如下：

```c
attribute_name : attribute_value ;
```

冒号前为属性的名称，冒号后为属性值，行尾是一个分号。Liberty要求冒号的前后各留一个空格符。且简单属性只能单行书写。

[@lst:libExampleSimple]中声明了组pin A中的简单属性direction，其值为output：

```{#lst:libExampleSimple .c caption=}
pin(A) {

direction : output ;

}
```

**复杂属性**

复杂属性声明格式如下：

```c
attribute_name (parameter1, [parameter2,
parameter3 ...] );
```

属性名称后跟一对圆括号，圆括号中为一个或以上参数构成的列表，行尾是一个分号。

[@lst:libExampleArray]中声明了名为line的复杂属性，其值为列表1,2,6,8：

```{#lst:libExampleArray .c caption=}
line (1, 2, 6, 8);
```

列表中每个元素的内容是逗号隔开的一系列值时，列表内容实际意义上是一个二维数组。如[@lst:libExampleArray2]所示

```{#lst:libExampleArray2 .c}
values ("0.00, 0.21", "0.11, 0.23") ;
```

3. 定义声明

定义声明功能上用于声明新的简单属性。其形式上与复杂声明相同。其格式如下：

```c
define (attribute_name, group_name, attribute_type) ;
```

[@lst:libExampleDefine]声明了一个定义声明，其定义了新的名为bork的简单属性。它只在pin组中合法，且它的形式为string。

```{#lst:libExampleDefine .c caption=}
define (bork, pin, string) ;
```

###§2.2.3 值的种类

首先定义值为简单属性冒号后的属性值以及复杂属性和定义声明中列表的一项。
则Liberty中值的种类有以下几种：

* 字符串
* 枚举
* 整数
* 浮点
* 空

需要说明的是，这里的枚举表现形式是字符串，但要求取值只能在枚举列表中选其一。因此这里的枚举包含了布尔(boolean)类型。在Liberty中，枚举类型一般不加双引号。但有双引号时仍然合法。

###§2.2.4 注释

Liberty支持C风格的注释。具体为一下两种：

* 单行注释。以两个斜杠(/)开始，结尾处是换行符(\n)。如[@lst:libExampleSComment]
* 多行注释。以斜杠加星号(/\*)开始，以星号加斜杠(\*/)结束。中间允许有换行符。如[@lst:libExampleMComment]

```{#lst:libExampleMComment .c caption=}
key : "value";
//single line comment...
key2 : "value2";
```

```{#lst:libExampleSComment .c caption=}
key : "value"
/*  multiple
line
comment...
*/
```

###§2.2.5 格式总结
从文本格式角度，总结Liberty格式范式如下：

整个文件LIB由若干个组(group)组成

**LIB:**

    LIB :== group *

![BNF_LIB](E:\库\documents\大四下\PJ\doc\output_sources\diagram_lib\LIB.png)

将含有圆括号的复杂属性声明和定义声明以及组定义归纳为组(group)

**group:**

    group    ::= key '(' array ')' ( ';' | '{' ( attribute | group )+ '}' )

![BNF_group](E:\库\documents\大四下\PJ\doc\output_sources\diagram_lib\group.png)

将简单属性定义为属性(atrribute)

**attribute**

    attribute ::= key ':' value ';'

![BNF_attribute](E:\库\documents\大四下\PJ\doc\output_sources\diagram_lib\attribute.png)

将若干个值组成的列表定义为数组(array)。由于存在多维数组，数组的元素可以是数组。

**array**

    array    ::= value ( ',' value )*
           | array ( ',' array )*

![BNF_attribute](E:\库\documents\大四下\PJ\doc\output_sources\diagram_lib\array.png)

**value**

    value    ::= string
           | enum
           | int
           | float
           | null

![BNF_attribute](E:\库\documents\大四下\PJ\doc\output_sources\diagram_lib\value.png)

**key**

    key      ::= string

![BNF_attribute](E:\库\documents\大四下\PJ\doc\output_sources\diagram_lib\key.png)

其中，[2]中并未给出string，int，float的严格形式定义。

§2.3 json schema介绍
-------------------

###§2.3.1 json schema概览

json schema是检验json数据结构的强有力工具，它可以规定一个json数据的结构层次，必要的键以及值的种类等。下面展示了一个json schema以及它的功能。

有一json schema 如[@lst:schemaExample]：

```{#lst:schemaExample .json caption=}
{
  "type": "object",
  "properties": {
    "first_name": { "type": "string" },
    "last_name": { "type": "string" },
    "birthday": { "type": "string", "format": "date-time" },
    "address": {
      "type": "object",
      "properties": {
        "street_address": { "type": "string" },
        "city": { "type": "string" },
        "state": { "type": "string" },
        "country": { "type" : "string" }
      }
    }
  }
}
```

同时有两个json数据如下

* 数据1

```json
{
  "name": "George Washington",
  "birthday": "February 22, 1732",
  "address": "Mount Vernon, Virginia, United States"
}
```

* 数据2

```json
{
  "first_name": "George",
  "last_name": "Washington",
  "birthday": "1732-02-22",
  "address": {
    "street_address": "3200 Mount Vernon Memorial Highway",
    "city": "Mount Vernon",
    "state": "Virginia",
    "country": "United States"
  }
}
```
以此json schema来检查这两个json，得到的结果为数据1为合法，数据2为不合法。原因是：schema中规定了键"address"的值是一个对象，而数据1中"address"的值为一个字符串。而数据2不但满足了此规定，同时满足了schema中对"address"的值对象中每个键值对的具体要求。

###§2.3.2 json schema关键字

json schema有一系列的关键字，每个关键字有不同的功能。关键字发挥了schema的作用。以下列举json schema关键字及其具体功能说明。

1. 通用的关键字

* enum。其值必须是数组，必须包含至少一个元素，每个元素必须互不相同。规定了键值对值的可取范围。若值在enum中未出现，则不合法。
* type。其值可以是字符串或数组。若是数组，数组元素必须是字符串，数组元素必须互不相同。字符串的值只能取string, number, integer, object, array, boolean, null中的一个。type关键字规定了键值对值的类型。特殊地，integer所允许的值的集合是number所允许值的集合的一个子集。其允许值集合符合的范式为：

```
    integer     ::= (-)? ( ( (digit1-9 (digit)*) ) | '0' )
    digit       ::= digit1-9
                | '0'
    digit1-9    ::= '1'| '2'| '3'| '4'| '5'| '6'| '7'| '8'| '9'
```

* allOf。其值必须是数组，数组元素个数至少为一。数组元素须是对象，每个对象内容须是合法的json schema。它规定json数据必须符合其数组中每个元素所表示的schema。
* anyOf。类似allOf。它规定json数据必须符合其一个或多个数组元素表示的schema。
* oneOf。类似allOf。它规定json数据必须符合其一个数组元素表示的schema。
* not。其值须是一个对象，对象内容须是合法的json schema。它规定json数据不能符合其值所表示的schema。
* definitions。 其值必须是一个对象。其中每个值都必须是合法的json schema。它无关目标数据合法性，仅定义了一个schema。

2. 针对对象的关键字

* maxProperties。其值必须为大于等于0的整数。规定了一个对象允许最多的键值对数目。
* minProperites。其值必须为大于等于0的整数。不出现此关键字时，默认其值为0。规定了一个对象允许最少的键值对数目。
* required。其值是一个数组，数组元素必须是不重复的字符串。规定了一个object中必须出现的键。
* properties。其值为对象。对象中的值也必须是对象，每个对象必须是合法的json schema。若json数据中出现了properties中出现过的键，其值必须符合properties中值描述的schema。
* patternProperties。其值是对象。其中每个键必须是正则表达式，其值是对象。每个对象的内容必须是合法的json schema。
* additionalProperties。其值为布尔类型。当其值为false时，如果json数据中出现了未在properties中出现，同时不符合patternProperties中键中正则描述的键，则json数据不合法。
* dependencies。其值必须是对象。其中键值对的值可以是对象或是数组。若为对象，内容必须是合法的json schema；若为数组，必须包含一个以上元素，每个元素须是字符串，且元素之间不可重复。为对象时，json数据中出现的一切与dependencies中的键相同的键，其值都必须符合dependencies中对应值所描述的schema；为数组时，若json数据中出现了与dependencies中键相同的键，则数据中必须出现以数组中每一个字符串为名的键。

3. 针对数字的关键字

* multipleOf。其值必须是一个数字。数字必须大于0。它规定了json数据中的目标数字必须是其值的整数倍。
* maximum。其值是必须是一个数字。它规定了json数据中的目标数字的最大值。
* exclusiveMaximum。其值是布尔值。当值为true时。json数据中的目标数字只能小于maximum值而不能等于。exclusiveMaximum不出现时，相当于其值为false。
* minimum。类似于maximum。规定最小值。
* exclusiveMinimum。类似exclusiveMaximum，作用于minimum。

4. 针对字符串的关键字

* maxLength。其值必须是大于等于0的数字。规定了目标字符串的最大长度。
* minLength。类似maxLength。规定了目标字符串最小长度。
* pattern。其值必须是字符串，内容是合法的正则表达式。当目标字符串能被正则表达式匹配时，目标json合法。

5. 针对数组的关键字

* items。其值必须是对象或数组。若为对象，对象的内容须是合法的json schema。若为数组，数组元素须是对象，对象的内容须是合法的json schema。其值为对象，且目标数组的每个元素都符合对象内容中的json schema时，目标数据合法；其值为数组时，目标数组需要按顺序逐一符合数组元素中的json schema，目标数据才合法。
* additionalItems。其值为布尔或对象。若为对象，其内容须是合法的json schema。其值为false且items值为数组时，不允许目标数组元素个数超过items值数组长度。
* maxItems。其值必须是大于等于0的数字。规定了目标数组的最大长度。
* minItems。类似maxItems。规定最小长度。
* uniqueItems。其值必须是布尔。其值为true时，要求目标数组的每个元素互不相同。若不出现，则认为其值为false。

6. 元数据关键字

* title。其值必须为字符串。无关目标数据的合法性。意在记录schema的标题。
* description。类似title，意在记录对于schema的一段描述。
* default。类似title，意在提供一个json值可取的默认值。

7. 特殊关键字

* \$ref。其值必须是必须是有效的json pointer[12]。schema作用时，\$ref所在对象相当于\$ref指向的json内容。它实现了schema的重用，压缩了schema文件的大小。
* \$schema。其值必须是一个URI[11]同时是有效的json reference[13]。必须出现在根schema根层次。其意图在于声明所使用的json schema语法的版本，以及将json schema与普通的json数据文件区分开来。

§2.4 小结
--------

本节介绍了Liberty和json的格式规范，给出了两种格式各自的格式范式，为下文研究格式转换方法提供了理论基础。
同时介绍了json schema。给出了json schema的形式例子，列举并介绍了json schema的各个关键字。从这些关键字的作用可以看出，json schema拥有较强的表达能力，可以在很大程度上对json数据文件进行规范。

第三章 Liberty到json转换器 
========================

§3.1 需求分析
------------

###§3.1.1 功能需求

本程序的功能为格式的转换，将.lib文件转换为Liberty。可以单独使用，也可以被其他脚本调用。考虑到这一点，程序应当为命令行程序。
为了满足管道功能，应当可以选择将结果打到标准输出，同时也应当可以指定输出到文件。
为了使程序易用，应当能够显示帮助内容。

###§3.1.2 输入输出

所有选项通过程序入口参数实现。用户可以指定输入输出文件，以及程序功能选项。选项包括是否保留注释、是否输出到文件、是否显示帮助内容。

§3.2 转换方案研究
---------------

### §3.2.1 Liberty与json格式对比

上节中给出了与本文有关的基本知识。介绍了Liberty和json的格式规范，从结构和表达形式上分别总结了两种格式的范式。

而要将Liberty转换成json格式，相当于要使用json格式来表示Liberty所表达的内容。因此需要分析两种格式的异同，确定具体的转换方案。

根据json和Liberty的范式，从构成上看，两者有以下的相同点：

* 多层次结构组织。都支持递归的多层次结构。
* 键值对。两者都用键值对作为基本单元。
* 数组。两者都支持数组，且两者都支持数组各元素类型互不相同。

有以下不同点：

* Liberty的每个组有一个类型和一个名称(name)，相当于类别后跟两个值，一个是名称，一个是组的内容。而json的对象不强制要求有名称。
* Liberty严格来讲没有二维数组。二维数组用字符串一维数组实现；而json支持任意维数组。
* Liberty支持注释；json标准不支持注释。

从表达形式上看，两者有以下相同点：

* 都使用花括号({})来标记一个块。Liberty中为组的内容，json中是对象。
* 都使用冒号(:)作为键与值的分隔符。
* 数组元素都使用逗号(,)隔开。

有以下不同点：

* Liberty键值对后都有分号(;)；而json用逗号(,)。且json中，最后一个键值对不需要逗号。
* Liberty使用圆括号(())来标记数组；而json使用方括号([])。
* Liberty中空格符和回车符作为关键字；而json中不是。

###§3.2.2 转换方案

根据以上分析的Liberty和json格式的区别，提出格式转换的方案如下：

1. 组：

Liberty的组格式如[@lst:libG]：

```{#lst:libG .c caption=}
groupA(nameA)
{
    /* key-values*/
}
```

从json格式中的对象来考虑，它的内容实际上可以用json表示如[@lst:jsonG0]：

```{#lst:jsonG0 .json caption=}
"groupA" : 
{
    "name" : "nameA",
    "group_Content" : 
    {
        /* key-values */
    }
}
```
使用"name"作为特殊的关键字，其值为group的名称。然而可以看到这种形式不够简洁，增加了层次。可以进行简化，如[@lst:jsonG1]：

```{#lst:jsonG1 .json caption=}
"groupA" : 
{
    "name" :　"nameA",
    /* other key-values */
}
```
注意实际情况中name可能是字符串或数组，但由于它作为值存在，为数组或字符串都是合法的。

2. 简单属性

Liberty的简单属性格式如[@lst:libS0]：

```{#lst:libS0 .c caption=}
key1 : "string1";
key2 : string2;
key3 : 12450;
key4 : true;
```

由于Liberty中字符串有时使用双引号有时不使用，而json中字符串必须使用双引号。转换时必须考虑这一点并将其统一化。

简单属性可以用json表示如[@lst:jsonS0]：

```{#lst:jsonS0 .json caption=}
"key1" : "string1",
"key2" : "string2",
"key3" : 12450,
"key4" : true
```


3. 复杂属性和定义属性

Liberty的复杂属性和定义属性格式如[@lst:libC0]：

```{#lst:libC0 .c caption=}
key1 ( 1, 2, 4, 5, 0);
capacitive_load_unit (1.0, "ff");
values ("0.00, 0.21","0.11,0.23");
define (bork, pin, string);
curve_y (1, "0.8, 0.5, 0.2");
```

使用json的数组可以表示[@lst:libC0} 为{@lst:jsonC0] ：

```{#lst:jsonC0 .json caption=}
"key1" : [1, 2, 4, 5, 0],
"capacitive_load_unit" : [ 1.0, "ff" ],
"values" : [ [0.00, 0.21], [0.11, 0.23] ],
"define" : ["bork", "pin", "string"],
"curve_y" : [
                [1],
                [0.8,0.5,0.2]
            ]
```

§3.3 转换程序设计
---------------

### §3.3.1 格式转换概论

通常地讲，格式转换器属于编译器，它将原始语言转换成目标语言。但是两者的目的有所区别（编译的目的是形成可执行程序）。格式转换的过程是编译过程的一部分，其构成是词法分析、语法分析、改变格式并输出。
其中，词法分析的目标是将输入的字符序列转换为标记(token)序列，供后续步骤使用。
这里的标记是源语言的最小单位，以[@lst:token0] 来说明：

```{#lst:token0 .c caption=}
capacitive_load_unit (1.0, "ff");
```

其中所含的标记序列如[@tbl:token]所示：

  | 语素                 | 标记类型 |
  |----------------------|----------|
  | capacitive_load_unit | 标识符   |
  | ' '                  | 空白符   |
  | \(                   | 左圆括号 |
  | 1.0                  | 标识符   |
  | ,                    | 逗号     |
  | ' '                  | 空白符   |
  | "ff"                 | 标识符   |
  | )                    | 右圆括号 |
  | ;                    | 语句结束 |
  : 标记序列表 {#tbl:token}

值得注意的是，词法分析只做标记的识别与分类，不考虑标记直接的相互关系。例如它不考虑这里左圆括号与右圆括号是否匹配。
得到标记序列的过程是词法分析的第一步，被称作标记化(tokenization)。执行这个步骤的功能部件称作标记生成器。词法分析的另一个步骤叫做评估(Evaluate)。执行评估的功能部件叫做评估器(Evaluator)。
评估器对第一步产生的标记序列给出估值，它可以直接舍弃一些无用的标记。

语法分析是对标记序列进行处理，根据规定的某种形式文法(Formal grammar)，确定标记序列的语法结构的过程。语法结构是例如“表达式”、“复杂属性的键”等内容。语法分析过程要判定是否有语法错误，例如左右括号不匹配。在发现错误时输出错误信息。

完成语法分析后，进行格式的转换和输出。根据目标格式的要求，将语法结构反过来组合成新的字符序列。这一过程将在语法结构上添加目标格式的标记。例 的输出将如[@lst:tokenOut]所示：

```{#lst:tokenOut  .json caption=}
"capacitive_load_unit":[1.0,"ff"]
```

格式转换过程以框图表示为{@fig:lexicalAnalysis}：

![diagram_format_change](E:\库\documents\大四下\PJ\doc\output_sources\Diagram_Lexical_analysis_.png){#fig:lexicalAnalysis}

### §3.3.2 正则表达式和有限状态自动机

词法分析通常基于有限状态自动机(FSM)。有限状态自动机是一个数学模型，表示有限个状态以及状态之间转移的动作等行为。通过有限自动机可以实现字符串匹配等功能。

正则表达式是一种用有限的字符集来描述的字符串，每个正则表达式字符串有一定的句法意义，它可以匹配一系列字符串。正则表达式等价于有限状态自动机。由于它功能强大、使用方便且灵活性高，在字符串匹配中经常使用正则表达式。

一个正则表达式的字符串一般被称作一个模式(pattern)，一个模式可以匹配一系列的字符串。一般正则表达式形式上有以下几种结构：
* 选择。一个竖直分隔符(|)代表选择。匹配被其分隔的字符集中其中一个。
* 数量限定。数量限定符号有加号(+)、问号(?)、和星号(*)几种。它们各自有不同的意义，作用都在于限定其前面出现的字符集的出现次数。不使用数量限定时，意义为出现一次。
* 匹配。使用圆括号来定义操作符的范围以及优先度。圆括号中的字符集作为一个单元，与其他功能结构作用，就可以构造复杂的正则表达式。

正则表达式在功能上还有引用、贪婪与非贪婪匹配等功能。引用一个子字符串是指，该子字符串要与前面某段子表达式匹配到的字符串相同。贪婪与非贪婪匹配时数量限定的方式。贪婪模式下，限定次数为某数量以上时，会匹配尽量多的重复次数，反之则会匹配尽量少的重复次数。使用中合理利用正则的这一类功能，可以满足不同的要求。

###§3.3.3 核心功能的实现

参照编译的“从词法分析到语法分析再进行格式转换”的步骤即可以实现格式转换，但是由于将处理过程分成了三个步骤进行，程序需要对整个序列进行三次处理。考虑到实际中使用的Liberty文件的规模很大，三次处理的效率低下，会导致执行时间过长。
为了解决这一问题，这里采取的做法是三个步骤在一次扫描中同时完成。核心功能的实现方法详述如下：

1. 初步分割

程序从将第一个字符开始遍历输入文件直至文件最后一个字符。根据每次遇到的字符，首先判断以下4种情况：

* 斜杠(/)
* 空白符
* 右花括号(})
* 其他

对于每一种情况，程序都会继续向后查找到相应的匹配字符，并处理中间的一段字符串。这相当于根据语法要求获取一段包含一个或多个语法结构的段落。处理完成后继续此过程。
其中斜杠情况包括了单行注释和跨行注释；右花括号是组的结束；其他情况包括了组声明、各种属性声明。以下进行具体介绍。

**注释处理**

由于json标准不支持注释，但是一些json解释器可以支持c风格的注释——可以读入带注释的json而不认为其不合法。因此此程序添加保留注释或去掉注释的选项。
实现上，单行注释提取"//"和换行符中间的内容；多行注释提取"/\*"和"\*/"之间的内容。

**空白符**

虽然Liberty中要求保留一些空白符，但是不处于一对双引号之间的空白符不存在任何意义，处理时直接忽略之。
这里的空白符包括空格符( )，制表符(\\t)和换行符(\\n)。

**各种声明**

没有找到以上几种字符时，首先程序向后寻找一个左花括号({)或分号(;)，可以提取各种声明。其中左花括号左边的内容是组声明，分号左边的是其他声明。
对于组声明，使用左右圆括号提取组类别和组的名称；对于简单声明，使用冒号(:)提取键与值；复杂声明的标记提取与组声明相同。

初步分割的程序框图如{@fig:getToken}：

![getToken](){#fig:getToken}


2. 值的类型判断及处理

分割后的字符串有以下几类：

* 组的类型
* 组的名称
* 各种属性的键
* 简单属性值
* 复杂属性和定义属性值

其中，组的类别和属性的键都是键，其类型是字符串；组的名称和复杂属性等同，其值可以是字符串或数组。
由于不同类型的值在输出时形式有别：字符串需要有双引号括起。因此必须有判定标记类型的功能。

使用正则表达式来匹配数字类型，布尔类型和null进行简单判断，余下情况即为字符串。

3. 数组的解析

复杂属性以及组的名称有可能是数组。Liberty中数组的出现有几种典型情况，如[@lst:libExampleArray3]：

```{#lst:libExampleArray3 .c caption=}
ff(IQ,IQN) {/*...*/}
index_1 ("0.028, 0.044, 0.076, 0.138, 0.264, 0.516, 1.02");
values ( \
          "0.006798, 0.007190, 0.007530, 0.007708, 0.007930, 0.008291, 0.008939", \
          "0.006802, 0.007194, 0.007534, 0.007713, 0.007933, 0.008296, 0.008945");
piece_define ("0 10 20") ;
curve_y (1, "0.8, 0.5, 0.2");
```

可以看到Liberty中数组的表示方式十分灵活。从维度上看，支持一维和二维数组。从双引号使用方式上看，一维数组可以使用双引号也可以不使用，二维数组中的每个一维数组也可以使用或不使用双引号。但如例2-中curve_y，如果都不使用双引号，则无法区分一维和二维数组。从空格的使用情况来看，双引号外的空格没有意义，但双引号内的空格有时作为数组元素的分隔符，有时没有意义。

解决这个问题的算法如下：

------------------------------------

       SPLIT_ARRAY(A_STRING):
    1    list L
    2    //first time split:
    3    if A_STRING.contain(',') AND A_STRING.not_contain('"') then L <-A_STRING
    4    else L <- SPLIT_A_BY_COMMA_OUTSIDE_DOUBLEQUATE
    5    //second time split:
    6    for str in L
    7      do DELETE_BLANK_AFTER_COMMA(str)
    8      if spliteByBlank then SPLIT_BY_BLANK(str)
    9      else SPLIT_BY_COMMA(str)

------------------------------------

4. 添加逗号

不同于Liberty每个属性后都有一个分号结束，json的键值对后面使用的是逗号，且一组键值对的最后一个元素不允许有逗号。因此需要根据情况添加逗号。
添加逗号的算法如下：
在每次添加一个键值对的时候考虑上一个键值对的逗号是否添加。需要考虑以下两个情况：

* 对象中第一个键值对前面不能添加逗号。
* 注释后的一个键值对前面不能添加逗号。

因此在对象结束后和添加注释后进行标记，添加逗号时根据标记的值确定是否需要添加即可。

5. 语法错误处理

源文件发生语法错误时，不能转换成为合法的json，程序将报错，输出发生语法错误的相关字符串内容，输出已完成的内容，并错误退出。

6. 排版功能

转换成json时，为了输出文件格式美观，需要进行排版。排版的主要内容是实现缩进。实现方法如下：
使用一个变量记录当前的层次深度。每次遇见新的对象，即进入更深一个层次时，深度增加;遇到对象结束时，深度减小。在每个键值对的前面，根据深度添加制表符(\t)。

###§3.3.4 程序入口参数处理

为了让程序能在命令行下面使用，需要对入口参数进行处理。程序参数列表如下：

* -o 文件名
* -c
* -h
* 文件名

如果程序没有参数，将从标准输入获取输入。有参数时，输入需要从文件中获得，程序将优先检查是否出现-h，若出现-h则无视其它参数，程序显示帮助内容后退出；若不出现-h，程序优先查找-o，并将-o后面一个参数作为输出文件名，若-o后无参数，程序报错；最后程序将其他参数当作输入文件，若输入文件数量超过1个，程序报错。

§3.4 测试
---------------

程序设计过程中同时进行了单元测试。测试不通过时，修改程序至测试通过。最终完成设计实现。单元测试内容繁多，不在此赘述。这里列举最终的黑盒测试结果。
程序编译使用gcc5.3，测试环境为linux opensuse 13.1。

1. 入口参数测试

已有一文件input.lib,其内容如[@lst:inputLib]所示：

```{#lst:inputLib  .c caption=}
library(a){
  //comment
  key1: value1 ;
  /* comment2 */
  key2: value2 ;
}
```

入口参数测试结果如[@tbl:inputTest]：

  | 输入参数 | 测试目的   | 期待结果         | 通过情况 |
  |----------|------------|------------------|----------|
  | 无       | 无输入测试 | 程序等待标准输入 | 通过     |
  | -h       | 帮助       | 显示帮助内容     | 通过     |
  |-h input.lib                    |有其他参数时帮助    |显示帮助内容         |通过  |
  |input.lib                       |输入文件参数       |显示转换结果，无注释   |通过  |
  |-c input.lib                    |-c参数            |显示帮助内容，有注释   |通过  |
  |input.lib -o output.json        |-o 参数           |结果输出到文件        |通过  |
  |input.lib -o                    |-o后无文件名       |报错                |通过  |
  |input.lib -o output.json -c    |-o与-c同时存在  |结果输出到文件，结果有注释 |通过  |
  |-o output.json -c input.lib    |参数顺序无关性   |结果输出到文件，结果有注释  |通过  |
  : 入口参数测试表 {#tbl:inputTest}

2. 转换功能测试

使用一个以Liberty格式书写的输入文件统一测试程序对于各种情况的属性声明的转换结果。该文件内容如[@lst:convertTest1]

```{#lst:convertTsst1   .c caption=}
convertTest ("headname") {
  normalGroup ("name1")
  {
    //simple
    normalkey : stringWithoutQuotes ;
    normalkey2 : "stringWithQuotes" ;
    normalkey3 : "string with blank" ;
    //array
    normallist("1,2,4,5,0");
    listBlank("1 2 4 5 0");
    2dList("1,2,4,5,0","2,3,3,5,6");
    2dListMultiLine("1,2,4,5,0",\
            "2,3,3,5,6");
    strangeList(strangeElement,"1,2,3,4,5,6",anotherStrangeElement,30);
    singleElement("b");
    2d_SingleElment("b","c");
    /*type test*/
    bool1 : true ;
    bool2 : "false" ;
    null : null;
    number1 : 1;
    number2 : -1;
    number3 : 0;
    number4 : -0;
    number5 : 0.1;
    number6 : -0.5e-33;
    number7 : 99E+120;
    notNumber1 : +1;
    notNumber2 : +0;
    notNumber3 : .3;
  }
  listName1 (A,  B)
  {
  }
  listName2 ("A,B")
  {
  }
  deepGroup ("deep1")
  {
    group("deep2")
    {
      group("deep3")
      {}
    }
  }
}
```

不保留注释测试，结果如[@lst:convertOut1]所示：

```{#lst:convertOut1   .json caption=}
{

  "convertTest" : 
  {
    "name" : "headname",
    "normalGroup" : 
    {
      "name" : "name1",
      "normalkey" : "stringWithoutQuotes",
      "normalkey2" : "stringWithQuotes",
      "normalkey3" : "string with blank",
      "normallist" : [1,2,4,5,0],
      "listBlank" : [1,2,4,5,0],
      "2dList" : 
      [
        [1,2,4,5,0],
        [2,3,3,5,6]
      ],
      "2dListMultiLine" : 
      [
        [1,2,4,5,0],
        [2,3,3,5,6]
      ],
      "strangeList" : 
      [
        ["strangeElement"],
        [1,2,3,4,5,6],
        ["anotherStrangeElement"],
        [30]
      ],
      "singleElement" : ["b"],
      "2d_SingleElment" : 
      [
        ["b"],
        ["c"]
      ],
      "bool1" : true,
      "bool2" : false,
      "null" : null,
      "number1" : 1,
      "number2" : -1,
      "number3" : 0,
      "number4" : -0,
      "number5" : 0.1,
      "number6" : -0.5e-33,
      "number7" : 99E+120,
      "notNumber1" : "+1",
      "notNumber2" : "+0",
      "notNumber3" : ".3"
    },
    "listName1" : 
    {
      "name" : ["A","B"]
    },
    "listName2" : 
    {
      "name" : ["A","B"]
    },
    "deepGroup" : 
    {
      "name" : "deep1",
      "group" : 
      {
        "name" : "deep2",
        "group" : 
        {
          "name" : "deep3"
        }
      }
    }
  }
}

```

3. 错误输入测试

使用格式错误的输入文件测试程序的错误处理能力。使用程序的标准输入模式进行测试。命令形式均为：
  $./lib2json
之后输入测试输入内容。测试内容及程序所报错误见[@tbl:wrongTest]。程序还会输出已转换部分内容，表中不记录：

|输入           |所报错误     |
|-----------   |----------:|
|head           |statement format: syntax error! |
|head()         |statement format: syntax error! |
|head(name      |statement format: syntax error! |
|h(n){a}     |statement format: syntax error! |
|h(n){a:b}     |statement format: syntax error! |
|h(n){a[[]]}     |statement format: syntax error! |
|h(n){a();}     |keyValue not match! kv:$a();$ syntax error|
|h(n){a:;}     |keyValue not match! kv:$a:;$ syntax error|
|h(n){a([[]]);}     |keyValue not match! kv:$a([[]);;$ syntax error|
|h(n){a:;;}     |keyValue not match! kv:$a:;$ syntax error|
|h(n){a:a;;}     |keyValue not match! kv:$;$ syntax error|
|h(n){/*}       |cannot find "\*/" syntax error!|

: 错误处理测试表 {#tbl:wrongTest}

§3.5 小结
---------------

本节通过对比Liberty和json格式的异同，给出了用json表示Liberty的具体方案。并给出了详细的转换程序设计，以及程序的测试结果。测试结果上看，程序拥有完整的功能，并有一定程度的错误处理能力。
实现了转换程序，即可以将原有的Liberty文件转换成为json文件，便于使用json schema检查其内容。下一章将介绍将json schema在检查Liberty所转换的json文件内容的规范的应用。

第四章 Liberty详细内容及json schema编写
====================================

§4.1 Liberty功能简述
-------------------

为了针对特定应用编写正确的json schema，需要对Liberty的内容进行研究，确定特定声明的具体内容。
Liberty的功能涵盖面很广，可以表达工艺库(Technology Library)、环境(Environments)、核心的单元(Core Cells)、时序单元(Sequential Cells)、测试单元(Test Cells)、时序弧(Timing Arcs)、静态和动态电源(Power and Electromigration)、低功耗模型(Advanced Low-Power Modeling)、复合电流源(composite current source)等。
特定应用中，所使用的Liberty声明将是所有Liberty内容规范的一个子集。针对一个子集编写一个json schema，即可对一系列文件内容规范进行定义。

§4.2 json schema编写
-------------------

下面举例说明json schema的编写方法。

使用的library的开头部分如[@lst:s_part1]所示：

```{#lst:s_part1 .json}
{
  "library" : 
  {
    "name" : "typical_1v2c25",
    "delay_model" : "table_lookup",
    "in_place_swap_mode" : "match_footprint",
    "library_features" : ["report_delay_calculation"],
    "revision" : 1.0,
```

其中library为整个json对象的唯一内容。编写json schema时，首先编写根schema内容。然后根据liberty文档中的内容定义，编写针对每一个键值对的子schema。对应schema开头部分如[@lst:sj_part1]所示：

```{#lst:sj_part1 .json}
{
    "$schema": "http://json-schema.org/draft-04/schema#",
    "properties" : {
        "library": {
            "type" : "object",
            "properties": {
                "name": {
                    "type": "string"
                },
                 "delay_model": {
                    "type": "string",
                    "enum":["table_lookup"]
                },
                "in_place_swap_mode":{
                  "type":"string"
                },
                "library_features":{
                  "type":"array",
                  "items":{
                    "type":"string",
                    "enum":["report_delay_calculation", "report_noise_calculation", "report_user_data","allow_update_attribute"]
                  }
                },
                "revision":{
                    "type":["number","string"]
                },
```

对于对象类型，需要声明其包含的内容，使用properties关键字。每个键都要对应一个子schema。对于字符串枚举，使用enum关键字，根据文档中规定的其可选值规定其可选范围。对于数组类型，使用items来规定其数组元素的内容。

有时需要规定数字类型值的取值范围，使用minimum和maximum关键字，如[@lst:sj_max_min]所示:

```{#lst:sj_max_min .json}
"output_threshold_pct_fall":{
              "type": "number",
              "minimum":0.0,
              "maximum":100.0
          },
```
有时字符串类型的值是布尔表达式，或者特定的其它表达式，可以用pattern关键字使用正则表达式来规定，如[@lst:sj_regex]所示：

```{#lst:sj_regex .json}
"vomin":{
  "oneOf":
  [
    {"type":"number"},
    {
      "type":"string",
      "pattern":"(((\\d+\\.\\d+|\\d+)|(VDD)|(VSS)|(VCC))\\s*[+\\-*\\/]\\s*)*(((\\d+\\.\\d+)|(\\d+))|(VDD)|(VSS)|(VCC))\\s*"
    }
  ]
},
```

其中oneOf说明vomin可以为数字，也可以为满足该pattern的字符串。

内容规范完全相同的，可以使用json schema的\$ref关键字，如[@lst:sj_ref]所示：

```{#lst:sj_ref .json}
  "fall_transition":{
    "$ref":"#/properties/library/properties/cell/properties/pin/properties/internal_power/properties/rise_power"
  }
```

一般properties关键字后面配合使用required关键字，规定properties中必须出现的内容。在pin中，使用的required如[@lst:sj_req]中片段：

```{#lst:sj_req .json}
  },
  "required":["name","direction","capacitance"]
}
```

§4.3 json schema验证测试
-------------------

编写一个json schema文件，对3个被测文件进行了测试。使用[14]中的工具进行测试。由于文件较大无法测试，删去第二个cell组开始的重复cell组。三个测试均通过。


第五章 总结与展望
====================================

§5.1 工作总结
-------------------

数据格式是软件开发中的一个重要因素，合理的数据格式设计可以便于程序的编写。json作为当前流行的数据格式，拥有很多优点。因此本文主要研究如何使用json文件格式来表示Liberty文件的内容，并用json schema验证内容规范。
Liberty格式发展时间较长，修改多次，其中已经有一些不合理性，比如文中列举的其复杂属性的数组形式。本文实现了Liberty转换到json格式的程序，对程序进行了个方面测试。并给出了json schema在规范库文件的应用实例。

§5.2 工作的不足以及工作展望
-----------------------

本文的工作内容偏向于基本格式的规范与转换，没有提出一个新标准。同时，在研究Liberty文档时，由于文档内容繁多，无法确保没有疏漏。因此，如何制定可以完全包含Liberty内容的新标准，以使各EDA软件可以完全使用json作为数据格式，是可以继续进行的一项工作。

参考文献
=======

[1] Introducing JSON.http://www.json.org/

[2] Liberty15.https://www.opensourceliberty.org/

[3] Liberty07.www.eecs.berkeley.edu/~alanmi/publications/other/

[4] Understanding JSON Schema.http://spacetelescope.github.io/understanding-json-schema/

[5] JSON Schema: core definitions and terminology
json-schema-core.http://json-schema.org/latest/json-schema-core.html

[6] JSON Schema.https://en.wikipedia.org/wiki/JSON#JSON_Schema

[7] Lexical analysis.https://en.wikipedia.org/wiki/Lexical_analysis

[8] Parsing.https://en.wikipedia.org/wiki/Parsing

[9] Regular Expression.https://en.wikipedia.org/wiki/Regular_expression

[10] The JavaScript Object Notation (JSON) Data Interchange Format, MARCH 2014.http://www.rfc-editor.org/info/rfc7158

[11][RFC3986]  Berners-Lee, T., Fielding, R., and L. Masinter, “Uniform Resource Identifier (URI): Generic Syntax,” STD 66, RFC 3986, January 2005

[12][json-pointer]   Bryan, P. and K. Zyp, “JSON Pointer (work in progress),” September 2012.

[13][json-reference]   Bryan, P. and K. Zyp, “JSON Reference (work in progress),” September 2012.

[14] JSON Schema Validator.http://www.jsonschemavalidator.net/


<!--stackedit_data:
eyJoaXN0b3J5IjpbNTM1MDAzMDE4LC01ODE5ODYzOThdfQ==
-->