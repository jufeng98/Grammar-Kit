---

I. 一般注意事项
===============

1. 编写语法并不意味着生成的解析器会工作并产生良好的AST。
   棘手的部分是将一些 *看起来正确* 的原始语法 **调整** 为 *可工作* 的语法，即生成可工作解析器的语法。
   一旦你掌握了一些基础知识，剩下的就像将不同的块组合成一个工作解决方案一样简单。
2. 在编辑语法时，最好认为您在更高的抽象级别上操纵生成的代码。
3. 手写类和生成类应始终位于不同的源代码目录中。

---

II. 指南: 生成解析器
====================

2.1 解析器基础
--------------

每个规则要么匹配要么不匹配，因此每个BNF表达式都是布尔表达式。
**True** 表示输入序列的某些部分匹配，**false** *总是*表示即使输入的某些部分被匹配，也没有任何东西被匹配。
以下是一些语法到代码的映射：

顺序:

````
// rule ::= part1 part2 part3
public boolean rule() {
  <header>
  boolean result = false;
  result = part1();
  result = result && part2();
  result = result && part3();
  if (!result) <rollback any state changes>
  <footer>
  return result;
}
````

有序选择:

````
// rule ::= part1 | part2 | part3
public boolean rule() {
  <header>
  boolean result = false;
  result = part1();
  if (!result) result = part2();
  if (!result) result = part3();
  if (!result) <rollback any state changes>
  <footer>
  return result;
}
````

零个或多个构造：

````
// rule ::= part *
public boolean rule() {
  <header>
  while (true) {
    if (!part()) break;
  }
  <footer>
  return true;
}
````

相应地实现了 *一个或多个*、*可选*、*且谓语* 和 *非谓语* 构造。
如您所见，生成的代码可以像任何手写代码一样轻松调试。
像 *pin* 和 *recoverWhile* 这样的属性、规则修饰符为这个通用结构添加了一些行。

2.2 使用 *recoverWhile（错误恢复）* 属性
----------------------------------------

1. 在大多数情况下，应在循环内的规则上指定此属性
2. 该规则也应始终在某处具有 *pin*属性
3. 属性值应该是谓词规则，即保持输入不变

规则定义如下：

1. 属性规则照常处理
2. 无论结果如何，当谓词规则匹配时，解析器都将继续消费token

````
script ::= statement *
private statement ::= select_statement | delete_statement | ... {recoverWhile="statement_recover"}
private statement_recover ::= !(';' | SELECT | DELETE | ...)
select_statement ::= SELECT ... {pin=1}  // 锚点会匹配某些东西
                                         // pin="SELECT" 是一个有效的替代选项
````

2.3 当没有东西能提供帮助时: *external（外部）* 规则
---------------------------------------------------

1.有时在代码中做正确的事情更容易。
2.有时无法避免一些外部依赖。
3.有时我们已经在其他地方实现了一些逻辑。

````
{
  parserUtilClass="com.sample.SampleParserUtil"
}
external my_external_rule ::= parseMyExternalRule false 10  // 我们可以传递一些额外参数
                                                            // .. 甚至是其他规则!
rule ::= part1 my_external_rule part3
// rule ::= part1 <<parseMyExternalRule true 5>> part3      // 是一个有效的替代选项
````

````
public class SampleParserUtil {
  public static boolean parseMyExternalRule(PsiBuilder builder, int level,       // 必需参数
                                            boolean extraArg1, int extraArg2) {  // 额外参数
    // 开展工作，询问观众，给朋友打电话
  }
}
````

2.4 具有优先级的紧凑表达式解析
------------------------------

当涉及到表达式时，递归下降解析器在堆栈深度方面效率低下。
支持一种更自然、更紧凑的处理方式。

1. 所有 "表达式" 规则应该继承 "根表达式" 规则。
   如果操作正确，这将确保AST即使在出现错误的情况下也具有最佳的深度和一致性。
   由于 *extends* 属性语义，冗余节点将被折叠，并且根表达式规则节点将永远不会出现在AST中, 使用 *快速文档* (Ctrl-Q/Ctrl-J) 验证。
2. 优先级从上到下增加，有序选择语义得以保留
3. 对二进制和后缀表达式使用左递归
4. 使用 *private* 规则定义一组具有相同优先级的运算符
5. 当默认的左关联性不合适时，使用 *rightAssociative* 属性

以下代码片段演示了BNF "expression" 部分看起来很紧凑,
所描述的语法不会对普通的BNF语法造成太大破坏 ([完整示例](testData/generator/ExprParser.bnf)):

````
// 为了保留此示例，省略了短函数调用和其他表达式
{
  extends(".*expr")=expr
  tokens=[number="regexp:[0-9]+" id="regexp:[a-z][a-z0-9]*"]
}
// 根表达规则
expr ::= assign_expr
  | add_group
  | mul_group
  | unary_group
  | exp_expr
  | qualification_expr
  | primary_group

// 定义具有相同优先级的运算符的私有规则
private unary_group ::= unary_plus_expr | unary_min_expr
private mul_group ::= mul_expr | div_expr
private add_group ::= plus_expr | minus_expr
private primary_group ::= simple_ref_expr | literal_expr | paren_expr

// 每个表达式的公共规则
assign_expr ::= expr '=' expr { rightAssociative=true }
unary_min_expr ::= '-' expr
unary_plus_expr ::= '+' expr
div_expr ::= expr '/' expr
mul_expr ::= expr '*' expr
minus_expr ::= expr '-' expr
plus_expr ::= expr '+' expr
exp_expr ::= expr ('^' expr) + // N-ary 变体, "(<op> expr ) +" 语法是强制性的。
paren_expr ::= '(' expr ')'

// 使用@Nullable限定符getter引入伪规则
// 让标识符和简单的引用有其elementType
fake ref_expr ::= expr? '.' identifier
simple_ref_expr ::= identifier {extends=ref_expr elementType=ref_expr}
qualification_expr ::= expr '.' identifier {extends=ref_expr elementType=ref_expr}

literal_expr ::= number
identifier ::= id
````

注意:

1. *operator* 部分可以包含任何有效的BNF表达式并定义"tails", 即

   ````div_expr ::= expr [div_modifier | '*'] '/' expr div_expr_tail````
2. 可以使用特定的表达式规则代替 *expr* 来缩小解析范围
3. 语法中可以有任意数量的"根表达式"，只要它们不相交

所有操作信息都将出现在错误消息中。为了避免这种情况并提高性能，请添加以下内容：

````
{
   consumeTokenMethod(".*_expr|expr")="consumeTokenFast"
}
````

为此语法生成的解析器（这是对所描述的Pratt解析的过程性重写，[这里](http://javascript.crockford.com/tdop/tdop.html)不包括所有表达式的方法。
根规则只有两种方法。注释包括操作符优先级表：

````
  /* ********************************************************** */
  // 根表达式: expr
  // 操作符优先级表:
  // 0: BINARY(assign_expr)
  // 1: BINARY(plus_expr) BINARY(minus_expr)
  // 2: BINARY(mul_expr) BINARY(div_expr)
  // 3: PREFIX(unary_plus_expr) PREFIX(unary_min_expr)
  // 4: N_ARY(exp_expr)
  // 5: POSTFIX(ref_expr)
  // 6: ATOM(simple_ref_expr) ATOM(literal_expr) PREFIX(paren_expr)
  public static boolean expr(PsiBuilder builder_, int level_, int priority_) {
     // code to parse ATOM and PREFIX operators
     // .. and ..
     // call expr_0()
  }

  public static boolean expr_0(PsiBuilder builder_, int level_, int priority_) {
     // here goes priority-driven while loop for BINARY, N-ARY and POSTFIX operators
  }
````

---

III. 指南: 生成PSI类层次结构
============================

3.1 PSI 基础
------------

1. 若您不希望某条规则出现在AST中，请尽早为该规则添加 *private* 属性。第一条规则将隐式标记为 *private*。
2. 使用 "extends" 属性同时达到两个目的：一是让PSI层次结构更合理，二是使AST更扁平。

````
{
  extends(".*_expr")=expr               // 使AST的字面深度只有一级: FileNode/LiteralExpr
                                        //    否则就会看起来像这样: FileNode/Expr/LiteralExpr
  tokens = [
    PLUS='+'
    MINUS='-'
  ]
  ...
}
expr ::= factor add_expr *
private factor ::= primary mul_expr *   // 在 AST 中我们不需要这个节点
private primary ::= literal_expr        // .. 还有这个节点
left add_expr ::= ('+'|'-') factor      // 如果使用没有 "left" 规则的经典递归下降
left mul_expr ::= ('*'|'/') primary     //    那么没有 "extends" 的AST看起来会更糟
literal_expr ::= ...                    //    FileNode/Expr/AddExpr/MulExpr/LiteralExpr
````

3.2 使用 *fake（伪）* 规则和用户方法组织 PSI
--------------------------------------------

````
{
  extends("(add|mul)_expr")=binary_expr // 此属性可以直接放置在规则之后
  extends(".*_expr")=expr               // .. 但我更喜欢不那么杂乱的语法
}

// 将不会被解析器纳入考虑
fake binary_expr ::= expr + {
  methods=[
    left="/expr[0]"                     // 只要表达式中有 "+"，就会是@NotNull
    right="/expr[1]"                    // "expr" 是自动计算的子属性的名称（单数或列表）
  ]
}

expr ::= factor add_expr *
... "PSI 基础" 示例的剩余部分
````

将生成以下代码：

````
public interface BinaryExpr {
  List<Expr> getExprList();
  @NotNull
  Expr getLeft();
  @Nullable
  Expr getRight();
}

public interface AddExpr extends BinaryExpr { ... }

public interface MulExpr extends BinaryExpr { ... }

// PsiElementVisitor 实现
public class Visitor extends PsiElementVisitor {
  ...
  public visitAddExpr(AddExpr o) {
    visitBinaryExpr(o);
  }
  public visitMulExpr(MulExpr o) {
    visitBinaryExpr(o);
  }
  public visitBinaryExpr(BinaryExpr o) {
    visitExpr(o);
  }
  ...
}
````

3.3 通过实现 *mixin* 来实现接口
-------------------------------

````
{
  mixin("my_named")="com.sample.psi.impl.MyNamedImplMixin"
}
my_named ::= part1 part2 part3
// my_named ::= part1 part2 part3 {mixin="com.sample.psi.impl.MyNamedImplMixin"} // 也是有效的替代选项

````

````
public class MyNamedImplMixin extends MyNamed implements PsiNamedElement {
  // 来自 PsiNamedElement 接口的方法
  @Nullable @NonNls String getName() { ... }
  PsiElement setName(@NonNls @NotNull String name) throws IncorrectOperationException { ... }
}
````

3.4 通过方法注入实现接口
------------------------

````
{
  psiImplUtilClass="com.sample.SamplePsiImplUtil"
  implements("my_named")="com.intellij.psi.PsiNamedElement"
}
my_named ::= part1 part2 part3 {
  methods=[getName setName]             // 无需指定参数和返回类型
}
````

````
public class SamplePsiImplUtil {
  // methods from PsiNamedElement interface with an extra MyName parameter
  public static @Nullable String getName(MyNamed o) { ... }
  public static PsiElement setName(MyNamed o, @NotNull String name) throws IncorrectOperationException { ... }
}
````

3.5 存根索引支持
----------------

存根索引API强制对PSI类使用不同的规约：

* 应该有一个手动编写的所谓的 *stub*类
* PSI节点类型应扩展 _IStubElementType_(与通常的 _IElementType_相比)
* PSI接口应扩展 _StubBasedPsiElementBase&lt;StubClass&gt;_
* PSI实现类应扩展 _StubBasedPsiElementBase&lt;StubClass&gt;_

生成器不涵盖前两点。
_IStubElementType_ 以及 _stub_ 类本身都应该小心地手工实现。
_IStubElementType_ 值可以通过 _elementTypeFactory_ 属性提供给生成的解析器。

请注意，最后一点可能会破坏 _extends_ 属性所指示的PSI继承层次。

其余的问题可以通过两种方式解决。 直接 _implements/mixin_ 方法:

````
property ::= id '=' expr
  {
    implements=["com.sample.SampleElement" "com.intellij.psi.StubBasedPsiElement<com.sample.PropertyStub>"]
    mixin="com.sample.SampleStubElement<com.sample.PropertyStub>"

    // 最低需要:
    // implements=["com.intellij.psi.PsiElement" "com.intellij.psi.StubBasedPsiElement<com.sample.PropertyStub>"]
    // mixin="com.intellij.extapi.psi.StubBasedPsiElementBase<com.sample.PropertyStub>"
  }

````

_stubClass_-based 方法:

````
property ::= id '=' expr
  {
    stubClass="com.sample.PropertyStub"
  }

````

IV. 提示和技巧
==============

4.1. 结尾逗号
-------------

想象一种允许在列表中使用结尾逗号的语言，例如

```
( item1, item2, item3, )
```

在这种情况下，您可以使用这样的解析器规则：

```
element_list ::= '(' element (',' (element | &')'))* ')' {pin(".*")=1}
```

... 未完成

V. 独立使用POC
==============

本节仅为 **概念证明** 。下面描述的所有内容都不受支持，而且肯定已经过时。

Grammar Kit项目提供工件配置来构建`Grammar Kit.jar`和`expression console sample.jar`，
以及构建“light-psi-all.jar”的运行配置（在一个文件中仅包含所需的 *IntelliJ Platform* 类）。

为了启用最简的 `java -jar grammar-kit.jar ...` 命令需要的jar已经位于
[grammar-kit.jar!/META-INF/MANIFEST.MF](https://github.com/JetBrains/Grammar-Kit/blob/master/resources/META-INF/MANIFEST.MF#L3-L7),
所以既可以
**(1)** 在`grammar-kit.jar`旁边的 `lib` 子文件夹中提供原始的 *IntelliJ Platform* jar 或者
**(2)** `grammar-kit.jar` 旁边的`light-psi-all.jar`.

````
<dir>
├─ lib/<IntelliJ jars> (1)
├─ light-psi-all.jar   (2)
└─ grammar-kit.jar

// using java -jar
java -jar grammar-kit.jar <output-dir> <grammars-and-dirs>

// using java -cp
java -cp grammar-kit.jar;<all-the-needed-jars> org.intellij.grammar.Main <output-dir> <grammars-and-dirs>
````

查阅 [expression parser](grammars/ExprParser.bnf):

````
java -jar expression-console-sample.jar
````
