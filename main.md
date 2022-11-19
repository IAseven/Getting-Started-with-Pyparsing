## "Hello, World!"

Pyparsing附带了一些例子，包括一个基本的 "Hello, World!" 解析器。这个简单的例子也在[O'Reilly ONLamp.com](http://onlamp.com) 文章 [用Python构建递归下降解析器](http://www.onlamp.com/-pub/a/python/2006/01/26/pyparsing.html)。在本节中，我使用这个相同的例子来介绍 pyparsing 中的许多基本解析工具。

目前的 "Hello, World!"分析器只限于问候语的形式。

`word, word !`

这有点限制了我们的选择，所以让我们扩展语法以处理更复杂的问候语。比方说，我们想解析以下任何一种情况。

```
Hello, World!
Hi, Mom!
Good morning, Miss Crabtree!
Yo, Adrian!
Whattup, G?
How's it goin', Dude?
Hey, Jude!
Goodbye, Mr. Chips!
```

为这些字符串编写解析器的第一步是确定它们所遵循的模式。按照我们的最佳做法，我们把这种模式写成BNF。用普通话来描述问候语，我们会说："问候语是由一个或多个词组成的（这是敬语），后面是一个逗号，后面是一个或多个额外的词（这是问候语的主题，或问候对象），最后是一个感叹号或问号"。作为BNF，这个描述看起来像。

```
greeting ::= salutation comma greetee endpunc
salutation ::= word+
comma ::= ,
greetee ::= word+
word ::= a collection of one or more characters, which are any alpha or ' or .
endpunc ::= ! | ?
```

> 当然，写一个解析器来提取 "Hello, World!"中的成分是多余的。但希望通过扩展这个例子来实现一个通用的问候语解析器，我已经涵盖了大部分的解析基础知识。

这个BNF几乎可以直接翻译成pyparsing，使用基本的pyparsing元素Word、Literal、OneOrMore，以及辅助方法oneOf。(从BNF到pyparsing的一个翻译问题是，BNF是传统上对语法的 "自上而下 "的定义。而Pyparsing必须 "自下而上 "地构建它的语法。以确保引用的变量在被使用之前就被定义了）。
``` py
word = Word(alphas+"'.")
salutation = OneOrMore(word)
comma = Literal(",")
greetee = OneOrMore(word)
endpunc = oneOf("! ?")
greeting = salutation + comma + greetee + endpunc
```
oneOf是一个方便的快捷方式，用于定义一个字面替代物的列表。它的写法更简单。

`endpunc = oneOf("! ?")`

或者:

`endpunc = Literal("!") | Literal("?")`

你可以用一个项目的列表来调用oneOf，也可以用一个由空格分隔的项目的单一字符串来调用oneOf。

在这组样本字符串上使用我们的问候语分析器，可以得到以下结果。

```py
['Hello', ',', 'World', '!']
['Hi', ',', 'Mom', '!']
['Good', 'morning', ',', 'Miss', 'Crabtree', '!']
['Yo', ',', 'Adrian', '!']
['Whattup', ',', 'G', '?']
["How's", 'it', "goin'", ',', 'Dude', '?']
['Hey', ',', 'Jude', '!']
['Goodbye', ',', 'Mr.', 'Chips', '!']
```

所有的东西都能解析成单独的标记，但结果却没有什么结构。这个解析器，还有相当多的工作要做，以挑选出每个问候语字符串的重要部分。例如，为了识别构成问候语初始部分的标记--敬语--我们需要迭代结果，直到我们到达逗号标记。
``` py
for t in tests:
    results = greeting.parseString(t)
    salutation = []
    for token in results:
        if token == ",": break
        salutation.append(token)
    print salutation
```
讨厌！我们还不如一开始就写一个逐个字符的扫描器! 幸运的是，我们可以通过使我们的分析器更聪明一些来避免这种繁琐的工作。

由于我们知道问候语中的salutation和greetee部分是逻辑组，我们可以使用pyparsing的Group类来给返回的结果增加结构。通过将salutation和greetee的定义改为。
``` py
salutation = Group( OneOrMore(word) )
greetee = Group( OneOrMore(word) )
```
我们的结果开始看起来更有条理:
```py
[['Hello'], ',', ['World'], '!']
[['Hi'], ',', ['Mom'], '!']
[['Good', 'morning'], ',', ['Miss', 'Crabtree'], '!']
[['Yo'], ',', ['Adrian'], '!']
[['Whattup'], ',', ['G'], '?']
[["How's", 'it', "goin'"], ',', ['Dude'], '?']
[['Hey'], ',', ['Jude'], '!']
[['Goodbye'], ',', ['Mr.', 'Chips'], '!']
```
而我们可以使用基本的列表到变量的赋值来访问不同的部分。

请注意，我们不得不放入抓取变量dummy来处理解析后的逗号字符。在解析过程中，逗号是一个非常重要的元素，因为它显示了解析器在哪里停止读取敬语并开始读取问候语。但是在返回的结果中，逗号其实一点都不有趣，如果能从返回的结果中抑制它就更好了。你可以通过将逗号的定义包裹在一个pyparsing Suppress实例中来实现。

`comma = Suppress( Literal(",") )`

实际上，pyparsing中内置了许多快捷方式，由于这个函数非常普遍，以下任何一种形式都能完成同样的事情。
```py
comma = Suppress( Literal(",") )
comma = Literal(",").suppress()
comma = Suppress(",")
```
使用这些形式中的一种来抑制解析的逗号，我们的结果被进一步清理且可以阅读。
```py
[['Hello'], ['World'], '!']
[['Hi'], ['Mom'], '!']
[['Good', 'morning'], ['Miss', 'Crabtree'], '!']k
[['Yo'], ['Adrian'], '!']
[['Whattup'], ['G'], '?']
[["How's", 'it', "goin'"], ['Dude'], '?']
[['Hey'], ['Jude'], '!']
[['Goodbye'], ['Mr.', 'Chips'], '!']
```
现在，处理结果的代码可以去掉令人心烦的虚拟变量，而直接使用。
```py
for t in tests:
    salutation, greetee, endpunc = greeting.parseString(t)
```
现在我们有了一个像样的解析器和一个得到结果的好方法，我们可以开始享受测试数据的乐趣了。首先，让我们把敬语和问候语加入到它们自己的列表。
```py
salutes = []
greetees = []
for t in tests:
    salutation, greetee, endpunc = greeting.parseString(t)
    salutes.append( ( " ".join(salutation), endpunc) )
    greetees.append( " ".join(greetee) )
```
我还对解析后的标记做了一些其他改动。  
* 使用"".join(list)将分组后的代币转换为简单的字符串
* 在一个元组中保存了每个问候语的结尾标点，以区分感叹句和问题句。

现在我们已经收集了这些各种各样的名字和敬语，我们可以用它们来构思一些额外的、从未见过的问候和介绍。  
在导入随机模块后，我们可以合成一些新的问候语:
```py
for i in range(50):
    salute = random.choice( salutes )
    greetee = random.choice( greetees )
    print "%s, %s%s" % ( salute[0], greetee, salute[1] )
```
现在我们看到全新的一套问候语:
```
Hello, Miss Crabtree!
How's it goin', G?
Yo, Mr. Chips!
Whattup, World?
Good morning, Mr. Chips!
Goodbye, Jude!
Good morning, Miss Crabtree!
Hello, G!
...
```
我们也可以用以下代码模拟一些介绍语:
```py
for i in range(50):
    print '%s, say "%s" to %s.' % ( random.choice( greetees ),
    "".join( random.choice( salutes ) ),
    random.choice( greetees ) )
```
现在，鸡尾酒会开始转入高速发展阶段!
```
Jude, say "Good morning!" to Mom.
G, say "Yo!" to Miss Crabtree.
Jude, say "Goodbye!" to World.
Adrian, say "Whattup?" to World.
Mom, say "Hello!" to Dude.
Mr. Chips, say "Good morning!" to Miss Crabtree.
Miss Crabtree, say "Hi!" to Adrian.
Adrian, say "Hey!" to Mr. Chips.
Mr. Chips, say "How's it goin'?" to Mom.
G, say "Whattup?" to Mom.
Dude, say "Hello!" to World.
```
so，现在我们已经对pyparsing模块有了一些乐趣。使用一些比较简单的pyparsing类和方法，我们已经准备好向世界说 "Whattup "了。

## 是什么让Pyparsing如此特别？
Pyparsing在设计时考虑到了一些特定的目标。这些目标是基于这样的前提：语法必须易于编写，易于理解，并能随着给定的分析器的分析需求的变化和扩大而适应。这些目标背后的意图是尽可能地简化解析器的设计任务，让pyparsing用户把注意力集中在解析器上，而不是被解析库或语法的机械性分心。本节的其余部分列出了Pyparsing禅的要点。 

+ **语法规范应该是Python程序中看起来很自然的一部分，易于阅读，并且是Python程序员熟悉的风格和格式** 

  * Pyparsing通过几种方式实现了这一点:
  * 使用运算符将解析器元素连接在一起。Python 对定义运算符函数的支持使我们能够超越标准的对象构造语法，我们可以组成解析表达式，自然地读取.   
    而不是这样:
    ```py
    streetAddress = And( [streetNumber, name,
                            Or( [Literal("Rd."), Literal("St.")] ) ] )
    ```
    我们可以这样写:
    ```py
    streetAddress = streetNumber + name + ( Literal("Rd.") | Literal("St.") )
    ```
  * pyparsing中的许多属性设置方法都返回self，因此这些方法中的几个可以被链在一起。这允许语法中的解析器元素更加自成一体。例如，一个常见的解析器表达式是对一个整数的定义，包括它的名字的说明，以及附加一个解析动作，将整数字符串转换为 Python int。使用属性，这看起来就像:
    ```
    integer = Word(nums)
    integer.Name = "integer"
    integer.ParseAction = lambda t: int(t[0])
    ```
    使用返回自身的属性设置器，可以将其改为:
    ```py
    integer = Word(nums).setName("integer").setParseAction(lambda t:int(t[0]))
    ```
+ **类名比专门的排版更容易阅读和理解**  
这可能是pyparsing与正则表达式以及基于正则表达式的解析工具的最明确的区别。在介绍中给出的IP地址和电话入门14个号码的例子暗示了这个想法，但是当正则表达式的控制字符也是要匹配的文本的一部分时，正则表达式就变得真正不可捉摸了。其结果是用反斜线来转义控制字符，并将其解释为输入文本的混杂物。下面是一个匹配简化的C函数调用的正则表达式，它被限制为接受零个或多个参数，这些参数可以是单词或整数。  
`(\w+)\((((\d+|\w+)(,(\d+|\w+))*)?)\)`

    * 要一眼看出哪些括号是分组运算符，哪些是要匹配的表达式的一部分并不容易。如果输入的文本包含 \ . , * 或 ? 字符，事情就变得更加复杂。同样表达式的pyparsing版本是:
        ```py
        Word(alphas) + "(" + Group( Optional(Word(nums)|Word(alphas) +
            ZeroOrMore("," + Word(nums) | Word(alphas))) ) + ")"
        ```
    * 在pyparsing版本中，分组和重复是明确的，容易阅读。事实上，这种x + ZeroOrMore(", "+x)的模式非常常见，有一个pyparsing的辅助方法delimitedList，可以发出这种表达式。使用delimitedList，我们的pyparsing演绎进一步简化为:  
        ```py
        Word(alphas)+ "(" + Group( Optional(delimitedList(Word(nums)|Word(alphas))) ) + ")"
        ```

+ **空白标记杂乱无章，分散了语法定义的注意力**

  * 除了 "特殊字符其实并不特殊 "的问题，正则表达式还必须明确指出输入文本中可以出现空白的地方。在这个C函数的例子中，正则表达式将匹配。  
    `abc(1,2,def,5)`  
    but would not match:  
    `abc(1, 2, def, 5)`  
  * 不幸的是，要预测这样一个表达式中可能出现或可能不出现的可选空格并不容易，所以必须在整个表达式中随意加入 \s* 表达式，从而进一步掩盖了真正的文本匹配意图:  
    `(\w+)\s*\(\s*(((\d+|\w+)(\s*,\s*(\d+|\w+))*)?)\s*\)`  
  * 相比之下，pyparsing默认跳过了分析器元素之间的空白，因此，这个相同的pyparsing表达式:  
    `Word(alphas)+ "(" + Group( Optional(delimitedList(Word(nums)|Word(alphas))) ) + ")"`  
    匹配列出的对abc函数的任何一个调用，没有任何额外的空白指示符.
  * 这个概念也适用于注释，它可以出现在源程序的任何地方。想象一下，试图匹配一个函数，在这个函数中，开发者插入了一个注释来记录参数列表中的每个参数。在pyparsing中，这可以通过以下代码完成:
    ```py
    cFunction = Word(alphas)+ "(" + \
        Group( Optional(delimitedList(Word(nums)|Word(alphas))) ) + ")"
    cFunction.ignore( cStyleComment )
    ```
+ **解析过程的结果应该不仅仅是代表一个嵌套的标记列表，特别是当语法变得复杂的时候**  
    * Pyparsing使用一个名为ParseR esults的类返回解析过程的结果。ParseResults将支持简单语法的基于列表的访问（例如使用[]、len、iter和slicing进行索引），但它也可以表示嵌套的结果，以及对结果中的命名字段进行口令式和对象属性式访问。解析我们的C函数例子的结果是:  
    `['abc', '(', ['1', '2', 'def', '5'], ')']`
    * 你可以看到，函数参数已经被收集到它们自己的子列表中，使得在后期分析中提取函数参数更加容易。如果语法定义包括结果名称，那么可以通过名称而不是通过容易出错的列表索引来访问特定字段。这些更高层次的访问技术对于理解复杂语法的结果至关重要.


+ **解析时间是进行额外文本处理的好时机** 
    * 在解析时，解析器对输入文本中的字段的格式进行了许多检查：测试数字字符串的有效性，或匹配标点符号的模式，如引号内的字符串。如果保留为字符串，解析后的代码将不得不重新检查这些字段，将其转换为Python的整数和字符串，并且在转换前可能不得不重复同样的验证测试.
    * Pyparsing支持定义解析时的回调（称为解析动作），你可以将其附加到语法中的单个表达式上。由于解析器在匹配各自的模式后立即调用这些函数，所以通常很少或不需要额外的验证。例如，为了从被解析的引号字符串的主体中提取字符串，一个简单的解析动作可以去除开头和结尾的引号，例如:  
        `quotedString.setParseAction( lambda t: t[0][1:−1] )`
    * 即可。没有必要测试前导和尾随的字符是否是引号--除非它们是引号，否则函数不会被调用.
    * Parse动作也可以用来执行额外的验证检查，比如测试一个匹配的词是否存在于有效词的列表中，如果不存在，则引发ParseException。Parse动作还可以返回一个构造的列表或应用对象，本质上是将输入文本编译成一系列可执行或可调用的用户对象。在用pyparsing设计解析器时，解析动作可以成为一个强大的工具.
+ **语法必须容忍变化，因为语法的发展或输入文本变得更具挑战性**  

    当你不得不写解析器时，不容易避免沮丧的死亡螺旋，这种情况很常见。一开始只是一个简单的模式匹配练习，后来会逐渐变得复杂和难以操作。输入的文本可能包含与模式不完全匹配的数据，但无论如何都是需要的，所以解析器要打一个小补丁以包括新的变化。或者，解析器为之编写的语言获得了一个新的语言语法补充。这种情况发生了几次之后，补丁开始妨碍原始模式的定义，进一步的补丁变得越来越困难。当一个新的变化发生在几个月左右的平静期之后，重新获得解析器的知识比预期的要长，这只是增加了挫折感。Pyparsing不能治愈这个问题，但它的语法定义技术和它在语法和分析器代码中培养的编码风格使许多问题变得简单。语法的个别元素可能是明确的，容易找到的，相应地也容易扩展或修改。下面是一个pyparsing用户发给我的一段精彩的话，他正在考虑为一个特别棘手的分析器写一个语法。"我可以写一个自定义的方法，但我过去的经验是，一旦我得到了基本的pyparsing语法，它就会变成更多的自我文档化，更容易维护/扩展."

## 从表格中解析数据--使用Parse Actions和ParseResults

作为我们的第一个例子，让我们看一下可能在数据文件中给出的一组简单的大学足球比赛的分数。每一行文字都给出了每场比赛的日期，然后是大学名称和每个学校的分数.
```
09/04/2004 Virginia 44 Temple 14
09/04/2004 LSU 22 Oregon State 21
09/09/2004 Troy State 24 Missouri 14
01/02/2003 Florida State 103 University of Miami 2
```
我们对这个数据的BNF是简单而干净的:
```
digit ::= '0'..'9'
alpha ::= 'A'..'Z' 'a'..'z'
date ::= digit+ '/' digit+ '/' digit+
schoolName ::= ( alpha+ )+
score ::= digit+
schoolAndScore ::= schoolName score
gameResult ::= date schoolAndScore schoolAndScore
```
我们通过将这些BNF定义转换为pyparsing类实例来开始建立我们的解析器。就像我们在扩展的 "Hello, World!"程序中所做的那样，我们将从定义基本构件开始，这些构件将在以后被组合成完整的语法:
```py
# nums and alphas are already defined by pyparsing
num = Word(nums)
date = num + "/" + num + "/" + num
schoolName = OneOrMore( Word(alphas) )
```
请注意，你可以使用+运算符来组合pyparsing表达式和字符串字面符号。使用这些基本元素，我们可以通过将它们组合成更大的表达式来完成语法的编写:
```py
score = Word(nums)
schoolAndScore = schoolName + score
gameResult = date + schoolAndScore + schoolAndScore
```
我们使用gameResult表达式来解析输入文本的每一行:

```py
tests = """\
    09/04/2004 Virginia 44 Temple 14
    09/04/2004 LSU 22 Oregon State 21
    09/09/2004 Troy State 24 Missouri 14
    01/02/2003 Florida State 103 University of Miami 2""".splitlines()
for test in tests:
    stats = gameResult.parseString(test)
    print stats.asList()
```
就像我们在 "Hello, World!"解析器中看到的那样，我们从这个语法中得到一个非结构化的字符串列表:

```py
['09', '/', '04', '/', '2004', 'Virginia', '44', 'Temple', '14']
['09', '/', '04', '/', '2004', 'LSU', '22', 'Oregon', 'State', '21']
['09', '/', '09', '/', '2004', 'Troy', 'State', '24', 'Missouri', '14']
['01', '/', '02', '/', '2003', 'Florida', 'State', '103', 'University', 'of',
'Miami', '2']
```
我们要做的第一个改变是将date返回的标记组合成一个单一的MM/DD/YYYY日期字符串。pyparsing Combine类通过简单地包装组成的表达式来为我们做这件事:
```py
date = Combine( num + "/" + num + "/" + num )
```
有了这一改动，解析的结果变成了:
```py
['09/04/2004', 'Virginia', '44', 'Temple', '14']
['09/04/2004', 'LSU', '22', 'Oregon', 'State', '21']
['09/09/2004', 'Troy', 'State', '24', 'Missouri', '14']
['01/02/2003', 'Florida', 'State', '103', 'University', 'of', 'Miami', '2']
```
Combine实际上为我们执行了两项任务。除了将匹配的标记串联成一个单一的字符串外，它还强制要求这些标记在传入的文本中是相邻的.  

接下来要做的改变是把学校名称也合并起来。因为Combine的默认行为要求标记是相邻的，我们不会使用它，因为有些学校名称有嵌入的空格。相反，我们将定义一个例程，在解析时运行，将标记连接起来并作为一个单一的字符串返回。如前所述，这种例程在pyparsing中被称为解析动作，它们在解析过程中可以执行各种功能.

在这个例子中，我们将定义一个解析动作，它接收被解析的标记，使用字符串连接函数，并返回连接后的字符串。这是一个非常简单的解析动作，它可以被写成一个 Python lambda。通过调用 setParseAction，解析动作被挂到一个特定的表达式上，如:
```py
schoolName.setParseAction( lambda tokens: " ".join(tokens) )
```
解析动作的另一个常见用途是做额外的语义验证，超出表达式中定义的基本语法匹配。例如，date的表达式将接受03023/808098/29921作为有效日期，这当然是不可取的。一个验证输入日期的解析动作可以使用time.strptime将时间字符串解析为实际的日期:
```py
time.strptime(tokens[0],"%m/%d/%Y")
```
If strptime fails, then it will raise a ValueError exception. Pyparsing uses its own exception class, ParseException, for signaling whether an expression matched or not. Parse actions can raise their own exceptions to indicate that, even though the syntax matched, some higher-level validation failed. Our validation parse action would look like this:
```py
def validateDateString(tokens):
    try:
        time.strptime(tokens[0], "%m/%d/%Y")
    except ValueError,ve:
        raise ParseException("Invalid date string (%s)" % tokens[0])
date.setParseAction(validateDateString)
```
如果我们将输入的第一行中的日期改为19/04/2004，就会出现异常情况:
```py
pyparsing.ParseException: Invalid date string (19/04/2004) (at char 0), (line:1, col:1)
```
解析结果的另一个修改器是 pyparsing Group 类。Group并不改变被解析的标记；相反，它将它们嵌套在一个子列表中。Group是一个有用的类，可以为解析返回的结果提供结构:
```py
score = Word(nums)
schoolAndScore = Group( schoolName + score )
```
通过分组和连接，解析后的结果现在被结构化为字符串的嵌套列表:
```py
['09/04/2004', ['Virginia', '44'], ['Temple', '14']]
['09/04/2004', ['LSU', '22'], ['Oregon State', '21']]
['09/09/2004', ['Troy State', '24'], ['Missouri', '14']]
['01/02/2003', ['Florida State', '103'], ['University of Miami', '2']]
```
最后，我们将再添加一个解析动作来执行数字字符串到实际整数的转换。这是解析动作的一个非常常见的用途，它也显示了pyparsing如何能够返回结构化的数据，而不仅仅是解析字符串的嵌套列表。这个解析动作也很简单，可以实现为一个lambda:
```py
score = Word(nums).setParseAction( lambda tokens : int(tokens[0]) )
```
再一次，我们可以定义我们的解析动作来执行这种转换，而不需要在int的参数不是有效的整数字符串的情况下进行错误处理。这个lambda唯一被调用的时间是与pyparsing表达式Word (nums)相匹配的字符串，这保证了只有有效的数字字符串会被传递给解析动作.

我们解析的结果开始看起来像真正的数据库记录或对象了:
```py
['09/04/2004', ['Virginia', 44], ['Temple', 14]]
['09/04/2004', ['LSU', 22], ['Oregon State', 21]]
['09/09/2004', ['Troy State', 24], ['Missouri', 14]]
['01/02/2003', ['Florida State', 103], ['University of Miami', 2]]
```
在这一点上，返回的数据经过了结构化和转换，所以我们可以对数据做一些实际的处理，比如按日期列出比赛结果并标记获胜的球队。从parse String传回的ParseResults对象允许我们使用嵌套列表符号对解析后的数据进行索引，但是对于具有这种结构的数据，事情很快就会变得很糟糕:
```py
for test in tests:
    stats = gameResult.parseString(test)
    if stats[1][1] != stats[2][1]:
        if stats[1][1] > stats[2][1]:
            result = "won by " + stats[1][0]
        else:
            result = "won by " + stats[2][0]
    else:
        result = "tied"
    print("%s %s(%d) %s(%d), %s" % (stats[0], stats[1][0], 
          stats[1][1], stats[2][0], stats[2][1], result))
```
索引不仅使代码难以遵循（而且容易出错！），对解析后的数据的处理对结果中事物的顺序非常敏感。如果我们的语法包括一些可选字段，我们就必须包括其他逻辑来测试这些字段是否存在，并相应地调整索引。这使得解析器变得非常脆弱.

我们可以尝试使用多变量赋值来减少索引，就像我们在'"Hello, World!" on Steroids!:
```py
for test in tests:
    stats = gameResult.parseString(test)
    gamedate,team1,team2 = stats # <- assign parsed bits to individual variable names
    if team1[1] != team2[1]:
        if team1[1] > team2[1]:
            result = "won by " + team1[0]
        else:
            result = "won by " + team2[0]
    else:
        result = "tied"
    print("%s %s(%d) %s(%d), %s" % (gamedate, team1[0], 
          team1[1], team2[0], team2[1], result))
```
> **最佳实践: 使用结果名称**  
> 使用结果名称来简化对解析结果中特定标记的访问，并保护你的解析器不受后来文本和语法变化的影响，以及不受可选数据字段的影响。.

但这仍然使我们对解析的数据的顺序敏感.

相反，我们可以在语法中定义不同表达式应该使用的名称，以标记这些表达式返回的结果标记。要做到这一点，我们在语法中插入对 setResults-Name 的调用，这样表达式就会在它们被累积到整个语法的 Parse-Results 中时给标记.
```py
schoolAndScore = Group(
    schoolName.setResultsName("school") +
    score.setResultsName("score") )
gameResult = date.setResultsName("date") + schoolAndScore.setResultsName("team1") + schoolAndScore.setResultsName("team2")
```
而处理结果的代码更具有可读性:
```py
if stats.team1.score != stats.team2.score:
    if stats.team1.score > stats.team2.score:
        result = "won by " + stats.team1.school
    else:
        result = "won by " + stats.team2.school
else:
    result = "tied"
print "%s %s(%d) %s(%d), %s" % (stats.date, stats.team1.school, stats.team1.score, stats.team2.school, stats.team2.score, result)
```
这段代码有一个额外的好处，就是能够通过名称而不是索引来引用单个标记，使处理代码不受标记顺序的变化和可选数据字段的存在/不存在的影响。.

用结果名称创建ParseResults将使你能够使用字典式语义来访问标记。例如，你可以使用ParseResults对象向带有标签字段的插值字符串提供数据值，进一步简化输出代码:
```py
print "%(date)s %(team1)s %(team2)s" % stats
```
这就得出了以下结果:
```py
09/04/2004 ['Virginia', 44] ['Temple', 14]
09/04/2004 ['LSU', 22] ['Oregon State', 21]
09/09/2004 ['Troy State', 24] ['Missouri', 14]
01/02/2003 ['Florida State', 103] ['University of Miami', 2]
```
ParseResults还实现了keys()、items()和values()方法，并支持用Python的关键字进行调用。
> 即将到来的景点!
> 最新版本的Pyparsing（1.4.7）包括了一些符号，使其更容易向表达式添加结果名称，将本例中的语法代码减少为:  
>    ```py
>    schoolAndScore = Group( schoolName("school") + \
>           score("score") )
>    gameResult = date("date") + schoolAndScore("team1") + \
>           schoolAndScore("team2")
>    ```
> 现在没有任何借口不给你的解析结果命名了!

为了调试，你可以调用dump()来返回一个显示嵌套标记列表的字符串，然后是键和值的分层列表。下面是对第一行输入文本调用stats.dump()的一个例子:
```py
print stats.dump()

['09/04/2004', ['Virginia', 44],
['Temple', 14]]
- date: 09/04/2004
- team1: ['Virginia', 44]
 - school: Virginia
 - score: 44
- team2: ['Temple', 14]
 - school: Temple
 - score: 14
```
最后，你可以通过调用stats.asXML()并指定一个根元素名称来生成代表这个相同层次结构的XML。:
```py
print stats.asXML("GAME")
```
```xml
<GAME>
    <date>09/04/2004</date>
    <team1>
        <school>Virginia</school>
        <score>44</score>
    </team1>
    <team2>
        <school>Temple</school>
        <score>14</score>
    </team2>
</GAME>
```
还有最后一个问题要处理，与输入文本的验证有关。Pyparsing会解析一个语法，直到它到达语法的末尾，然后返回匹配的结果，即使输入字符串中有更多的文本。例如，这个语句:
```py
word = Word("A")
data = "AAA AA AAA BA AAA"
print OneOrMore(word).parseString(data)
```
将不会引发异常，而只是返回:
```py
['AAA', 'AA', 'AAA']
```
> 帮助性提示--用stringEnd结束你的语法
> 通过用stringEnd结束你的语法，或者将stringEnd附加到用于调用parse String的语法中，确保没有悬空的输入文本。如果该语法没有匹配所有给定的输入文本，它将在解析器停止解析的地方引发一个ParseException.

即使该字符串继续有更多的 "AAA "字样需要解析。很多时候，这个 "额外 "的文本实际上是更多的数据，但有一些不匹配，不能满足继续解析语法的要求.

为了检查你的语法是否已经处理了整个字符串，pyparsing 提供了一个 StringEnd 类（和一个内置表达式 stringEnd），你可以将其添加到语法的结尾。这是你表示 "在这一点上，我希望没有更多的文本--这应该是输入字符串的结束 "的方式。如果语法没有对输入的某些部分进行解析，那么 StringEnd 将引发一个 ParseException。注意，如果有尾部的空白，pyparsing会在测试字符串结束前自动跳过它。.

在我们目前的应用中，在解析表达式的末尾添加stringEnd可以防止意外地匹配
```
09/04/2004 LSU 2x2 Oregon State 21
as:
09/04/2004 ['LSU', 2] ['x', 2]
```
将其视为LSU和X学院之间的平局。 相反，我们得到一个ParseException，看起来像:
`pyparsing.ParseException: Expected stringEnd (at char 44), (line:1, col:45)`  
以下是解析器代码的完整列表:
```py
from pyparsing import Word, Group, Combine, Suppress, OneOrMore, alphas, nums,\
    alphanums, stringEnd, ParseException
import time


num = Word(nums)
date = Combine(num + "/" + num + "/" + num)


def validateDateString(tokens):
    try:
        time.strptime(tokens[0], "%m/%d/%Y")
    except ValueError, ve:
        raise ParseException("Invalid date string (%s)" % tokens[0])


date.setParseAction(validateDateString)
schoolName = OneOrMore(Word(alphas))
schoolName.setParseAction(lambda tokens: " ".join(tokens))
score = Word(nums).setParseAction(lambda tokens: int(tokens[0]))
schoolAndScore = Group(schoolName.setResultsName("school") +
                       score.setResultsName("score"))
gameResult = date.setResultsName("date") + schoolAndScore.setResultsName("team1") + \
    schoolAndScore.setResultsName("team2")
tests = """\
 09/04/2004 Virginia 44 Temple 14
 09/04/2004 LSU 22 Oregon State 21
 09/09/2004 Troy State 24 Missouri 14
 01/02/2003 Florida State 103 University of Miami 2""".splitlines()
for test in tests:
    stats = (gameResult + stringEnd).parseString(test)
    if stats.team1.score != stats.team2.score:
        if stats.team1.score > stats.team2.score:
            result = "won by " + stats.team1.school
        else:
            result = "won by " + stats.team2.school
    else:
        result = "tied"
    print "%s %s(%d) %s(%d), %s" % (stats.date, stats.team1.school, stats.team1.score,
                                    stats.team2.school, stats.team2.score, result)
    # or print one of these alternative formats
    # print "%(date)s %(team1)s %(team2)s" % stats
    # print stats.asXML("GAME")
```




## Extracting Data from a Web Page

## A Simple S-Expression Parser

## A Complete S-Expression Parser

## Parsing a Search String

## Search Engine in 100 Lines of Code

## Conclusion

## Index
