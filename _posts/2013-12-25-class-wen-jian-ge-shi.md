---
layout: post
title: "Class文件格式"
description: "描述 Java 虚拟机中定义的 Class 文件格式。每一个 Class 文件都对应着唯一一个类 或接口的定义信息,但是相对地,类或接口并不一定都得定义在文件里(譬如类或接口也可以通过 类加载器直接生成)。这里,我们只是通俗地将任意一个有效的类或接口所应当满足的格式称为 “Class 文件格式”,即使它不一定以磁盘文件的形式存在。"
category: ""
tags: [JVM]
---
{% include JB/setup %}

###ClassFile结构

    ClassFile {
        u4 magic;        //魔数,唯一作用是确定这个文件能否被虚拟机所接受的Class文件。魔数值固定为0xCAFEBABE,不会改变。

        u2 minor_version;    //副版本号
        u2 major_version;    //主版本号

        //常量池计数器,constant_pool_count 的值等于 constant_pool 表中的成员数加 1。
        //constant_pool 表的索引值只有在大于 0 且小于 constant_pool_count 时才会被 认为是有效的2
        //对于 long 和 double 类型有例外情况
        u2 constant_pool_count;

        //常量池,constant_pool 是一种表结构.
        //它包含Class文件结构及其子结构中引用的所有字符串常量、类或接口名、字段名和其它常量。
        //常量池中的每一项都具备相同的格式特征——第一个字节作为类型标记用于识别该项是哪种类型的常量,称为“tag byte”。
        //常量池的索引范围是 1 至 constant_pool_count−1。
        cp_info constant_pool[constant_pool_count-1];

        //访问标志,access_flags 是一种掩码标志,用于表示某个类或者接口的访问权限及基础属性
        //取值范围和相应含义见下表
        u2 access_flags;

        //类索引,this_class的值必须是对constant_pool表中项目的一个有效索引值。
        //constant_pool表在这个索引处的项必须为CONSTANT_Class_info类型常量,表示这个 Class 文件所定义的类或接口。
        u2 this_class;

        //父类索引,对于类来说,super_class 的值必须为 0 或者是对 constant_pool 表中 项目的一个有效索引值。
        //如果它的值不为 0,那 constant_pool 表在这个索引处的项 必须为 CONSTANT_Class_info 类型常量.
        //表示这个 Class 文件所定义的 类的直接父类。
        //当前类的直接父类,以及它所有间接父类的 access_flag中都不能带有ACC_FINAL标记。
        //对于接口来说,它的 Class 文件的 super_class 项的值必须是 对 constant_pool 表中项目的一个有效索引值。
        //constant_pool 表在这个索引处的 项必须为代表 java.lang.Object 的 CONSTANT_Class_info 类型常量.
        //如果 Class 文件的 super_class 的值为 0,那这个 Class 文件只可能是定义的是 java.lang.Object 类,只有它是唯一没有父类的类。
        u2 super_class;

        //接口计数器,interfaces_count 的值表示当前类或接口的直接父接口数量。
        u2 interfaces_count;

        //接口表,interfaces[]数组中的每个成员的值必须是一个对 constant_pool 表中项 目的一个有效索引值,
        //它的长度为 interfaces_count。每个成员 interfaces[i] 必 须为CONSTANT_Class_info类型常量,其中0 ≤ i < interfaces_count。
        //在 interfaces[]数组中,成员所表示的接口顺序和对应的源 代码中给定的接口顺序(从左至右)一样
        //即interfaces[0]对应的是源代码中最左边的接口。
        u2 interfaces[interfaces_count];

        //字段计数器,fields_count 的值表示当前 Class 文件 fields[]数组的成员个数。
        //fields[]数组中每一项都是一个 field_info 结构的数据项,它用于表示 该类或接口声明的类字段或者实例字段.
        u2 fields_count;

        //字段表,fields[]数组中的每个成员都必须是一个 fields_info 结构的数据项,用于表示当前类或接口中某个字段的完整描述。
        //fields[]数组描述当前类或接口 声明的所有字段,但不包括从父类或父接口继承的部分。
        field_info fields[fields_count];

        //方法计数器,methods_count 的值表示当前Class文件methods[]数组的成员个数。
        //Methods[]数组中每一项都是一个 method_info结构的数据项。
        u2 methods_count;

        //方法表,methods[]数组中的每个成员都必须是一个 method_info 结构
        method_info methods[methods_count];

        //属性计数器
        u2 attributes_count;

        //属性表,attributes 表的每个项的值必须是 attribute_info结构
        attribute_info attributes[attributes_count];
    }



| *标记名* | *值* | *含义* |
|:-------|:-------|:-------|
| ACC_PUBLIC | 0x0001 |￼可以被包的类外访问 |
| ACC_FINAL | 0x0010 | 不允许有子类 |
| ACC_SUPER | 0x0020 | 当用到 invokespecial 指令时,需要特殊处理3的 父类方法 |
| ACC_INTERFACE | 0x0200 |￼标识定义的是接口而不是类 |
| ACC_ABSTRACT | 0x0400 |￼不能被实例化 |
| ACC_SYNTHETIC | 0x1000 |￼标识并非 Java 源码生成的代码 |
| ACC_ANNOTATION | 0x2000 |￼标识注解类型 |
| ACC_ENUM | 0x4000 |￼标识枚举类型 |


###各种内部表示名称

####类和接口的二进制名称
在 Class 文件结构中出现的类或接口的名称,都通过全限定形式(Fully Qualified Form) 来表示,这被称作它们的“二进制名称”。这个名称使用 CONSTANT_Utf8_info结构来表示,因此如果忽略其他一些约束限制的话,这个名称可能来自整个 Unitcode 字符空间的任意字符组成。类和接口的二进制名称还会被 CONSTANT_NameAndType_info结构所引用,用于构成它们的描述符,引用这些名称通过引用它们的 CONSTANT_Class_info结构来实现。

####非全限定名
方法名,字段名和局部变量名都被使用非全限定名(Unqualified Names)进行存储。
非全限定名中不能包含 ASCII 字符"."、";"、"\["和"/"(也不能包含他们的Unitcode 表示形式,既类似"\u2E"这种形式)。
方法的非全限定名还有一些额外的限制,除了实例初始化方法"&lt;init&gt;"和类初始化方法 "&lt;clinit&gt;"以外,
其他方法非全限定名中不能包含 ASCII 字符"<"和">"(也不能包含他们的 Unicode 表示形式).

###描述符和签名
描述符(Descriptor)是一个描述字段或方法的类型的字符串。在Class文件格式中,描述符使用改良的 UTF-8 字符串来表示。
如果不考虑其他的约束,它可以使用Unicode字符空间中的任意字符。
签名(Signature)是用于描述字段、方法和类型定义中的泛型信息的字符串。

####语法符号
描述符和签名都是用特定的语法符号(Grammar)来表示,这些语法是一组可表达如何根据不同的类型去产生可恰当描述它们的字符序列的标识集合。


