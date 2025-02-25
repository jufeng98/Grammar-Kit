
Grammar-Kit
===========
[![official project](http://jb.gg/badges/official.svg)](https://confluence.jetbrains.com/display/ALL/JetBrains+on+GitHub)
[![Build Status](https://teamcity.jetbrains.com/app/rest/builds/buildType:(id:IntellijIdeaPlugins_GrammarKit_Build)/statusIcon.svg?guest=1)](https://teamcity.jetbrains.com/viewType.html?buildTypeId=IntellijIdeaPlugins_GrammarKit_Build&guest=1)
[![GitHub license](https://img.shields.io/badge/license-Apache%20License%202.0-blue.svg?style=flat)](http://www.apache.org/licenses/LICENSE-2.0)
[![X Follow](https://img.shields.io/badge/follow-%40JBPlatform-1DA1F2?logo=x)][jb:x]
[![Slack](https://img.shields.io/badge/Slack-%23intellij--platform-blue?style=flat-square&logo=Slack)][jb:slack]

面向语言插件开发者的 [IntelliJ IDEA 插件](http://plugins.jetbrains.com/plugin/6606)。

添加BNF语法和JFlex文件编辑支持，以及解析器/PSI代码生成器。

快速链接: [最新开发构建](https://teamcity.jetbrains.com/guestAuth/app/rest/builds/buildType:IntellijIdeaPlugins_GrammarKit_Build,status:SUCCESS/artifacts/content/GrammarKit*.zip),
[变更日志](CHANGELOG.md), [教程](TUTORIAL-CN.md), [用法](HOWTO-CN.md)

> [!重要]
>
> 从 2022.3 版本起， Grammar-Kit 插件需要 Java 17。

使用Grammar Kit构建的开源插件：

* [Clojure-Kit](https://github.com/gregsh/Clojure-Kit), 
  [intellij-rust](https://github.com/intellij-rust/intellij-rust),
  [intellij-erlang](https://github.com/ignatov/intellij-erlang),
  [intellij-elm](https://github.com/intellij-elm/intellij-elm)
* [intellij-elixir](https://github.com/KronicDeth/intellij-elixir),
  [Perl5-IDEA](https://github.com/Camelcade/Perl5-IDEA),
  [Dart](https://github.com/JetBrains/intellij-plugins/tree/master/Dart), 
  [intellij-haxe](https://github.com/HaxeFoundation/intellij-haxe),
  [Cypher](https://github.com/neueda/jetbrains-plugin-graph-database-support)

另请参见 [自定义语言支持教程](https://plugins.jetbrains.com/docs/intellij/custom-language-support-tutorial.html).

一般使用说明
--------------------------
1. 创建 grammar \*.bnf 文件, 参见插件代码中的 [Grammar.bnf](grammars/Grammar.bnf)。
2. 使用“实时预览”调整语法结构视图（Ctrl-Alt-P / Cmd-Alt-P）
3. 生成 解析器/ElementTypes/PSI 类 (Ctrl-Shift-G / Cmd-Shift-G)
4. 生成lexer\*.flex文件，然后运行JFlex生成器（均通过上下文菜单） 
5. 实现ParserDefinition然后注册到plugin.xml
6. 将混合解析及其他复杂功能集成至PSI

与Gradle一起使用
-----------------

如上所述，从IDE调用解析器和生成器是首选方式。<br/>
否则，如果以下限制不重要，可使用[gradle-grammar-kit-plugin](https://github.com/JetBrains/gradle-grammar-kit-plugin)：
* 不支持方法混入（未实现两遍生成）
* 泛型签名与注解可能存在不准确性


插件功能
===============

![编辑器支持](images/editor.png)

* 重构: 提取规则 (Ctrl-Alt-M/Cmd-Alt-M)
* 重构: 引入 token (Ctrl-Alt-C/Cmd-Alt-C)
* 编辑: 执行 _choice_ 语法分支快速翻转(通过 Alt-Enter)
* 编辑: 表达式解构/移除操作 (Ctrl-Shift-Del/Cmd-Shift-Del)
* 导航: 语法与词法文件结构速览面板 (Ctrl-F12/Cmd-F12)
* 导航: 文件双向跳转 (解析器 and PSI) (Ctrl-Alt-Home/Cmd-Alt-Home)
* 导航: 在属性模式内跳转到匹配表达式 (Ctrl-B/Cmd-B)
* 高亮: 自定义颜色 (通过 Settings/Colors and Fonts)
* 高亮: 锚点表达式标识 (悬停工具提示当前锚点值逻辑上下文)
* 高亮: 语法检查, 可通过 Settings/Inspections 查看列表
* 文档: FIRST/FOLLOWS/PSI 内容规则文档悬浮提示 (Ctrl-Q/Cmd-J)
* 文档: 属性文档悬浮提示 (Ctrl-Q/Cmd-J)
* [实时预览](TUTORIAL-CN.md): 打开语言实时预览编辑器 (Ctrl-Alt-P/Cmd-Alt-P)
* [实时预览](TUTORIAL-CN.md): 在预览窗口编辑器启动/停止语法评估高亮显示 (Ctrl-Alt-F7/Cmd-Alt-F7)
* 生成器: 生成 解析器/PSI 代码 (Ctrl-Shift-G/Cmd-Shift-G)
* 生成器: 生成自定义 _parserUtilClass_ 类
* 生成器: 生成 \*.flex - JFlex 词法定义
* 生成器: 在 \*.flex 文件运行JFlex生成器
* 图表: PSI 树图 (需要 UML 插件)


语法概述
===============
基本语法查阅 [Parsing Expression Grammar (PEG)](http://en.wikipedia.org/wiki/Parsing_expression_grammar)。
使用 ::= 表示 ← 符号。您还可以使用[..]表示可选序列，使用{||}表示选择，因为这些变体在现实世界的语法中很受欢迎。
Grammar-Kit 源代码是 Grammar-Kit 应用的主要示例。
可以在[这里](grammars/Grammar.bnf)找到BNF解析器和PSI生成的语法。

以下是它可能看起来的样子：

````
// 基本 PEG BNF 语法

root_rule ::= rule_A rule_B rule_C rule_D                // 序列表达式
rule_A ::= token | 'or_text' | "another_one"             // 选择表达式
rule_B ::= [ optional_token ] and_another_one?           // 可选表达式
rule_C ::= &required !forbidden                          // 谓语表达式
rule_D ::= { can_use_braces + (and_parens) * }           // 分组和重复

// Grammar-Kit BNF 语法

{ generate=[psi="no"] }                                  // 顶层全局属性
private left rule_with_modifier ::= '+'                  // 规则修饰符
left rule_with_attributes ::= '?' {elementType=rule_D}   // 规则属性

private meta list ::= <<p>> (',' <<p>>) *                // 带有参数的元规则
private list_usage ::= <<list rule_D>>                   // 应用元规则
````

基本语法扩展了全局属性和规则属性。
属性由放在大括号内的 *name=value* 指定。
规则属性直接放置在规则定义之后。
全局属性放置在规则定义的顶部或用分号与规则定义分隔。

生成器为每个BNF表达式生成一个静态方法，如下所示：
```
static boolean rule_name(..)               // 规则顶层表达式
static boolean rule_name_0(..)             // 规则子表达式
...                                        // ...
static boolean rule_name_N1_N2_..._NX      // 规则 子-子-...-子-表达式
```
像 *rule_name_N1_N2_..._NX* 之类的规则命名应当避免。

可以在全局属性块中一次为多个规则指定一个属性：

````
{
  extends(".*_expr")=expr        // 应用到所有 .*_expr 规则
  pin(".*_list(?:_\d+)*")=1      // 应用到所有 .*_list 规则和他们的子表达式
}
````

### 规则修饰符:

1. *private* (PSI 树): 跳过当前节点创建，使其子节点包含在其父节点中 
2. *left* (PSI 树):  取左侧的AST节点（前一个兄弟节点），并通过成为其父节点将其括起来。 
3. *inner* (PSI 树):  取左侧的AST节点（前一个兄弟节点），并通过成为其子节点将自己注入其中。
4. *upper* (PSI 树):  取父节点，并通过采用其所有子节点来替换它。

5. *meta* (解析器):  参数化规则；其解析函数可以接受其他解析函数作为参数。
6. *external* (解析器):  具有手写解析功能的规则；不生成解析代码。
   
7. *fake* (PSI 类):  用于对生成的PSI类进行整形的规则；仅生成PSI类。

修饰符可以组合使用, *inner* 只应该和 *left* 一起使用，
*private left* 等同 *private left inner*， 
*fake* 不应该和 *private* 组合使用。

默认情况下, 规则是 *public*, 即 *non-private*, *non-fake*, 等等。

### 元规则和外部表达式：
外部表达式 *<< ... >>* 只是外部规则的内联变体。它还可以用于指定元规则和参数。

例如:

````
meta comma_separated_list ::= <<param>> ( ',' <<param>> ) *
option_list ::= <<comma_separated_list (OPTION1 | OPTION2 | OPTION3)>>
````

外部规则表达式语法与外部表达式体相同：

````
 external manually_parsed_rule ::= methodName param1 param2 ...
````

外部表达式和外部规则对双引号和单引号字符串的解释不同。
通常，在规则或方法名称之后出现在外部表达式中的任何内容都会被处理
作为参数，按“原样”传递，但单引号字符串除外，这些字符串首先会去掉引号。
这有助于传递限定的枚举常量、java表达式等。

参数列表中的规则引用实现为 [GeneratedParserUtilBase.Parser](src/org/intellij/grammar/parser/GeneratedParserUtilBase.java) 实例.

### Tokens:
显式token通过 _tokens_ 全局属性声明，例如以 *token_name=token_value* 形式。
token名称是IElementType token常量的名称，token值通常是用单引号或双引号表示的字符串。

语法中的token可以通过名称或单引号或双引号中的值来引用。
建议在可能的情况下使用值，以提高可读性。
当存在与某些规则匹配的未加引号的令牌值时，可以使用名称来解决冲突。

隐式token是指未通过 _tokens_ 属性指定的token。
无引号隐式token（也称为关键字token）的名称等于其值。
有引号的隐式token（也称为文本匹配token）速度较慢，因为它们是由文本匹配的，而不是由词法分析器返回的IElementType常量匹配的。
文本匹配的token可以跨越词法分析器返回的多个真实token。

规则、token和文本匹配token具有不同的颜色。

### 错误恢复和报告的属性：
* _pin_ （锚点，值：数字或模式）调整解析器以处理不完整的匹配。
如果序列的前缀与锚点项匹配，则序列匹配。
成功到达锚点项后，解析器会尝试匹配其余项，无论它们是否匹配。
Pin值通过数字 *{Pin=2}* 或模式 *{Pin="rule_B"}* 表示所需的锚点项。
默认情况下，锚点应用于顶部序列表达式。可以使用目标模式包含子表达式：
*{pin(".\*")=1}* 应用到所有子表达式。

* _recoverWhile_ (错误恢复，值: 谓语规则) 匹配规则后的任意数量token，匹配以任何结果完成。
此属性有助于解析器在遇到不匹配token序列时进行恢复。
查阅 [用法部分](HOWTO.md#22-using-recoverwhile-attribute)了解更多。

* _name_ (值: 字符串) 为规则指定在错误报告中使用的名称。
例如 *name("_.*expr")=expression* 将表达式错误信息修改为 "&lt;expression&gt; required" 而不是一串很长的token列表。

### 生成的解析器结构:
生成器可以将解析器代码拆分为几个类，以更好地支持大型语法。

在简单的情况下，解析器将仅由几个生成的类组成。

实际的错误恢复和报告代码以及基于解析器的补全提供者支持代码和基本token匹配代码位于_parserUtilClass_类中。 
可以通过指定一些其他继承或模仿原始[GeneratedParserUtilBase](src/org/intellij/grammar/parser/GeneratedParserUtilBase.java)的类来修改它。
无需在项目中保留 GeneratedParserUtilBase 的副本, 自12.1版本起已包含在*IntelliJ Platform*中。

手动解析的代码，即 _external_ rules，必须采用和生成相同的方式实现，
通过 _parserUtilClass_ 类中的静态方法或任何其他通过 _parserImports_ 属性导入的类，就像这样：
````
{
  parserImports=["static org.sample.ManualParsing.*"]
}
````

### 词法器 and PSI:
词法分析器必须识别并返回解析器和生成器生成的IElementType常量。
基于JFlex的词法分析器可以从定义了所有必需token的语法中生成（*Generate JFlex lexer*菜单）。

*在\*.flex文件中运行JFlex Generator*菜单，调用JFlex生成lexer java代码。
关键字是从用法中直接选择的，而*string*、*identifier*和*comment*等token可以这样定义 (来自 [教程](TUTORIAL-CN.md)):

````
{
  tokens=[
    ...
    comment='regexp://.*'
    number='regexp:\d+(\.\d*)?'
    id='regexp:\p{Alpha}\w*'
    string="regexp:('([^'\\]|\\.)*'|\"([^\"\\]|\\.)*\")"
    ...
  ]
  ...
}
````

虽然 *实时预览* 模式支持完整的Java RegExp语法，然而JFlex仅支持一个子集 (查阅 [JFlex documentation](http://jflex.de/manual.html#SECTION00053000000000000000))
Grammar-Kit 会试图执行一些明显的转换。

词法器可以单独提供，也可以使用生成的 \*.flex 文件作为基础。

默认情况下，解析器和生成器生成token类型常量和PSI。
这可以分别通过 *generateTokens* 和 *generatePSI* 全局布尔属性关闭。

*elementType* 规则属性允许混合生成的代码和一些现有的手工PSI。 

[jb:slack]: https://plugins.jetbrains.com/slack
[jb:x]: https://x.com/JBPlatform