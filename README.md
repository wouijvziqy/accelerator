# Accelerator

基于`Golang`批量分析Jar包，用于检测`Java`安全漏洞

这将比`Java ASM`更容易上手使用，且有更高的执行效率

该工具是一个辅助工具，可以帮助安全研究人员快速分析`jar`文件，尤其是对于一些闭源项目

使用此功能的优点是易于上手并检测效率较高（比编写`ASM`的`MethodVisitor`简单很多）

使用这种方法的缺点是不能进行太多定制操作，并且在多指令规则下可能会出现误报

## 快速开始

`accelerator`需要指定一个规则文件（默认：rule.txt）

并且需要输入`jar`文件的目录，然后`accelerator`会解压该目录中所有`jar`并扫描其中的`class`文件

```shell
./accelerator -rule your_rule_file -jars your_jar_dir
```

目前编写规则仅支持`INVOKE`相关的指令（实际上这也是最主要的指令）

todo: 删除具体的INVOKE指令，因为实际情况下容易确定是VIRTUAL或者SPECIAL

### 单指令规则

单指令规则一般用来确定哪些方法中存在危险调用

```text
INVOKEVIRTUAL ... *
```

如果是单一的`INVOKE`指令，对某方法的指令集进行检测的过程中只要匹配到则认为成功

通常方法的`desc`属性并不容易记，所以可以使用通配符`*`来代替该位置

### 多指令规则

多指令规则将会有更大的用处

todo: 目前仅支持顺序的且一定出现的指令，实际上可以加入“非”条件进行进一步判断

```text
INVOKEVIRTUAL [first rule] *
INVOKEVIRTUAL [next rule] *
...
```

支持多条`INVOKE`指令的规则，将会按照顺序进行匹配

**无论中间夹杂多少其他指令**只要按照顺序可以匹配到每一条指令，则认为成功

## 工作原理

（1） 解压缩`jar`文件以获取所有`class`文件

（2） 根据`Oracle Java`规范对`class`文件进行解析

（3） 解析所有类的方法区域中的方法以获得指令集（在`attr`属性中）

（4） 通过寻找常量池来改进指令内容（方法名和字符串索引等）

（5） 解析用户规则并将其与当前方法的指令集匹配

## 使用示例

原生的SQL注入规则（拼接字符串）

字符串的拼接在`JVM`中会优化为`StringBuilder.append`方法，如果某方法调用了该方法且存在`SQL`注入的`execute`等方法则有风险

```text
INVOKEVIRTUAL java/lang/StringBuilder.append *
INVOKEINTERFACE java/sql/Statement.executeQuery *
```

基于`JdbcTemplate`的SQL注入规则（拼接字符串）

原理同上，不过执行`SQL`语句的是`JdbcTemplate`类

```text
INVOKEVIRTUAL java/lang/StringBuilder.append *
INVOKEVIRTUAL org/springframework/jdbc/core/JdbcTemplate.query *
```

简单的`RCE`检测
```text
INVOKEVIRTUAL java/lang/Runtime.exec *
```

简单的`RCE`检测（由于字符串拼接导致的命令注入）

原理同上，执行的命令中有拼接字符串的风险

```text
INVOKEVIRTUAL java/lang/StringBuilder.append *
INVOKEVIRTUAL java/lang/Runtime.exec *
```

一些`SSRF`规则
- INVOKEVIRTUAL java/net/URL.openConnection *
- INVOKEVIRTUAL org/apache/http/impl/client/CloseableHttpClient.execute *
- INVOKEINTERFACE okhttp3/Call.execute *

## 实战

### log4j-core-2.14.0.jar

定位目标`Jar`中可能存在的`JNDI`注入

Rule
```text
INVOKEINTERFACE javax/naming/Context.lookup *
```

Result
```text
org/apache/logging/log4j/core/net/JndiManager lookup
```

### spring-cloud-gateway-server-3.0.6.jar

寻找目标`Jar`中可能执行的`SPEL`表达式

Rule
```text
INVOKEINTERFACE org/springframework/expression/Expression.getValue *
```

Result
```text
org/springframework/cloud/gateway/discovery/DiscoveryClientRouteDefinitionLocator buildRouteDefinition
org/springframework/cloud/gateway/discovery/DiscoveryClientRouteDefinitionLocator getValueFromExpr
org/springframework/cloud/gateway/discovery/DiscoveryClientRouteDefinitionLocator lambda$getRouteDefinitions$2
org/springframework/cloud/gateway/support/ShortcutConfigurable getValue
```

进一步定位目标`SPEL`表达式是否是基于`StandardEvaluationContext`的，此类存在`RCE`风险

Rule
```text
INVOKESPECIAL org/springframework/expression/spel/support/StandardEvaluationContext.<init> *
INVOKEVIRTUAL org/springframework/expression/spel/standard/SpelExpressionParser.parseExpression *
INVOKEINTERFACE org/springframework/expression/Expression.getValue *
```

Result
```text
org/springframework/cloud/gateway/support/ShortcutConfigurable getValue
```

### spring-cloud-function-context-3.1.0.jar

该案列原理同上

Rule
```text
INVOKEINTERFACE org/springframework/expression/Expression.getValue *
```

Result
```text
org/springframework/cloud/function/context/catalog/SimpleFunctionRegistry$FunctionInvocationWrapper parseMultipleValueArguments
org/springframework/cloud/function/context/config/RoutingFunction functionFromExpression
```

Rule
```text
INVOKESPECIAL org/springframework/expression/spel/support/StandardEvaluationContext.<init> *
```

Result
```text
org/springframework/cloud/function/context/config/RoutingFunction <init>
```

## 参考

[Java Class File Format](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html)

[JVM Instruction Set](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html)