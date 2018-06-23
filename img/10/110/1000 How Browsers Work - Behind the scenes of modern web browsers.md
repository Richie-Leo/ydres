原文：Tali Garsiel and Paul Irish，[How Browsers Work: Behind the scenes of modern web browsers](https://www.html5rocks.com/en/tutorials/internals/howbrowserswork/)，[中文版](https://www.html5rocks.com/zh/tutorials/internals/howbrowserswork/)

### 介绍 Introduction

浏览器很可能是使用最广的软件。在这篇入门文章中，我会介绍它的幕后工作原理。我们会了解到，从您在地址栏输入google.com直到您在浏览器屏幕上看到Google首页的整个过程中都发生了些什么。

#### 我们要讨论的浏览器 The browsers we will talk about

目前桌面端有五种主流浏览器：IE、Firefox、Safari、Chrome和Opera。手机端主要有Android、iPhone、Opera Mini、Opera Mobile、UC、Nokia S40/S60和Chrome，除Opera之外全部基于Webkit内核。根据[StatCounter统计](http://gs.statcounter.com/)（2013年5月），桌面浏览器中Chrome、Firefox和Safari占比71%，手机端Android、iPhone和Chrome占比54%，因此我将围绕Firefox、Chrome和Safari等开源浏览器（Safari部分开源）展开讨论。

#### 浏览器的主要功能 The browser's main functionality

浏览器的主要功能就是向服务器发出请求，在浏览器窗口中展示您选择的网络资源。这里所说的资源一般是指HTML文档，也可以是PDF、图片或其他的类型。资源的位置由用户使用URI（统一资源标示符）指定。

浏览器解释并显示HTML文件的方式是在HTML和CSS规范中指定的。这些规范由网络标准化组织W3C（万维网联盟）进行维护。 
多年以来，各浏览器都没有完全遵从这些规范，同时还在开发自己独有的扩展程序，这给网页开发者带来严重的兼容性问题。如今，大多数浏览器几乎都较好地遵循规范。

浏览器的用户界面有很多彼此相同的元素，其中包括：
- 用来输入URI的地址栏
- 前进和后退按钮
- 书签设置选项
- 用于刷新和停止加载当前文档的刷新和停止按钮
- 用于返回主页的主页按钮

奇怪的是，关于浏览器的用户界面并没有任何正式规范，这是多年来的最佳实践自然发展以及彼此之间相互模仿的结果。HTML5 也没有定义浏览器必须具有的用户界面元素，但列出了一些通用的元素，例如地址栏、状态栏和工具栏等。当然，各浏览器也可以有自己独特的功能，比如Firefox的下载管理器。

#### 浏览器的主要结构 The browser's high level structure

浏览器的主要组件包括（[1.1](https://www.html5rocks.com/en/tutorials/internals/howbrowserswork/#1_1)）：
1. **用户界面 User Interface** － 包括地址栏、前进/后退按钮、书签菜单等。除了浏览器主窗口中显示的页面内容外，其他显示的各个部分都属于用户界面。
2. **浏览器引擎 Browser Engine** － 管理用户界面和渲染引擎之间的操作。
3. **渲染引擎 Rendering Engine** － 负责显示请求的内容，如果请求的内容是HTML，它就负责解析HTML和CSS内容，并将解析后的内容显示在屏幕上。
4. **网络 Networking** － 用于网络调用，比如HTTP请求。其接口与平台无关，并为所有平台提供底层实现。
5. **用户界面后端 UI Backend** － 用于绘制基础的小部件，比如组合框和窗口。其公开了与平台无关的通用接口，而在底层使用操作系统的用户界面方法。
6. **Javascript解释器 Javascript Interpreter** － 用来解释、执行Javascript代码。
7. **数据存储 Data Storage** － 属于持久层，浏览器需要保存cookie等各种数据，同时支持HTML5规范中的LocalStorage、IndexedDB、WebSQL和FileSystem等存储机制。

<center><img src="https://github.com/Richie-Leo/ydres/blob/master/img/10/110/1000/01-layers.png?raw=true" /><br />图1：浏览器主要组件</center>

值得注意的是，和大多数浏览器不同，Chrome每个标签页分别对应一个渲染引擎实例。每个标签页都是一个独立的进程。

### 渲染引擎 The rendering engine

渲染引擎的作用嘛...当然就是“渲染”了，也就是在浏览器的屏幕上显示请求的内容。

默认情况下，渲染引擎可显示HTML和XML文档与图片。通过插件（或浏览器扩展程序），还可以显示其他类型的内容；例如，使用PDF查看器插件就能显示PDF文档。但是在本章中，我们将集中介绍其主要用途：显示使用CSS格式化的HTML内容和图片。

#### 各种渲染引擎 Rendering engines

不同浏览器使用的渲染引擎不一样，IE使用Trident，FireFox使用Gecko，Safari使用WebKit，Chrome和Opera（版本15开始）使用派生自WebKit的Blink。

Webkit是为Linux平台研发开源渲染引擎，后来由Apple移植到Mac及Windows上，详细内容参考[webkit.org](http://webkit.org/)。

#### 渲染主流程 The main flow

渲染引擎首先通过网络层获得文档的内容，通常以8K分块的方式完成。

下面是渲染引擎在取得内容之后的基本流程：
<center><img src="https://github.com/Richie-Leo/ydres/blob/master/img/10/110/1000/02-flow.png?raw=true" /><br />图2：渲染引擎基本流程</center>

渲染引擎开始解析HTML，并将标签转化为内容树中的DOM节点。同时也会解析外部CSS文件以及样式元素中的样式数据。HTML中这些带有视觉指令的样式信息将用于创建另一个树结构：==Render Tree - 渲染树==。

渲染树包含多个带有视觉属性（如颜色和尺寸）的矩形。这些矩形的排列顺序就是它们在屏幕上显示的顺序。

渲染树构建完毕之后，进入==Layout - 布局==阶段，也就是为每个节点分配一个应出现在屏幕上的确切坐标。下一个阶段是==Paint - 绘制==，渲染引擎会遍历渲染树，由用户界面后端层将每个节点绘制出来。

需要着重指出的是，这是一个渐进的过程。为达到更好的用户体验，渲染引擎会力求尽快将内容显示在屏幕上。它不必等到整个HTML文档解析完毕，就会开始构建渲染树和设置布局。在不断接收和处理来自网络的其它内容时，渲染引擎会先将部分内容解析并显示出来。

#### 渲染主流程示例 Main flow examples
<center><img src="https://github.com/Richie-Leo/ydres/blob/master/img/10/110/1000/03-webkitflow.png?raw=true" /><br />图3：webkit主流程<br /><br /></center>

<center><img src="https://github.com/Richie-Leo/ydres/blob/master/img/10/110/1000/04-gecko-flow.jpg?raw=true" /><br />图4：Mozilla的Geoko渲染引擎主流程</center>

从图3和图4可以看出，虽然WebKit和Gecko使用的术语略有不同，但整体流程是基本相同的。

Gecko将格式化的可视元素组成的树称为==Frame Tree - 框架树==。每个元素都是一个==Frame - 框架==。WebKit使用的术语是==Render Tree - 渲染树==，它由==Render Objects - 渲染对象==组成。对于元素的放置，WebKit使用的术语是==Layout - 布局==，而Gecko称之为==Reflow - 重排==。对于连接DOM节点和可视化信息从而创建呈现树的过程，WebKit使用的术语是==Attachment - 附加==。有一个细微的非语义差别，就是Gecko在HTML与DOM树之间还有一个称为==Content Sink - 内容槽==的层，用于生成DOM元素。我们会逐一论述流程中的每一部分：

### 解析与DOM树构建 Parsing and DOM tree construction

#### 解析综述 Parsing－general

解析是渲染引擎中非常重要的一个环节，因此我们要更深入地讲解。首先，来介绍一下解析。

解析文档是指将文档转化成为有意义的结构，也就是可让代码理解和使用的结构。解析得到的结果通常是代表了文档结构的节点树，它称作==Parse Tree - 解析树==或者==Syntax Tree - 语法树==。

示例 - 解析`2 + 3 - 1`这个表达式，会返回下面的树：

<center><img src="https://github.com/Richie-Leo/ydres/blob/master/img/10/110/1000/05-mathematical-expression-tree.png?raw=true" /><br />图5：数学表达式树节点</center>

##### 语法 Grammars

解析是以文档所遵循的语法规则（编写文档所用的语言或格式）为基础的，所有可以解析的格式都必须符合确定性文法（由词汇和语法规则构成），这称为上下文无关文法。人类语言并不属于这样的语言，因此无法用常规的解析技术进行解析。

##### 解析器和词法分析器的组合 Parser–Lexer combination

解析过程分成两个子过程：词法分析和语法分析。

词法分析是将输入内容分割成大量==Tokens - 标记==的过程。标记是语言中的词汇，即构成内容的单元。在人类语言中，它相当于字典中的单词。

语法分析是应用语言的语法规则的过程。

解析器通常将解析工作分给以下两个组件来处理：==Lexer - 词法分析器==（有时也称为==Tokenizer - 分词器==），负责将输入内容分解成一个个有效标记；而==Parser - 解析器==负责根据语言的语法规则分析文档的结构，从而构建==Parse Tree - 解析树==。词法分析器知道如何将无关的字符（比如空格和换行符）分离出来。

<center><img src="https://github.com/Richie-Leo/ydres/blob/master/img/10/110/1000/06-source-doc-to-parse-tree.png?raw=true" /><br />图6：从源文档到解析树</center>

解析是一个迭代的过程。通常，解析器会向词法分析器请求一个新标记，并尝试将其与某条语法规则进行匹配。如果发现了匹配规则，解析器会将一个对应于该标记的节点添加到解析树中，然后继续请求下一个标记。

如果没有规则可以匹配，解析器就会将标记存储到内部，并继续请求标记，直至找到可与所有内部存储的标记匹配的规则。如果找不到任何匹配规则，解析器就会引发一个异常。这意味着文档无效，包含语法错误。

##### 翻译 Translation

很多时候，解析树还不是最终产品。解析通常是在翻译过程中使用的，而翻译是指将输入文档转换成另一种格式。编译就是这样一个例子。编译器可将源代码编译成机器代码，具体过程是首先将源代码解析成解析树，然后将解析树翻译成机器码文档。

<center><img src="https://github.com/Richie-Leo/ydres/blob/master/img/10/110/1000/07-compilation-flow.png?raw=true" /><br />图7：编译流程</center>

#### 解析示例 Parsing example

在图5中，我们通过一个数学表达式建立了解析树。现在，让我们试着定义一个简单的数学语言，用来演示解析的过程。

词汇：我们的语言可包含整数、加号和减号。

语法：
1. 构成语言的语法单位是==Expression - 表达式==、==Term - 项==和==Operation - 运算符==。
2. 我们用的语言可以包含任意数量的表达式。
3. 表达式的定义是：一个项，后面跟一个运算符，然后再跟一个项。
4. 运算符是加号或减号。
5. 项是一个整数或一个表达式。

让我们分析一下`2 + 3 - 1`。 
匹配语法规则的第一个子串是`2`，而根据第5条语法规则，这是一个项。匹配语法规则的第二个子串是`2 + 3`，而根据第3条规则*一个项接一个运算符，然后再接一个项*，这是一个表达式。下一个匹配项已经到了输入的结束。`2 + 3 - 1`是一个表达式，因为我们已经知道`2 + 3`是一个项，这样就符合*一个项接一个运算符，然后再接一个项*的规则。`2 + +`不与任何规则匹配，因此是无效的输入。

##### 词汇表和语法的正式定义 Formal definitions for vocabulary and syntax

词汇表通常利用正则表达式来定义。

例如，我们的示例语言可以定义如下：

```
INTEGER: 0|[1-9][0-9]*
PLUS: +
MINUS: -
```

正如您所看到的，这里用正则表达式给出了整数的定义。

语法通常使用一种称为==BNF - 巴科斯范式==的格式来定义。我们的示例语言可以定义如下：

```
expression :=  term  operation  term
operation :=  PLUS | MINUS
term := INTEGER | expression
```

之前我们说过，如果语言的语法属于上下文无关文法，就可以由常规解析器进行解析。上下文无关文法的直观定义就是该文法可以完全通过BNF表达。正式定义参考[Wikipedia's article on Context-free grammar](http://en.wikipedia.org/wiki/Context-free_grammar)。

##### 解析器类型 Types of parsers

有两种基本类型的解析器：==Top Down Parsers - 自顶向下解析器==和==Bottom Up Parsers - 自底向上解析器==。直观地说，自顶向下解析器从语法的高层结构出发，尝试从中找到匹配的结构。而自底向上解析器从低层规则出发，将输入内容逐步转化为语法规则，直至满足高层规则。

让我们来看看这两种解析器会如何解析我们的示例：

自顶向下解析器会从高层的规则开始：首先将`2 + 3`标识为一个表达式，然后将`2 + 3 - 1`标识为一个表达式（标识表达式的过程涉及到匹配其他规则，但是起点是最高级别的规则）。

自底向上解析器将扫描输入内容，找到匹配的规则后，将匹配的输入内容替换成规则。如此继续替换，直到输入内容的结尾。部分匹配的表达式保存在解析器的堆栈中。

已匹配输入| 堆栈 | 未匹配输入
---|---|---
- | - | 2 + 3 - 1
2 | 条目 | + 3 - 1
2 + | 条目 操作符 | 3 - 1
2 + 3 | 表达式 | - 1
2 + 3 - | 表达式 操作符 | 1
2 + 3 - 1 | 表达式 | - 

自底向上解析器也称为==Shift-Reduce Parser - 移位归约解析器==，因为输入在向右移位（设想有一个指针从输入内容的开头移动到结尾），并且逐渐归约到语法规则上。

##### 自动生成解析器 Generating parsers automatically

有一些工具可以帮助您生成解析器，它们称为解析器生成器。您只要向其提供对应语言的文法（词汇和语法规则），它就会生成相应的解析器。创建解析器需要对解析有深刻理解，而人工创建并优化解析器并不是一件容易的事情，所以解析器生成器是非常实用的。

WebKit使用两个非常有名的解析器生成器组件：用于创建词法分析器的Flex以及用于创建解析器的Bison（您可能已经接触过Lex和Yacc）。Flex的输入是使用正则表达式定义的标记文件。Bison的输入是采用BNF格式的语法规则。

#### HTML解析器 HTML Parser

HTML解析器的工作是将HTML文档解析为解析树。

##### HTML文法定义 The HTML grammar definition

W3C组织制定[规范](https://www.html5rocks.com/en/tutorials/internals/howbrowserswork/#w3c)定义了HTML的词汇表和语法。

##### 非上下文无关文法 Not a context free grammar

我们在解析过程简介中已经了解到，上下文无关文法可以用BNF进行正式定义。

很遗憾，所有的常规解析器都不适用于HTML（当然我提出它们并不只是因为好玩，它们可以用于解析CSS和JavaScript）。HTML不能简单的用解析器所需的上下文无关文法来定义。

有一种可以定义HTML的正规格式：DTD（Document Type Definition，文档类型定义），但它不是上下文无关文法。

乍一看起来很奇怪：HTML和XML非常相似，有很多XML解析器可以使用。HTML存在一个XML变体(XHTML)，那么有什么大的区别呢？

区别在于HTML的处理更为宽容，它允许忽略一些特定标签，有时可以省略开始或结束标签。总的来说，它是一种软语法（soft grammar），而XML语法是严格的。

显然，这种看上去细微的差别实际上却带来了巨大的影响。一方面，这是HTML如此流行的原因：它能包容您的错误，简化网页开发。另一方面，这使得它很难编写正式的文法。概括地说，HTML无法很容易地通过常规解析器解析，也无法通过XML解析器来解析。

##### HTML DTD

HTML使用DTD进行定义，此格式可用于定义[SGML](http://en.wikipedia.org/wiki/Standard_Generalized_Markup_Language)族的语言，包括了对所有允许元素及它们的属性和层次关系的定义。如上文所述，HTML DTD不属于上下文无关文法。

DTD存在一些变体。严格模式完全遵守HTML规范，而其他模式可支持以前的浏览器所使用的标记。这样做的目的是确保向下兼容一些早期版本的内容。最新的严格模式DTD可以在这里找到：[www.w3.org/TR/html4/strict.dtd](http://www.w3.org/TR/html4/strict.dtd)

##### DOM

解析器输出解析树，它是由DOM元素和属性节点构成的树结构。DOM是文档对象模型 (Document Object Model) 的缩写，它是 HTML文档的对象表示，同时也是外部内容（例如 JavaScript）与HTML元素之间的接口。解析树的根节点是[`document`](http://www.w3.org/TR/1998/REC-DOM-Level-1-19981001/level-one-core.html#i-Document)对象。

DOM与标记之间几乎是一一对应的关系。比如下面这段标记：

```html
<html>
  <body>
    <p>
      Hello World
    </p>
    <div><img src="example.png" /></div>
  </body>
</html>
```

可翻译成如下的DOM树：：
<center><img src="https://github.com/Richie-Leo/ydres/blob/master/img/10/110/1000/08-example-dom-tree.png?raw=true" /><br />图8：示例标签对应的DOM树</center>

和HTML一样，DOM规范也是由W3C组织制定的，参考[www.w3.org/DOM/DOMTR](http://www.w3.org/DOM/DOMTR)，这是关于文档操作的通用规范。其中一个特定模块描述针对HTML元素，可以在[ www.w3.org/TR/2003/REC-DOM-Level-2-HTML-20030109/idl-definitions.html](http://www.w3.org/TR/2003/REC-DOM-Level-2-HTML-20030109/idl-definitions.htm)这里找到。

我所说的树包含DOM节点，指的是树节点由实现DOM接口的元素构成。浏览器在具体的实现中会有一些供内部使用的其他属性。

##### 解析算法 The parsing algorithm

我们在之前章节已经说过，HTML无法用常规的自顶向下或自底向上解析器进行解析。

原因在于：
1. 语言本身的宽容特性。
2. 浏览器对一些常见的非法HTML有容错机制。
3. 解析过程是往复的，通常源码不会在解析过程中发生改变，但在html中，脚本标签包含的`document.write`可能添加标签，这说明在解析过程中实际上修改了输入。

由于不能使用常规的解析技术，浏览器就创建了自定义的解析器来解析HTML。

HTML5规范详细地描述了[解析算法](http://www.whatwg.org/specs/web-apps/current-work/multipage/parsing.html)。此算法由两个阶段组成：==Tokenization - 分词==和==Tree Construction - 树构建==。

分词是词法分析过程，将输入内容解析成多个标记。HTML标记包括起始标签、结束标签、属性名称和属性值。

分词器识别标记，传递给树构造器，然后接受下一个字符以识别下一个标记；如此反复直到输入的结束。

<center><img src="https://github.com/Richie-Leo/ydres/blob/master/img/10/110/1000/09-html-parsing-flow.png?raw=true" /><br />图9：HTML解析流程</center>

##### 分词算法 The tokenization algorithm

该算法的输出结果是HTML标记。该算法使用状态机来表示。每一个状态接收来自输入流的一个或多个字符，并根据这些字符更新下一个状态。当前的分词状态及构建树状态共同影响结果，这意味着，即使接收的字符相同，对于下一个正确的状态也会产生不同的结果，具体取决于当前的状态。该算法相当复杂，无法在此详述，所以我们通过一个简单的示例来帮助大家理解其原理。

基本示例 - 将下面的HTML代码分词：

```html
<html>
  <body>
    Hello world
  </body>
</html>
```

初始状态为`Data State`，遇到字符`<`时状态转换为`Tag open state`，接收一个`a－z`的字符会创建一个`Start tag token`，状态转换为`Tag name state`，保持这个状态直到接收到字符`>`，在此期间接收的每个字符都会附加到这个标签名称上，本例中创建标签是`html`。

接收到`>`时，当前标签就结束了，状态回到`Data state`。接下来以同样的方式处理`<body>`标签，这样`html`和`body`标签都识别出来了。现在，回到了`Data state`状态，接收到`Hello world`中的字符`H`时，将生成一个字符标记，直到接收`</body>`中的`<`，我们将为`Hello world`中的每个字符都生成一个字符标记。

现在又回到了`Tag open state`，接收下一个字符`/`将创建一个`end tag`标记，状态转换为`Tag name state`，保持这一状态直到遇到`>`，产生一个新的标签标记，状态回到`Data state`。后面的`</html>`与`</body>`处理过程相同。

<center><img src="https://github.com/Richie-Leo/ydres/blob/master/img/10/110/1000/10-example-tokenizing.png?raw=true" /><br />图10：示例输入的分词（token-词条，tag-标签）</center>

##### 树的构建算法 Tree construction algorithm

创建解析器实例的同时也会创建Document对象，在树构建阶段，以Document为根节点的DOM树也会不断进行修改，向其中添加元素。树构建器接收分词器生成的每个节点并进行处理，规范中为每个HTML标记定义了对应的DOM元素，这些元素由树构建器创建并添加到DOM树中，同时还会添加到未闭合元素栈中，此栈用于纠正层级嵌套错误和处理未关闭的标签。其算法也可以用状态机来描述，这些状态称为“插入模式”。

让我们来看看示例输入的树构建过程：

```html
<html>
  <body>
    Hello world
  </body>
</html>
```

树构建阶段的输入是分词阶段生成的标记序列。首先是`initial mode`，接收`html`标记后转换为`before html`模式，对标记进行再处理，创建一个HTMLHtmlElement元素，将其附加到Document根对象上。

然后状态转换为`before head`，接收到`body`标记时，即使示例中没有`head`标记，也会隐式创建一个HTMLHeadElement元素并添加到树中。

现在转换到`in head`模式，然后是`after head`。系统对`body`标记重新处理，创建一个HTMLBodyElement并加入到树中，同时转换到`in body`模式。

接下来接收由`Hello world`字符串生成的一系列字符标记，接收第一个字符时会创建并插入一个text节点，其他字符附加到该节点。

接收到body结束标记时，转换为`after body`模式，接着接收到html结束标记，转换到`after after body`模式，接收到文件结束符后，整个解析过程结束。

<center><img src="https://github.com/Richie-Leo/ydres/blob/master/img/10/110/1000/11-example-tree-construction.gif?raw=true" /><br />图11：示例html树的构建过程</center>

##### 解析结束后的操作 Action when the parsing is finished

在此阶段，浏览器会将文档标注为交互状态，并开始解析那些处于延迟模式（deferred）的脚本，也就是那些应在文档解析完成后才执行的脚本。然后，文档状态将设置为complete，随后触发一个load事件。

完整的分词和构建树算法参考[HTML5规范](http://www.w3.org/TR/html5/syntax.html#html-parser)。

##### 浏览器的容错机制 Browsers error tolerance

您在浏览 HTML 网页时从来不会看到*语法无效*的错误。这是因为浏览器会纠正任何无效内容，然后继续工作。

以下面的 HTML 代码为例：

```html
<html>
  <mytag>
  </mytag>
  <div>
  <p>
  </div>
    Really lousy HTML
  </p>
</html>
```

这段HTML违反了很多语法规则（`mytag`不是标准标签，`p`和`div`元素之间的嵌套有误等等），但是浏览器仍然会正确地显示这些内容，并且毫无怨言。因为有大量的解析器代码会纠正HTML网页作者的错误。

不同浏览器的错误处理机制相当一致，但令人称奇的是，这并不是HTML当前规范的一部分。和书签管理以及前进/后退按钮一样，它也是浏览器在多年发展中的产物。很多网站都普遍存在着一些已知的无效HTML结构，每一种浏览器都会尝试通过和其他浏览器一样的方式来修复这些无效结构。

HTML5规范定义了一部分这样的要求。WebKit在HTML解析器类的开头注释中对此做了很好的概括。

> 解析器对分词输入内容进行解析，构建文档树。如果文档的格式正确，就直接进行解析。<br />
> 但不幸的是，我们必须处理很多格式不正确的HTML文档，浏览器必须能够容错。<br />
> 我们至少要注意下面几种错误情况。
> 1. 在未闭合的标签中添加明确禁止的元素。这种情况下，应该关闭所有标签，直到禁止添加该元素的标签为止，将禁止标签添加在它的后面。
> 2. 不能直接添加的元素。这很可能是网页作者忘记添加了其中的一些标记（或者其中的标记是可选的）。这些标签可能包括：HTML HEAD BODY TBODY TR TD LI（还有遗漏的吗？）。
> 3. 在一个==Inline - 行内元素==中添加==Block - 块级元素==。关闭所有行级元素，直到出现下一个较高级的块级元素。
> 4. 如果这样仍然无效，可关闭所有元素，直到可以添加元素为止，或者忽略该标记。

让我们看一些WebKit容错的示例：

###### 使用了`</br>`而不是`<br>`

一些网站使用`</br>`而不是`<br>`，为了兼容IE和Firefox，WebKit将其与`<br>`等同处理。

代码：
```c++
if (t->isCloseTag(brTag) && m_document->inCompatMode()) {
     reportError(MalformedBRError);
     t->beginTag = true;
}
```

###### 散乱的表格 A stray table

指一个表格嵌套在另一个表格中，但不在它的某个单元格内。

比如下面这个例子：

```html
<table>
    <table>
        <tr><td>inner table</td></tr>
    </table>
    <tr><td>outer table</td></tr>
</table>
```

WebKit会将其层次结构更改为两个同级表格：

```html
<table>
    <tr><td>outer table</td></tr>
</table>
<table>
    <tr><td>inner table</td></tr>
</table>
```

代码：

```c++
if (m_inStrayTableContent && localName == tableTag)
        popBlock(tableTag);
```

WebKit使用一个栈来保存当前元素内容，它会从外部表格的栈中弹出内部表格。这样，这两个表格就变成了同级关系。

###### 嵌套的表单元素 Nested form elements

如果用户在一个表单元素中又放入了另一个表单，那么第二个表单将被忽略。

代码：

```c++
if (!m_currentFormElement) {
        m_currentFormElement = new HTMLFormElement(formTag,    m_document);
}
```

###### 过深的标签嵌套 A too deep tag hierarchy

代码的注释已经说得很清楚了：
> 示例网站www.liceo.edu.mx标签嵌套达到了1500，内容嵌套在大量的`<b>`中。我们只允许最多20层相同标签的嵌套，如果再嵌套更多，就会全部忽略。

```c++
bool HTMLParser::allowNestedRedundantTag(const AtomicString& tagName)
{

unsigned i = 0;
for (HTMLStackElem* curr = m_blockStack;
         i < cMaxRedundantTagDepth && curr && curr->tagName == tagName;
     curr = curr->next, i++) { }
return i != cMaxRedundantTagDepth;
}
```

###### 放错位置的html或者body结束标记 Misplaced html or body end tags

同样，代码的注释已经说得很清楚了：
> 支持格式非常糟糕的 HTML代码。我们从不关闭body标记，因为一些愚蠢的网页会在实际文档结束之前就关闭。我们通过调用end()来执行关闭操作。

代码：
```c++
if (t->tagName == htmlTag || t->tagName == bodyTag )
return;
```

所以网页作者需要注意，除非您想作为反面教材出现在WebKit容错代码段的示例中，否则还请编写格式正确的HTML代码。

#### CSS解析 CSS parsing

还记得简介中解析的概念吗？和HTML不同，CSS是上下文无关文法，可以使用简介中描述的各种解析器进行解析。事实上，CSS 规范定义了[CSS的词法和语法](http://www.w3.org/TR/CSS2/grammar.html)。

让我们来看一些示例： 
词法（词汇表）用正则表达式定义每个标记：

```
comment   \/\*[^*]*\*+([^/*][^*]*\*+)*\/
num   [0-9]+|[0-9]*"."[0-9]+
nonascii  [\200-\377]
nmstart   [_a-z]|{nonascii}|{escape}
nmchar    [_a-z0-9-]|{nonascii}|{escape}
name    {nmchar}+
ident   {nmstart}{nmchar}*
```

`ident`是标识符 (identifier) 的缩写，比如类名。`name`是元素的ID（通过`#`来引用）。

语法用BNF进行描述：

```
ruleset
  : selector [ ',' S* selector ]*
    '{' S* declaration [ ';' S* declaration ]* '}' S*
  ;
selector
  : simple_selector [ combinator selector | S+ [ combinator? selector ]? ]?
  ;
simple_selector
  : element_name [ HASH | class | attrib | pseudo ]*
  | [ HASH | class | attrib | pseudo ]+
  ;
class
  : '.' IDENT
  ;
element_name
  : IDENT | '*'
  ;
attrib
  : '[' S* IDENT S* [ [ '=' | INCLUDES | DASHMATCH ] S*
    [ IDENT | STRING ] S* ] ']'
  ;
pseudo
  : ':' [ IDENT | FUNCTION S* [IDENT S*] ')' ]
  ;
```

解释：这是一个规则集结构示例：
```css
div.error, a.error {
  color:red;
  font-weight:bold;
}
```

div.error和a.error是选择器，大括号里面包的内容是这个规则集包含的规则。这个结构由下面语法正式定义：
```
ruleset
  : selector [ ',' S* selector ]*
    '{' S* declaration [ ';' S* declaration ]* '}' S*
  ;
```

它表示一个规则集具有一个或多个*选择器*，多个选择器永逗号和空格（S表示空格）分隔开。每个规则集包含大括号及大括号中的一条或多条分号隔开的*声明*。*声明*和*选择器*由后面的BNF格式定义。

##### Webkit CSS解析器 Webkit CSS parser

Webkit使用Flex和Bison解析器生成器，根据CSS语法文件自动生成CSS解析器。回忆一下解析器的介绍，Bison创建一个自底向上的解析器，Firefox使用人工编写的自顶向下解析器。它们都会将每个CSS文件解析成StyleSheet对象，每个对象包含CSS规则，CSS规则对象包含选择器和声明对象，以及其他一些CSS语法的对象。

<center><img src="https://github.com/Richie-Leo/ydres/blob/master/img/10/110/1000/12-parsing-css.png?raw=true" /><br />图12：解析css</center>

#### 处理脚本及样式表的顺序 The order of processing scripts and style sheets

##### 脚本 Scripts

网页脚本默认情况下是同步处理的。开发者希望解析器遇到`<script>`标记时立即解析并执行脚本，文档的解析将停止，直到脚本执行完毕。如果脚本是外部的，那么解析过程会停止，直到从网络同步获取到脚本资源后再继续。这个模式已经使用了多年，也在HTML4和HTML5规范中特别指定了。开发者也可以将脚本标注为`defer`，这样它就不会停止文档解析，而是等到解析结束才执行。HTML5增加了一个选项，可将脚本标记为异步，以便由其他线程解析和执行。

##### 预测解析 Speculative parsing

Webkit和Firefox都做了这个优化，当执行脚本时，另一个线程解析剩下的文档，并加载后面需要通过网络加载的资源。这种方式可以使资源并行加载从而使整体速度更快。需要注意的是，预解析并不改变DOM树，它将这个工作留给主解析过程，自己只解析外部引用的资源，比如外部脚本、样式表和图片。

##### 样式表 Style sheets

样式表采用另一种不同的模式。理论上，应用样式表不会更改DOM树，因此没有必要由于等待样式表而停止文档解析。但这涉及到一个问题，就是脚本在文档解析阶段会请求样式信息，如果当时还没有加载和解析样式，脚本就会得到错误的结果，这会产生很多问题。这看上去是一个非典型案例，但事实上非常普遍。Firefox在样式表加载和解析的过程中，会禁止所有脚本。而对于WebKit而言，仅当脚本尝试访问的样式属性可能受尚未加载的样式表影响时，它才会禁止该脚本。

### 渲染树构建 Render tree construction

在DOM树构建的同时，浏览器还会构建另一个树结构：==Render Tree - 渲染树==。这是由可视化元素按照其显示顺序而组成的树，也是文档的可视化表示。它的作用是便于按照正确的顺序绘制文档内容。

Firefox将渲染树中的元素称为==Frames - 框架==。WebKit使用术语==Renderer - 渲染器==或==Render Object - 渲染对象==。 <br />
渲染器知道如何布局并将自己及其子元素绘制出来。 <br />
WebKit的RenderObject类是所有渲染器的基类，其定义如下：

```c++
class RenderObject{
  virtual void layout();
  virtual void paint(PaintInfo);
  virtual void rect repaintRect();
  Node* node;  //the DOM node
  RenderStyle* style;  // the computed style
  RenderLayer* containgLayer; //the containing z-index layer
}
```

每一个渲染器都代表了一个矩形的区域，通常对应于相关节点的==CSS Box - CSS盒模型==，这一点在CSS2规范中有所描述。它包含诸如宽度、高度和位置等几何信息。<br /> 
Box的类型通过节点的display样式属性指定（请参阅样式计算章节）。下面的webkit代码说明了如何根据display属性决定某个节点创建何种类型的渲染器。

```c++
RenderObject* RenderObject::createObject(Node* node, RenderStyle* style)
{
    Document* doc = node->document();
    RenderArena* arena = doc->renderArena();
    ...
    RenderObject* o = 0;

    switch (style->display()) {
        case NONE:
            break;
        case INLINE:
            o = new (arena) RenderInline(node);
            break;
        case BLOCK:
            o = new (arena) RenderBlock(node);
            break;
        case INLINE_BLOCK:
            o = new (arena) RenderBlock(node);
            break;
        case LIST_ITEM:
            o = new (arena) RenderListItem(node);
            break;
       ...
    }

    return o;
}
```

元素类型也需要考虑，例如form表单控件和table表格都有特殊的框架。 <br />
在WebKit中，如果一个元素需要创建特殊的渲染器，就会重载createRenderer方法。

#### 渲染树和Dom树的关系 The render tree relation to the DOM tree

渲染器和DOM元素相对应，但并非一一对应，不可见的DOM元素不会加到渲染树中，例如head元素。如果元素的display属性值为none，那么也不会显示在渲染树中（但是visibility属性值为hidden的元素仍会显示）。

有一些DOM元素对应多个可视对象，它们一般是具有复杂结构的元素，无法用一个矩形来描述。例如，select元素有三个渲染器：一个用于显示区域，一个用于下拉列表框，还有一个用于按钮。如果由于宽度不够，文本无法在一行中显示而分为多行，那么新的行也会创建新的渲染器。<br />
另一个关于多渲染器的例子是格式无效的HTML。根据CSS规范，行级元素只能包含块级元素或行级元素中的一种。如果出现了混合内容，则应创建匿名的块级渲染器，以包裹行级元素。

一些渲染器在树上的位置与对应元素的DOM节点不同，例如浮动定位和绝对定位的元素，它们位于正常文档流之外，放置在树中的其他地方，并映射到真正的框架，而放在原位的是占位框架。

<center><img src="https://github.com/Richie-Leo/ydres/blob/master/img/10/110/1000/13-render-tree-and-dom-tree.png?raw=true" /><br />图13：渲染树和对应的Dom树</center>

#### 构建渲染树的流程 The flow of constructing the tree

Firefox中，系统将展示层注册为DOM更新的监听器，展示层将框架创建工作委托给FrameConstructor，由该构造器解析样式（参考样式计算章节）并创建框架。

WebKit中，解析样式和创建渲染器的过程称为附加。每个DOM节点都有一个attach方法。Attach过程是同步进行的，将节点插入DOM树时调用新的节点attach方法。

处理html和body标签时构建渲染树根节点。这个根节点渲染对象对应于CSS规范中所说的容器block，这是最上层的block，包含了其他所有block。它的尺寸就是viewport视口，即浏览器窗口显示区域的尺寸，Firefox称之为ViewPortFrame，而WebKit称之为RenderView。这就是Document对应的渲染对象。渲染树的其余部分在DOM树节点插入时构建。

参考[CSS2处理模型](http://www.w3.org/TR/CSS21/intro.html#processing-model)。

#### 样式计算 Style Computation

创建渲染树需要计算出每个渲染对象的可视属性，这可以通过计算每个元素的样式属性得到。

样式包括各种来源的样式表、行内样式元素和HTML中的可视化属性（例如bgcolor属性）。其中后者将经过转化以匹配CSS样式属性。

样式表的来源包括浏览器的默认样式表、由网页作者提供的样式表以及由浏览器用户提供的用户样式表（浏览器允许您定义自己喜欢的样式。以Firefox为例，用户可以将自己喜欢的样式表放在Firefox Profile文件夹下）。

样式计算存在以下难点：

1. 样式数据是一个超大的结构，存储了无数的样式属性，这可能造成内存问题。
2. 如果不进行优化，为每一个元素查找匹配的规则会造成性能问题。要为每一个元素遍历整个规则列表来寻找匹配规则，这是一项浩大的工程。选择器会具有很复杂的结构，这就会导致某个匹配过程一开始看起来很可能是正确的，但最终发现其实是徒劳的，必须尝试其他匹配路径。
   例如下面这个组合选择器：
   ```css
   div div div div｛…｝
   ```
   这意味着规则适用于3个div嵌套元素的子元素`<div>`。如果您要检查规则是否适用于某个特定的`<div>`元素，应沿选择树上的一条向上路径进行检查。您可能需要向上遍历节点树，结果发现只有两个div，规则并不适用于该元素。然后，您必须再次尝试树中的其它路径。
3. 应用规则涉及非常复杂的级联，它们定义了规则的层次。

让我们来看看浏览器是如何处理这些问题的：

##### 共享样式数据 Sharing style data

WebKit节点会引用样式对象(RenderStyle)。这些对象在某些情况下可以由不同节点共享。这些节点是同级关系，并且：

1. 这些元素必须处于相同的鼠标状态（例如，不允许其中一个是`:hover`状态，而另一个不是）
2. 任何元素都没有ID
3. 标签名必须匹配
4. class属性必须匹配
5. 对应的属性集合必须完全相同
6. 链接状态必须匹配
7. 焦点状态必须匹配
8. 任何元素都不应受属性选择器的影响，这里所说的影响是指在选择器中的任何位置有任何使用了属性选择器的选择器匹配
9. 元素中不能有任何行内样式属性
10. 不能使用任何同级选择器。WebCore在遇到任何同级选择器时，只会引发一个全局开关，并停用整个文档的样式共享（如果存在）。这包括 `+` 选择器以及 `:first-child` 和 `:last-child` 等选择器。

##### Firefox规则树 Firefox rule tree

为了简化样式计算，Firefox还采用了另外两种树：==Rule Tree - 规则树==和==Style Context Tree - 样式上下文树==。WebKit也有样式对象，但它们不是保存在类似样式上下文树这样的树结构中，只是由DOM节点指向此类对象的相关样式。

<center><img src="https://github.com/Richie-Leo/ydres/blob/master/img/10/110/1000/14-firefox-style-context-tree.png?raw=true" /><br />图14：Firefox样式上下文树</center>

样式上下文包含结果值。要计算出这些值，应按照正确顺序应用所有的匹配规则，并将其从逻辑值转化为具体的值。例如，如果逻辑值是屏幕大小的百分比，则需要换算成绝对单位。规则树的主意真的很巧妙，它使得节点之间可以共享这些值，以避免重复计算，还可以节约空间。

所有匹配的规则都存储在树中。路径中的底层节点拥有较高的优先级。规则树包含了所有已知规则匹配的路径。规则的存储是延迟进行的。规则树不会在开始的时候就为所有的节点进行计算，而是只有当某个节点样式需要进行计算时，才会计算路径并添加到规则树中。

这个想法相当于将规则树路径视为词典中的单词。如果我们已经计算出如下的规则树：

<center><img src="https://github.com/Richie-Leo/ydres/blob/master/img/10/110/1000/15-tree.png?raw=true" /></center>

假设我们需要为内容树中的另一个元素匹配规则，并且找到匹配路径是`B - E - I`（按照此顺序）。由于我们在树中已经计算出了路径`A - B - E - I - L`，因此就已经有了此路径，这就减少了现在所需的工作量。

让我们看看规则树如何帮助我们减少工作。

###### 结构划分 Division into structs

样式上下文可分割成多个结构。这些结构体包含了特定类别（如border或color）的样式信息。结构中的属性都是继承或非继承的。继承属性如果未由元素定义，则继承自其父代。非继承属性（也称为重置属性）如果未进行定义，则使用默认值。

规则树通过缓存整个结构（包含计算出的结果值）为我们提供帮助。这一想法假定底层节点没有提供结构的定义，则可使用上层节点中的缓存结构。

###### 使用规则树计算样式上下文 Computing the style contexts using the rule tree

在计算某个特定元素的样式上下文时，我们首先计算规则树中的对应路径，或者使用现有的路径。然后我们沿此路径应用规则，在新的样式上下文中填充结构。我们从路径中拥有最高优先级的底层节点（通常也是最特殊的选择器）开始，向上遍历规则树，直到结构填充完毕。如果该规则节点对于此结构没有任何规范，那么我们可以实现更好的优化：寻找路径更上层的节点，找到后指定完整的规范并指向相关节点即可。这是最好的优化方法，因为整个结构都能共享。这可以减少结果值的计算量并节约内存。<br /> 
如果我们找到了部分定义，就会向上遍历规则树，直到结构填充完毕。

如果我们找不到结构的任何定义，那么假如该结构是继承类型，我们会在上下文树中指向父元素的结构，这样也可以共享结构。如果是重置类型的结构，则会使用默认值。

如果最特殊的节点确实添加了值，那么我们需要另外进行一些计算，以便将这些值转化成实际值。然后我们将结果缓存在树节点中，供子代使用。

如果某个元素与其同级元素都指向同一个树节点，那么它们就可以共享整个样式上下文。

让我们来看一个例子，假设我们有如下HTML代码：

```html
<html>
  <body>
    <div class="err" id="div1">
      <p>
        this is a <span class="big"> big error </span>
        this is also a
        <span class="big"> very  big  error</span> error
      </p>
    </div>
    <div class="err" id="div2">another error</div>
  </body>
</html>
```

还有如下规则：

```css
div {margin:5px;color:black}
.err {color:red}
.big {margin-top:3px}
div span {margin-bottom:4px}
#div1 {color:blue}
#div2 {color:green}
```

为了简便起见，我们只需要填充两个结构：color结构和margin结构。color结构只包含一个成员（即color），而margin结构包含四条边。 <br />
形成的规则树如下图所示（节点的标记方式为`节点名 : 指向的规则序号`）：
<center><img src="https://github.com/Richie-Leo/ydres/blob/master/img/10/110/1000/16-rule-tree.png?raw=true" /></center>

上下文树如下图所示（`节点名 : 指向的规则节点`）：
<center><img src="https://github.com/Richie-Leo/ydres/blob/master/img/10/110/1000/17-context-tree.png?raw=true" /></center>

假设我们解析HTML时遇到了第二个`<div>`标记，我们需要为此节点创建样式上下文，并填充其样式结构。<br /> 
经过规则匹配，我们发现该`<div>`的匹配规则是第1、2和6条。这意味着规则树中已有一条路径可供我们的元素使用，我们只需要再为其添加一个节点以匹配第6条规则（规则树中的F节点）。<br /> 
我们将创建样式上下文并将其放入上下文树中。新的样式上下文将指向规则树中的F节点。

现在我们需要填充样式结构。首先要填充的是margin结构。由于最后的规则节点(F)并没有添加到margin结构，我们需要上溯规则树，直至找到在先前节点插入中计算过的缓存结构，然后使用该结构。我们会在指定margin规则的最上层节点（即B节点）上找到该结构。

我们已经有了color结构的定义，因此不能使用缓存的结构。由于color有一个属性，我们无需上溯规则树以填充其他属性。我们将计算端值（将字符串转化为RGB等）并在此节点上缓存经过计算的结构。

第二个`<span>`元素处理起来更加简单。我们将匹配规则，最终发现它和之前的span一样指向规则G。由于我们找到了指向同一节点的同级，就可以共享整个样式上下文了，只需指向之前span的上下文即可。

对于包含了继承自父代的规则的结构，缓存是在上下文树中进行的（事实上color属性是继承的，但是Firefox将其视为reset属性，并缓存到规则树上）。<br /> 
例如，如果我们在某个段落中添加font规则：

```css
p { font-family: Verdana; font-size: 10px; font-weight: bold }
```

那么这个p在内容树中的子节点div，会共享和它parent一样的font结构，这种情况发生在没有为这个div指定font规则时。

Webkit中，并没有规则树，匹配的声明会被遍历四次，先是应用非important的高优先级属性（之所以先应用这些属性，是因为其他的依赖于它们－比如display），其次是高优先级important的，接着是一般优先级非important的，最后是一般优先级important的规则。这样，出现多次的属性将被按照正确的级联顺序进行处理，最后一个生效。

总结一下，共享样式对象（结构中完整或部分内容）解决了问题1和3，Firefox的规则树帮助以正确的顺序应用规则。

##### 对规则进行处理以简化匹配过程 Manipulating the rules for an easy match

样式规则有几个来源：
- 外部样式表或style标签内的css规则
  ```css
  p {color: blue}
  ```
- 行内样式属性
  ```html
  <p style="color: blue" />
  ```
- html可视化属性（映射为相应的样式规则）
  ```html
  <p bgcolor="blue" />
  ```

后两种很容易和元素进行匹配，因为元素拥有样式属性，而且HTML属性可以使用元素作为键值进行映射。

我们之前在第2个问题中提到过，CSS规则匹配可能比较棘手。为了解决这一难题，可以对CSS规则进行一些处理，以便访问。

样式表解析完毕后，系统会根据选择器将CSS规则添加到某个哈希表中。这些哈希表的选择器各不相同，包括ID、类名称、标签名称等，还有一种通用哈希表，适合不属于上述类别的规则。如果选择器是ID，规则就会添加到ID映射表中；如果选择器是类名，规则就会添加到类名映射表中，依此类推。<br /> 
这种处理可以大大简化规则匹配。我们无需查看每一条声明，只要从哈希表中提取元素的相关规则即可。这种优化方法可排除掉95%以上规则，因此在匹配过程中根本就不用考虑这些规则了 ([4.1](https://www.html5rocks.com/zh/tutorials/internals/howbrowserswork/#4_1))。

我们以如下的样式规则为例：

```css
p.error {color: red}
# messageDiv {height: 50px}
div {margin: 5px}
```

第一条规则将插入类名映射表，第二条将插入ID映射表，而第三条将插入标签映射表。 <br />
对于下面的 HTML 代码段：
```html
<p class="error">an error occurred </p>
<div id=" messageDiv">this is a message</div>
```

我们首先找到p元素对应的规则，类名映射表将包含一个error的key，找到p.error的规则，div在ID映射表和标签映射表中都有相关的规则，剩下的工作就是找出这些由key对应的规则中哪些确实是正确匹配的。

例如，如果div的规则是
```css
table div {margin:5px}
```

这条规则仍然会从标记表中提取出来，因为键是最右边的选择器，但这条规则并不匹配我们的div元素，因为div没有table祖先。
WebKit和Firefox都进行了这一处理。

##### 以正确的级联顺序应用规则 Applying the rules in the correct cascade order

样式对象具有与每个可视化属性一一对应的属性（均为CSS属性但更为通用）。如果某个属性未由任何匹配规则所定义，那么部分属性就可由父代元素样式对象继承。其他属性具有默认值。

如果定义不止一个，就会出现问题，需要通过级联顺序来解决。

###### 样式表的级联顺序 Style sheet cascade order

某个样式属性的声明可能会出现在多个样式表中，也可能在同一个样式表中出现多次。这意味着应用规则的顺序极为重要。这称为层叠顺序。根据CSS2规范，层叠的顺序为（优先级从低到高）：

1. 浏览器声明
2. 用户声明
3. 作者的一般声明
4. 作者的important声明
5. 用户important声明

浏览器声明是重要程度最低的，而用户只有将该声明标记为important才可以替换网页作者的声明。同样顺序的声明会根据特异性进行排序，然后再是其指定顺序。HTML可视化属性会转换成匹配的CSS声明。它们被视为低优先级的网页作者规则。

###### 特异性 Specificity

选择器的特异性由[CSS2规范](http://www.w3.org/TR/CSS2/cascade.html#specificity)定义如下：

- 如果声明来自于style属性，而不是带有选择器的规则，则记为1，否则记为0 (= a)
- 计算选择器中id属性的数量（＝b）
- 计算选择器中class及伪类的数量（＝c）
- 计算选择器中元素名及伪元素的数量（＝d）

将四个数字按`a-b-c-d`这样连接起来（使用一个基数很大的计数系统），构成特异性。

您使用的进制取决于上述类别中的最高计数。 <br />
例如，如果a=14，您可以使用十六进制。如果a=17，那么您需要使用十七进制；当然不太可能出现这种情况，除非是存在如下的选择器：html body div div p ...（在选择器中出现了17个标记，这样的可能性极低）。

一些例子：

```css
 *             {}  /* a=0 b=0 c=0 d=0 -> specificity = 0,0,0,0 */
 li            {}  /* a=0 b=0 c=0 d=1 -> specificity = 0,0,0,1 */
 li:first-line {}  /* a=0 b=0 c=0 d=2 -> specificity = 0,0,0,2 */
 ul li         {}  /* a=0 b=0 c=0 d=2 -> specificity = 0,0,0,2 */
 ul ol+li      {}  /* a=0 b=0 c=0 d=3 -> specificity = 0,0,0,3 */
 h1 + *[rel=up]{}  /* a=0 b=0 c=1 d=1 -> specificity = 0,0,1,1 */
 ul ol li.red  {}  /* a=0 b=0 c=1 d=3 -> specificity = 0,0,1,3 */
 li.red.level  {}  /* a=0 b=0 c=2 d=1 -> specificity = 0,0,2,1 */
 #x34y         {}  /* a=0 b=1 c=0 d=0 -> specificity = 0,1,0,0 */
 style=""          /* a=1 b=0 c=0 d=0 -> specificity = 1,0,0,0 */
```

###### 规则排序 Sorting the rules

找到匹配的规则之后，应根据级联顺序将其排序。WebKit对于较小的列表会使用冒泡排序，而对较大的列表则使用归并排序。对于以下规则，WebKit通过重载`>`运算符来实现排序：

```c++
static bool operator >(CSSRuleData& r1, CSSRuleData& r2)
{
    int spec1 = r1.selector()->specificity();
    int spec2 = r2.selector()->specificity();
    return (spec1 == spec2) : r1.position() > r2.position() : spec1 > spec2;
}
```

#### 渐进式处理 Gradual process

WebKit使用一个标记来表示是否所有的顶级样式表（包括@imports）均已加载完毕。如果在attach过程中尚未完全加载样式，则使用占位符，并在文档中进行标注，等样式表加载完毕后再重新计算。

### 布局 Layout

当渲染对象被创建并添加到树中，它们并没有位置和大小，计算这些值的过程称为布局或==Reflow - 重排==。

Html使用基于流的布局模型，这意味着大多数情况下只要一次遍历就能计算出几何信息。处于流中靠后位置元素通常不会影响靠前位置元素的几何特征，因此布局可以按从左至右、从上至下的顺序遍历文档。但是也有例外情况，比如HTML表格的计算就需要不止一次的遍历 ([3.5](https://www.html5rocks.com/zh/tutorials/internals/howbrowserswork/#3_5))。

坐标系是相对于根框架而建立的，使用的是top坐标和left坐标。

布局是一个递归的过程。它从根渲染器（对应于HTML文档的`<html>`元素）开始，然后递归遍历部分或所有的框架层次结构，为每一个需要计算的渲染器计算几何信息。

根渲染器的位置左边是0,0，其尺寸为视口（也就是浏览器窗口的可见区域）。<br />
所有的渲染器都有一个layout或者reflow方法，每一个渲染器都会调用其需要进行布局的子元素的layout方法。

#### 脏位标记系统 Dirty bit system

浏览器使用脏位标记系统来避免每个细小变化都进行整体布局。改变或者添加了渲染对象，就将它和子元素标记为`脏`，表示需要布局。

有两种标记：`脏`和`子元素脏`，`子元素脏`标记说明即使这个渲染对象没有变化，但至少有一个子元素需要布局。

#### 全局和增量布局 Global and incremental layout

全局布局是指触发了整个渲染树范围的布局，触发原因可能包括：
1. 影响所有渲染器的全局样式更改，例如字体大小更改。
2. 窗口大小调整。

布局可以采用增量方式，也就是只对`脏`的渲染器进行布局（这样可能存在需要进行额外布局的弊端）。 
当渲染器为`脏`时，会异步触发增量布局。例如，当来自网络的额外内容添加到DOM树之后，新的渲染器附加到了渲染树中。

<center><img src="https://github.com/Richie-Leo/ydres/blob/master/img/10/110/1000/18-reflow.png?raw=true" /><br />图18：增量布局，仅标记为脏的渲染对象及其子元素重新布局</center>

#### 异步和同步布局 Asynchronous and Synchronous layout

增量布局的过程是异步的，Firefox为增量布局生成了reflow队列，以及一个调度执行这些批处理命令。WebKit也有一个计时器用来执行增量布局：遍历树，为dirty状态的渲染对象进行布局。

另外，当脚本请求样式信息时，例如offsetHeight，会同步的触发增量布局。

整体布局一般都是同步触发。

有些时候，layout会被作为一个初始布局之后的回调，比如滑动条的滑动。

#### 优化 Optimizations

当因为大小或者位置改变（并不是大小改变）而触发布局时，渲染对象的大小将会从缓存中读取，而不会重新计算。

一般情况下，如果只有子树发生改变，则布局并不从根开始。这种情况发生在，变化发生在元素自身并且不影响它周围元素，例如，将文本插入文本域（否则，每次击键都将触发从根开始的重排）。

#### 布局过程 The layout process

布局一般有下面这几个部分：
1. 父渲染对象决定它自己的宽度
2. 父渲染对象读取子元素，并：
   1. 设置子元素渲染对象（设置它的x和y）
   2. 在需要时（它们当前为dirty或是处于整体布局或者其他原因）调用子元素渲染对象的layout方法，这将计算子元素高度
3. 父渲染对象使用子元素渲染对象的累积高度，以及margin和padding的高度来设置自己的高度：这将被父渲染对象的父级使用
4. 将dirty标识设置为false

Firefox使用一个state对象（nsHTMLReflowState）做为参数去布局（firefox称为reflow），state包含父对象的宽度及其他内容。

Firefox布局的输出是一个metrics对象（nsHTMLReflowMetrics）。它包括渲染对象计算出的高度。

#### 宽度计算 Width calculation

渲染对象的宽度使用容器的宽度、渲染对象样式中的宽度及margin、border进行计算。例如，下面这个div的宽度：
```html
<div style="width: 30%"/>
```

webkit中宽度的计算过程是（RenderBox类的calcWidth方法）：
- 容器的宽度是容器的可用宽度和0中的最大值，这里的可用宽度为：
  ```
  contentWidth=clientWidth()-paddingLeft()-paddingRight()
  ```
  clientWidth和clientHeight代表一个对象内部的不包括border和滑动条的大小
- 元素的宽度指样式属性width的值，它可以通过计算容器的百分比得到一个绝对值
- 加上水平方向上的border和padding

这是最佳宽度的计算过程，接下来计算宽度的最大值和最小值。如果最佳宽度大于最大宽度则使用最大宽度，如果小于最小宽度则使用最小宽度。最后缓存这个值，当需要布局但宽度未改变时使用。

#### 换行 Line Breaking

当一个渲染对象在布局过程中需要换行时，则暂停并告诉它的父对象它需要换行，父对象将创建额外的渲染对象并调用它们的布局过程。

### 绘制 Painting

绘制阶段，遍历渲染树并调用渲染对象的paint方法将它们的内容显示在屏幕上，绘制使用UI基础组件，这在UI的章节有更多的介绍。

#### 全局和增量绘制 Global and Incremental

和布局一样，绘制也可以是全局的——绘制完整的树——或增量的。在增量的绘制过程中，一些渲染对象以不影响整棵树的方式改变，改变的渲染对象使其在屏幕上的矩形区域失效，这将导致操作系统将其看作dirty区域，并产生一个paint事件，操作系统很巧妙的处理这个过程，并将多个区域合并为一个。Chrome中，这个过程更复杂些，因为渲染对象在不同的进程中，而不是在主进程中。Chrome在一定程度上模拟操作系统的行为，表现为监听事件并派发消息给渲染根，在树中查找到相关的渲染对象，重绘这个对象（往往还包括它的子元素）。

#### 绘制顺序 The painting order

CSS2定义了[绘制顺序](http://www.w3.org/TR/CSS21/zindex.html)。这个就是元素压入堆栈的顺序，这个顺序影响着绘制，堆栈从后向前进行绘制。

一个块渲染对象的堆栈顺序是：
1. 背景颜色 background color
2. 背景图 background image
3. 边线 border
4. 子元素 children
5. 外框 outline

#### Firefox显示列表 Firefox display list

Firefox遍历渲染树，为绘制过的矩形创建一个显示列表，按照绘制顺序（首先是元素背景，然后边框等）记下对应的渲染对象。通过这个方法，重绘时只需查找一次树，而不需要多次查找，绘制所有背景、所有图片、所有边框等。

Firefox优化了这个过程，被隐藏的元素不会添加进来，比如元素完全在其它不透明元素下面。

#### WebKit局部重绘 WebKit rectangle storage

重绘前，WebKit将旧的矩形保存为位图，然后只绘制新旧矩形的差异区域。

### 动态变化 Dynamic changes

浏览器总是尝试以最小的动作响应一个变化，例如一个元素颜色的变化只会导致该元素的重绘，元素位置的变化将导致该元素和它的子元素以及兄弟元素布局和重绘，添加一个DOM节点也会导致这个元素的布局和重绘。类似增加html元素字体大小这样的重大变化将导致整个文档缓存失效、重布局和重绘。

### 渲染引擎的线程 The rendering engine's threads

渲染引擎是单线程的，除了网络操作外，几乎所有事情都在单线程中处理，在Firefox和Safari中是浏览器的主线程，Chrome中是tab进程的主线程。

网络操作可由几个并行线程执行，并行连接数量是受限的（通常是2－6个）。

#### 事件循环 Event loop

浏览器主线程是一个事件循环，它被设计为无限循环以保持执行过程的可用，等待事件（例如布局和绘制事件）并执行它们。下面是Firefox的主事件循环代码：

```c++
while (!mExiting)
    NS_ProcessNextEvent(thread);
```

### CSS2可视模型 CSS2 visual model

#### 画布 The Canvas

根据[CSS2规范](http://www.w3.org/TR/CSS21/intro.html#processing-model)，术语画布描述为*用来渲染显示格式化结构的空间*，即浏览器绘制内容的地方。画布每个维度的空间都无限大，浏览器基于viewport大小选择一块作为初始区域。

根据[www.w3.org/TR/CSS2/zindex.html](http://www.w3.org/TR/CSS2/zindex.html)的定义，画布如果包含在其他画布内则是透明的，否则使用浏览器自定义的颜色。

#### CSS盒模型

[CSS盒模型](http://www.w3.org/TR/CSS2/box.html)为文档树中的元素生成一个矩形盒装状区域box，按照视觉格式模型进行布局。每个box包括内容区域（如图片、文本等）及周围可选的padding、border和margin区域。

<center><img src="https://github.com/Richie-Leo/ydres/blob/master/img/10/110/1000/19-css2-box-model.jpg?raw=true" /><br />图19：CSS2盒模型</center>

每个节点生成0－n个这样的box。

每个元素都有一个display属性，用来指定生成的box类型，例如：
```
block：生成块状box
inline：生成一个行级box
none：不生成box
```

默认为行级元素，浏览器样式表可能设置其它默认值，例如div默认为块级元素，更多默认样式参考[www.w3.org/TR/CSS2/sample.html](http://www.w3.org/TR/CSS2/sample.html)。

#### 定位策略 Positioning scheme

有三种策略：
1. Normal 普通文档流：对象根据它在文档的中位置定位，这也意味着元素在渲染树和DOM树中位置一致，并根据它的box类型和大小进行布局
2. Float 流式：对象先按普通文档流一样布局，然后尽可能的向左或是向右移动
3. Absolute 绝对定位：对象在渲染树中的位置和DOM树中位置无关

通过position属性和float样式指定定位策略

- static和relative使用普通文档流策略
- absolute和fixed使用绝对定位

Static定位使用默认位置，无需额外指定，其他策略中，开发者通过top、bottom、left、right指定位置。

Box布局由以下因素决定：
- Box类型
- Box大小
- 定位策略
- 扩展信息，比如图片大小、屏幕尺寸等

#### Box类型 Box types

==Block box - 块级元素==：一个块状区域，在浏览器窗口上有自己的矩形。
<center><img src="https://github.com/Richie-Leo/ydres/blob/master/img/10/110/1000/20-block-box.png?raw=true" /></center>

==Inline box - 行级元素==：并没有自己的块状区域，而是包含在父级块状区域中。
<center><img src="https://github.com/Richie-Leo/ydres/blob/master/img/10/110/1000/21-inline-boxes.png?raw=true" /></center>

块级元素依次垂直排列，行级元素依次水平排列。
<center><img src="https://github.com/Richie-Leo/ydres/blob/master/img/10/110/1000/22-block-and-inline-formating.png?raw=true" /></center>

行级元素位于一行或多行中，行高不小于行内最高的元素，采用baseline对齐方式时，行高可能大于行内最高元素。Baseline对齐方式指元素的底部对齐另一个元素除底部外的其它位置。当容器宽度不够时，行级元素将被放到多行中，典型的就是段落元素中的分行。
<center><img src="https://github.com/Richie-Leo/ydres/blob/master/img/10/110/1000/23-lines-formating.png?raw=true" /></center>

#### 定位 Positioning

##### 相对定位 Relative

相对定位：先正常定位，然后计算差值，根据差值移动。
<center><img src="https://github.com/Richie-Leo/ydres/blob/master/img/10/110/1000/24-relative-positioning.png?raw=true" /><br />图24：相对定位</center>

##### 浮动定位 Floats

浮动定位的的box靠左或靠右对齐某条线，其余box围绕它周围布局。下面html：

```html
<p>
  <img style="float: right" src="images/image.gif" width="100" height="100">
  Lorem ipsum dolor sit amet, consectetuer...
</p>
```
将显示为：
<center><img src="https://github.com/Richie-Leo/ydres/blob/master/img/10/110/1000/25-float-positioning.png?raw=true" /><br />图25：浮动定位</center>

##### 绝对定位和固定定位 Absolute and fixed
这种情况下的布局忽略正常的文档流进行精确布局，元素不再属于文档流，位置取决于容器，Fixed时容器为viewport（视口，即浏览器窗口的可视区域）。
<center><img src="https://github.com/Richie-Leo/ydres/blob/master/img/10/110/1000/26-fixed-positioning.png?raw=true" /><br />图26：固定定位</center>

注意：fixed即使在文档流滚动时也不会移动。

*批注：absolute绝对定位，其位置相对于父元素固定，当文档存在滚动条并滚动时，绝对定位的元素随着父元素一起滚动；fixed固定定位其位置相对于viewport视口固定，当文档存在滚动条并滚动时，元素不会随着滚动条一起滚动*

#### 分层显示 Layered representation

CSS属性`z-index`指定分层显示，这是盒模型的第三个维度：对象在z轴上的位置。

Box会分到不同的栈中（也叫做栈上下文），每个栈中靠后的元素先绘制，栈顶靠前的元素绘制在其上面，出现重叠时靠前的元素将覆盖遮挡之前已经绘制的元素。这些栈按照z-index属性值排序，拥有z-index属性的box形成一个局部堆栈，viewport视口拥有外围栈。（*根据此处描述推测：每个z-index属性值对应一个栈，z-index值相同的元素位于同一个栈中。在同一个栈中，HTML中位置靠前的先入栈，靠近栈底，位置靠后的后入栈，靠近栈顶*）

例如：

```html
<style type="text/css">
      div {
        position: absolute;
        left: 2in;
        top: 2in;
      }
</style>

<p>
    <div
         style="z-index: 3;background-color:red; width: 1in; height: 1in; ">
    </div>
    <div
         style="z-index: 1;background-color:green;width: 2in; height: 2in;">
    </div>
 </p>
 ```
 
 结果是：
 <center><img src="https://github.com/Richie-Leo/ydres/blob/master/img/10/110/1000/27-fixed-positioning.png?raw=true" /></center>
 
 HTML中红色div出现在绿色div前面，按正常流程应当先绘制（从而被后面绘制的绿色div覆盖遮挡），但它的z-index值更高，在box中更靠前（，因此先绘制绿色div，再绘制红色div，从而达到上图红色div位于绿色div上层的效果）。
 
 ### 资源 Resources
1. 浏览器架构 Browser architecture
   1. Grosskurth, Alan. [A Reference Architecture for Web Browsers  (pdf)](http://grosskurth.ca/papers/browser-refarch.pdf)
   2. Gupta, Vineet. [How Browsers Work–Part 1–Architecture](http://www.vineetgupta.com/2010/11/how-browsers-work-part-1-architecture/)
2. 解析 Parsing
   1. Aho, Sethi, Ullman, Compilers: Principles, Techniques, and Tools (aka the "Dragon book"), Addison-Wesley, 1986
   2. Rick Jelliffe. [The Bold and the Beautiful: two new drafts for HTML 5.](http://broadcast.oreilly.com/2009/05/the-bold-and-the-beautiful-two.html)
3. Firefox
   1. L. David Baron, [Faster HTML and CSS: Layout Engine Internals for Web Developers.](http://dbaron.org/talks/2008-11-12-faster-html-and-css/slide-6.xhtml)
   2. L. David Baron, [Faster HTML and CSS: Layout Engine Internals for Web Developers (Google tech talk video)](https://www.youtube.com/watch?v=a2_6bGNZ7bA)
   3. L. David Baron, [Mozilla's Layout Engine](http://www.mozilla.org/newlayout/doc/layout-2006-07-12/slide-6.xhtml)
   4. L. David Baron, [Mozilla Style System Documentation](http://www.mozilla.org/newlayout/doc/style-system.html)
   5. Chris Waterson, [Notes on HTML Reflow](http://www.mozilla.org/newlayout/doc/reflow.html)
   6. Chris Waterson, [Gecko Overview](http://www.mozilla.org/newlayout/doc/gecko-overview.htm)
   7. Alexander Larsson, [The life of an HTML HTTP request](https://developer.mozilla.org/en/The_life_of_an_HTML_HTTP_request)
4. WebKit
   1. David Hyatt, [Implementing CSS(part 1)](http://weblogs.mozillazine.org/hyatt/archives/cat_safari.html)
   2. David Hyatt, [An Overview of WebCore](http://weblogs.mozillazine.org/hyatt/WebCore/chapter2.html)
   3. David Hyatt, [WebCore Rendering](http://webkit.org/blog/114/)
   4. David Hyatt, [The FOUC Problem](http://webkit.org/blog/66/the-fouc-problem/)
5. W3C规范 W3C Specifications
   1. [HTML 4.01 Specification](http://www.w3.org/TR/html4/)
   2. [W3C HTML5 Specification](http://dev.w3.org/html5/spec/Overview.html)
   3. [Cascading Style Sheets Level 2 Revision 1 (CSS 2.1) Specification](http://www.w3.org/TR/CSS2/)
6. 浏览器编译构建指导 Browsers build instructions
   1. Firefox. [https://developer.mozilla.org/en/Build_Documentation](https://developer.mozilla.org/en/Build_Documentation)
   2. WebKit. [http://webkit.org/building/build.html](http://webkit.org/building/build.html)