## JAVA

环境：java-sec-code

框架：springboot

Q&A：

- 查询结果如果为jar包或第三方导入内容，无法解析

### 框架检查

依赖检查

- pom.xml

```
#常用框架，涉及数据库、权限、前端...
Struts
Spring
Hibernate
Mybatis
Shiro
Vue
React
BootStrap
Dubbo
...
```

导入格式

```
<!-- https://mvnrepository.com/artifact/struts/struts -->
<dependency>
    <groupId>struts</groupId>
    <artifactId>struts</artifactId>
    <version>1.2.9</version>
</dependency>
```

依赖pom.xml文件检查脚本，自定义配置第三方依赖黑名单检查

```
#encoding:utf-8
import os
import re
from xml.dom.minidom import parse
import xml.dom.minidom
from optparse import OptionParser

options = OptionParser(usage="%prog [options]", description="Check Java Project Unsafe import.")
options.add_option("-f", "--file", type="string", dest="file", default="pom.xml", help="Input Java Project'spom.xml")
options.add_option("-b","--blacklist",type="string", dest="blacklist", default=None, help="Input the file which record Unsafe sources file.")

unsafe_blacklist = {
    "":(),
    "":()
}

"""
<!-- https://mvnrepository.com/artifact/struts/struts -->
<dependency>
    <groupId>struts</groupId>
    <artifactId>struts</artifactId>
    <version>1.2.9</version>
</dependency>
"""

def check_import(para_file, blacklist):
    result = ""
    
    DOMTree = xml.dom.minidom.parse(para_file)
    collection = DOMTree.documentElement
    depends = collection.getElementsByTagName("dependency")
    
    for depend in depends:
        groupId = depend.getElementsByTagName("groupId")
        if groupId in blacklist.keys():
            version = depend.getElementsByTagName("version")
            if version in blacklist[groupId]:
                result += groupId + "--" + version
                result += "\n"
    
    return result

def main():
    opts, args = options.parse_args()

    para_file = opts.file
    para_black = opts.blacklist

    if para_black == None:
        blacklist = unsafe_blacklist
    else:
        blacklist = para_black

    print(check_import(para_file, blacklist))

if __name__ == '__main__':
    main()
```

### 基础语句

官方文档：[CodeQL for Java and Kotlin — CodeQL (github.com)](https://codeql.github.com/docs/codeql-language-guides/codeql-for-java/)

#### 基础查询

基础结构

```
//查询语句需要一个或多个import指定语法
import xxx

//from指定查询的变量，变量的声明type var-name
from xxx

//where指定针对变量的查询条件
where xxx

//select指定每次输出的结果，可输出对应的变量和提示的信息 具体格式为element,"alert message"
select  xx
```

以下demo示范输入字符串的为空判断

```
public class TestJava {
    void myJavaFun(String s) {
        boolean b = s.equals(""); //s.isEmpty()
    }
}
```

**CodeQL**

MethodAccess ：方法访问，调用具有参数列表的方法

以下语句获取具有参数列表的方法，寻找方法名为equals且第一个参数值为""的方法

输出方法和log

```
from MethodAccess ma
where
    ma.getMethod().hasName("equals") and
    ma.getArgument(0).(StringLiteral).getValue() = ""
select ma, "This comparison to empty string is inefficient, use isEmpty() instead."
```

查找指定方法名为a且第b位参数为c的方法

```
from MethodAccess ma
where
    ma.getMethod().hasName("a") and
    ma.getArgument(b).(StringLiteral).getValue() = "c"
```

点击equals可以查看源码对应内容

![image-20230201161233236](codeql学习.assets/image-20230201161233236.png)

以下demo将需要搜索的结果类范围扩大，由String更改为Object

```
public class TestJava {
    void myJavaFun(Object o) {
        boolean b = o.equals("");
    }
}
```

此时查找会返回期望的String以外的Object类的结果

where处的条件限制改为如下语句，and语句扩展条件

.getQualifier：获取方法的限定表达式，如果有。此处判断是否为String类型的，避免Object类结果出现

```
where
  ma.getQualifier().getType() instanceof TypeString and
  ma.getMethod().hasName("equals") and
  ma.getArgument(0).(StringLiteral).getValue() = ""
```

查找特定类，指定名称中包含xx关键词

```
import java
from Class c
where c.getQualifiedName().indexOf("Config") >=0
select c.getQualifiedName() 
```

.getQualifiedName返回类名

![image-20230201155841349](codeql学习.assets/image-20230201155841349.png)

#### 基础java库

分析java程序时，可以直接使用封装好的集合

使用时记得.ql语句导入

```
import java
```

标准库中最重要的类具体可以分为以下五个类型

- 表示程序元素的类，例如类和方法
- 表示AST节点的类，例如语句和表达式
- 表示元数据的类，例如注释和评论
- 表示计算指标的类，例如圈复杂度和耦合
- 用于指导程序调用图的类

##### Elements

程序的要素需要注意

- 类要素包含：包Package、编译单元CompilationUnit、类型Type、方法Method、构造函数Constructor、变量Variable
- 共同的顶级super/父类是Element，其提供了通用的程序谓词，用于确定元素和名称和判断两个元素是否互相嵌套
- 引用一个可能是方法或构造方法的元素可使用Callable，Callable类是Method和Constructor的公共super类（就是不确定具体类的时候找需要类的公共父类(ˉ▽ˉ；)）

##### Types

Type有许多子类表示不同的类型

- PrimitiveType：表示原始类型，即boolean、byte、char、double、float、int、long、short、void、nulltype之一

- RefType：引用类型/非原始类型，其子类为Class类、Interface接口、EnumType枚举、Array数组

  如下语句即在程序中查询所有类型为int的变量

  ```
  import java
  
  from Variable v,PrimitiveType pt
  //获取类型指定类型为int
  where pt=v.getType() and pt.hasName("int")
  select v
  ```

  引用类型也根据其声明范围分类：TopLevelType表示在编译单元的顶层声明、NestedType即非Top

  如下语句查询名称与其编译单元名称不同的所有顶级类型

  ```
  import java
  
  from TopLevelType tl
  where tl.getName() != tl.getCompilationUnit().getName()
  select tl
  ```

  对应其他封装的Class：

  TopLevelClass：编译单元的顶层声明的类

  NestedClass：区别与Top的类，如LocalClass在方法或构造函数中声明的类、AnonymousClass匿名类

  还包括TypeObject、TypeCloneable、TypeRuntime等，具体含义同名

  如下语句可查找所有直接扩展Object的嵌套类

  ```
  import java
  
  from NestedClass nc
  //.getASupertype获取类对应的父类，此处配合使用*可递归查找所有父类
  where nc.getASupertype() instanceof TypeObject
  select nc
  ```

##### Generics

还有几个Type的子类用于处理泛型

GenericType：表示GenericInterface或GenericClass，可表示类型或声明，例如java中的java.util.Map接口

TypeVariable：类型参数，如泛型声明中的K和V

泛型的参数化实例需要提供具体类型，类型由ParameterizedType表示，实例化它的泛型类型由GenericType表示，从泛型到实例化可以配合谓词getSourceDeclaration

以下语句查询java.util.Map的所有参数化实例

```
import java

from GenericInterface map,ParameterizedType pt
//hasQualifiedName指定package和class，配合谓语查询
where map.hasQualifiedName("java.util","Map") and pt.getSourceDeclaration() = map
select pt
```

部分泛型指定实例化类型，可以使用谓词getATypeBound绑定查询类型变量的类型

以下语句查询所有类型绑定为Number的所有类型参数

```
import java

from TypeBound tb,TypeVariable tv
where tb = tv.getTypeBound() and tb.getType().hasQualifiedName("java.lang","Number")
select tv
```

每个泛型类型都有一个没有任何类型参数的原始版本RawType表示，其具有子类RawClass和RawInterface，配合使用谓词getSourceDeclaration获取相应的泛型类型

以下语句查找原始类型为Map的变量

```
import java

from RawType rt,Variable v
where rt = v.getType() and rt.getSourceDeclaration().hasWualifiedName("java.util","Map")
select v
```

##### Variables

变量，表示类的成员字段Field、局部变量LocalVariableDecl、参数Parameter

##### 抽象语法树

表示AST的节点：语句Stmt、表达式Expr。两者都提供成员谓词

- Expr.getAChildExpr：返回给定表达式的子表达式
- Stmt.getAChild：返回直接嵌套在给定语句中的语句或表达式
- Expr.getParent：返回父节点
- Stmt.getParent：返回父节点

以下语句查询父项为return语句的所有表达式

```
import java

from Expr e
where e.getParent() instanceof ReturnStmt
select e
```

以下语句查询父项为if语句的所有表达式

```
import java

from Expr e
where e.getParent() instanceof IfStmt
select e
```

语句的父项不总是表达式，QL库提供两个抽象类ExprParent和StmtParent表示可能是任何表达式或语句父节点的节点

### Spring

Spring Security提供抽象类**WebSecurityConfigurerAdapter**实现授权，需要重写三个configure实现自定义认证和授权

```
// 自定义身份认证的逻辑
protected void configure(AuthenticationManagerBuilder auth) throws Exception { }

// 自定义全局安全过滤的逻辑
public void configure(WebSecurity web) throws Exception { }

// 自定义URL访问权限的逻辑
protected void configure(HttpSecurity http) throws Exception { }
```

其中用于处理身份认证的类是AuthenticationManagerBuilder，每一个HttpSecurity都要绑定一个AuthenticationManager

AuthenticationManagerBuilder用于创建AuthenticationManager

优先查看权限类--身份认证，确认是否存在默认账户

项目如下，存在默认账户

![image-20230201152126455](codeql学习.assets/image-20230201152126455.png)

CodeQL搜索，寻找继承了WebSecurityConfigurerAdapter的类且重写方法参数或内部new了AuthenticationManagerBuilder的类

```
import java
```

### sql

思路



### 