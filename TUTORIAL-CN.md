如何阅读语法
=======================

BNF 语法非常容易阅读。 仅需要把 *::=* 符号替换为 *是* 或者 *匹配*。
我们也可以很容易的将错误恢复属性包含在此表述中：
```
paren_expr ::= '(' expr ')' {pin=1}
```
可以这样阅读: _paren_expr_ 匹配认为是成功的即使只有 '(' 实际匹配。
最初，解析器会在任何失败时停止匹配。
接着当解析器无论如何都试图匹配序列的剩余部分时，idea 会进入 _extendedPin_ 模式 (默认打开)，
即，在输入 "( )" 的情况下，将匹配 ')' 标记。如果未达到 _pinned_ 部分，它仍然会在第一次失败时停止匹配。

因此，_pin_(锚点) 的概念有助于解析器在输入缺少某些部分时进行恢复。
```
property ::= id '=' expr  {pin=2 recoverWhile=rule_recover}
private rule_recover ::= !(';' | id '=')
```
可以这样阅读: _property_ 匹配序列 _id_, '=' 和 _expr_。
如果遇到 '=' 部分，匹配认为是成功的。
**无论** 结果如何，当 _rule_recover_ 匹配时，如果解析器没有遇到 ';' 或规则开始(id 和 '=')，则会跳过所有tokens。
请注意，恢复规则始终是 _谓语_（通常是NOT谓语），因此它不会从输入中消耗任何东西。
因此，_recoverWhile_ 的概念有助于解析器在输入包含意外内容时进行恢复。

实时预览介绍
=========================

假设我们想为某种表达式语言创建一个语法，如下所示：
````
expr=1 * 2 + (3 - 8.3!);
text='This is a ' + 'text';
// line comment

test_pin_results=;                        // 预期出现表达式
some garbage to test error recovering



recovered =1/2                            // 缺少分号
recovered_again=1/2;
````

为此，让我们创建一个新文件 _sample.bnf_。
我们可以通过上下文菜单或ctrl-alt-P/meta-alt-P快捷键调用 *实时预览* 菜单，并将示例文本粘贴进去。

当我们修改语法时，可以使用 _Structure_ 工具窗口、_File Structure_ 弹出窗口(ctrl-F12/meta-F12)和 _PSI 查看器_ 窗口来观察PSI树。

_启动/停止语法高亮显示_ 动作（ctrl-alt-F7/meta-alt-F7）在预览编辑器中高亮显示当前插入符号位置的语法表达式。

最后，我的IDE看起来是这样的：

![Live Preview](images/livePreview.png)


这是我为上面的示例设计的语法。没有java代码，没有生成，没有运行测试。
我仍然需要添加一个词法分析器和一些额外的属性来生成一个真正的类似解析器的包和一些类名称如[main readme](README-CN.md)中所述，但现在我确信BNF部分正常。

可以使用编辑器上下文菜单生成初始的 `*.flex` 文件和 `*.java` 词法分析器。

有趣的是，我甚至可以将这种语言嵌入到我使用的其他一些文件中，以快速测试语法。

总结
================

所描述的工作流程可以总结如下：

1. 通过 *实时预览* 设计语法原型
2. 生成初始的`*.flex` 到源码目录，并从中生成`*.java`词法分析器
3. 创建 *ParserDefinition* 然后配置词法器和解析器测试类
4. 在生产环境中分别完善 *.flex 和 *.bnf

*注意 1:* Flex 文件应手动编辑，因为它可能包含 `*.bnf` 中没有的复杂逻辑。
这也意味着 *实时预览* 在 (4) 中用处不大，因为它需要支持2个不同的词法分析器。

*注意 2:* *PsiBuilder* 会跳过在 *ParserDefinition* 中声明的空格和注释。
为了模拟这种行为，*实时预览* 将任何与 *regexp* token匹配的空格或新行视为空白，这在规则的任何地方都没有使用。


完整的 sample.bnf 内容:
````
{
  tokens=[
    SEMI=';'
    EQ='='
    LP='('
    RP=')'

    space='regexp:\s+'
    comment='regexp://.*'
    number='regexp:\d+(\.\d*)?'
    id='regexp:\p{Alpha}\w*'
    string="regexp:('([^'\\]|\\.)*'|\"([^\"\\]|\\.)*\")"

    op_1='+'
    op_2='-'
    op_3='*'
    op_4='/'
    op_5='!'
  ]

  name(".*expr")='expression'
  extends(".*expr")=expr
}

root ::= root_item *
private root_item ::= !<<eof>> property ';' {pin=1 recoverWhile=property_recover}

property ::= id '=' expr  {pin=2}
private property_recover ::= !(';' | id '=')

expr ::= factor plus_expr *
left plus_expr ::= plus_op factor
private plus_op ::= '+'|'-'
private factor ::= primary mul_expr *
left mul_expr  ::= mul_op primary
private mul_op ::= '*'|'/'
private primary ::= primary_inner factorial_expr ?
left factorial_expr ::= '!'
private primary_inner ::= literal_expr | ref_expr | paren_expr
paren_expr ::= '(' expr ')' {pin=1}
ref_expr ::= id
literal_expr ::= number | string | float
````

尝试使用 _pin_ 和 _recoverWhile_ 属性，tokens和规则修饰符，看看这一切是如何工作的。
