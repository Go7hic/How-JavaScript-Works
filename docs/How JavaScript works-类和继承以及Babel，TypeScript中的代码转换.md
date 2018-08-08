# JavaScript 是如何工作的: 类和继承以及Babel，TypeScript中的代码转换

![](media/15332846510823/15335261847888.jpg)

#### 概述

在JavaScript中，没有原始类型，我们创建的所有东西都是对象。 例如，如果我们创建一个新字符串：
```js
const name = "SessionStack";
```

我们可以立即在新创建的对象上调用不同的方法：
```js
console.log(a.repeat(2)); //SessionStackSessionStack
console.log(a.toLowerCase()); // sessionstack
```

与其他语言不同，在JavaScript中，字符串或数字的声明会自动创建一个封装值的对象，并提供甚至可以在基元类型上执行的不同方法。

另一个有趣的事实是，诸如数组之类的复杂类型也是对象。 如果你看一下数组实例的typeof，你会发现它是一个对象。 列表中每个元素的索引只是对象中的属性。 因此，当您通过数组中的索引访问元素时，您实际上访问了数组对象的属性，并获得了后者的值。 说到数据的存储方式，这两个定义是相同的：
```js
let names = [“SessionStack”];

let names = {
  “0”: “SessionStack”,
  “length”: 1
}
```
因此，访问数组中的元素和对象的属性所花费的时间是相同的。 我发现了困难的方式。 前段时间我不得不对项目中的关键代码进行大量优化。 在尝试了所有简单选项后，我用数组替换了项目中使用的所有对象。 理论上，访问数组中的元素比访问哈希映射中的键更快。 我很惊讶地发现它对性能没有任何影响。 在JavaScript中，两个操作都实现为访问哈希映射中的键并花费相同的时间。

#### 使用原型模拟类
当我们想到对象时，首先想到的是类。 我们都习惯于根据类和它们之间的关系构建我们的应用程序。 虽然JavaScript中的对象无处不在，但该语言不使用经典的基于类的继承。 相反，它依赖于原型。
![](media/15332846510823/15335387142732.jpg)

我们将从一个为基类定义构造函数的简单示例开始：
```js
function Component(content) {
  this.content = content;
}

Component.prototype.render = function() {
    console.log(this.content);
}
```

我们将render函数附加到原型，因为我们希望Component类的每个实例都能够找到它。 在Component类的任何实例上调用此方法时，首先将在实例本身中执行搜索。 然后将在原型中执行搜索，这将是找到渲染方法的位置。
![](media/15332846510823/15335391738269.jpg)

所以现在让我们尝试扩展组件类。 我们将介绍一个新的子类。
```js
function InputField(value) {
    this.content = `<input type="text" value="${value}" />`;
}
```

如果我们希望InputField扩展组件类的功能并能够调用其render方法，我们需要更改其原型。 当在子类的实例上调用方法时，我们不希望在其空原型中进行搜索。 搜索应该在Component类中继续。
```js
InputField.prototype = Object.create(new Component());
```
这样，render方法可以在Component类的原型中找到。 为了获得继承，我们需要将InputField的原型连接到Component类的实例。 大多数库使用Object.setPrototypeOf方法来执行此操作。
![](media/15332846510823/15335392828805.jpg)

然而，这不是我们唯一需要做的事情。每次我们扩展课程时，我们都需要：

- 将子类的原型设置为父类的实例。
- 在子构造函数中调用父构造函数，以便可以执行父构造函数中的初始化逻辑。
- 介绍一种访问父方法的方法。当您覆盖方法并且想要在父方法中调用原始实现时，您需要这样做。

如您所见，如果要获得基于类的继承的所有功能，则需要每次都执行此复杂逻辑。每当您需要创建许多类时，将逻辑封装在可重用的函数中是有意义的。这就是开发人员最初解决基于类的继承问题的方法 - 通过使用不同的库模拟它。这些解决方案变得非常受欢迎，这使得语言中缺少某些东西显而易见。这就是为什么在ECMAScript 2015语言的第一个主要修订版中引入了用于创建支持基于类的继承的类的新语法。

#### 转换class
当提出ES6或ECMAScript 2015中的新功能时，JavaScript开发人员社区不能等待所有引擎和浏览器开始支持它们。 实现这一目标的一个好方法是通过转换。 它允许将ECMAScript 2015中编写的一段代码转换为任何浏览器都能理解的JavaScript。 这包括使用基于类的继承编写类的能力，并将它们转换为工作代码。
![](media/15332846510823/15335394406441.jpg)
最受欢迎的JavaScript转换器是Babel。 让我们看看如何通过在我们上面讨论的组件类的类定义上运行它来进行转换：
```js
class Component {
  constructor(content) {
    this.content = content;
  }

  render() {
  	console.log(this.content)
  }
}

const component = new Component('SessionStack');
component.render();
```
这就是Babel如何转换类定义：
```js
var Component = function () {
  function Component(content) {
    _classCallCheck(this, Component);

    this.content = content;
  }

  _createClass(Component, [{
    key: 'render',
    value: function render() {
      console.log(this.content);
    }
  }]);

  return Component;
}();
```
如您所见，代码转换为ECMAScript 5，可以在任何环境中执行。 此外，还添加了一些功能。 它们是Babel标准库的一部分。

_classCallCheck和_createClass作为函数包含在编译文件中。 第一个确保构造函数永远不会作为函数调用。 这是通过检查评估函数的上下文是否是Component对象的实例来实现的。 代码检查是否指向此类实例。 第二个函数_createClass处理对象属性的创建，这些属性作为带有键和值的对象列表传递。

为了探索继承如何工作，让我们分析从Component继承的InputField类。

```js
class InputField extends Component {
    constructor(value) {
        const content = `<input type="text" value="${value}" />`;
        super(content);
    }
}
```
这是我们使用Babel处理上述示例时得到的输出。
```js
var InputField = function (_Component) {
  _inherits(InputField, _Component);

  function InputField(value) {
    _classCallCheck(this, InputField);

    var content = '<input type="text" value="' + value + '" />';
    return _possibleConstructorReturn(this, (InputField.__proto__ || Object.getPrototypeOf(InputField)).call(this, content));
  }

  return InputField;
}(Component);
```
在此示例中，继承逻辑封装在_inherits函数调用中。 它通过将子类的原型设置为父类的实例来执行我们在上一节中描述的相同操作。

为了转换代码，Babel执行了几次转换。 首先，解析ECMAScript 2015代码并将其转换为称为抽象语法树的中间表示，我们已在前一篇文章中讨论过。 然后将此树转换为不同的抽象语法树，其中每个节点都转换为等效的ECMAScript 5。 最后，AST转换为代码。

#### Babel 里的 AST(抽象语法树)

AST包含节点，每个节点只有一个父节点。 在Babel中，节点有一个基本类型。 它包含有关节点的内容以及代码中可以找到的位置的信息。 有不同类型的节点，如文字，表示字符串，数字，空值等。还有用于流控制（if）和循环（for，while）的语句节点。 并且还有一种特殊类型的节点用于类。 它是基本Node类的子代。 它通过添加字段来扩展它，以存储对基类的引用和作为单独节点的类的主体。

让我们将以下代码片段转换为抽象语法树：
```js
class Component {
  constructor(content) {
    this.content = content;
  }

  render() {
    console.log(this.content)
  }
}
```

![](media/15332846510823/15335446664376.jpg)

在创建抽象语法树之后，将每个节点转换为其等效的ECMAScript 5节点，并返回到遵循ECMAScript 5标准的代码。 这是通过一个进程完成的，该进程找到离根节点最远的节点并将它们转换为代码。 然后，通过使用已为每个子项生成的片段将其父节点转换为代码，依此类推。 此过程称为深度优先遍历。

在上面的示例中，首先将生成两个MethodDefinition节点的代码，然后是类主体节点的代码，最后是ClassDeclaration节点的代码。

#### 使用TypeScript进行转换
```js
class Component {
    content: string;
    constructor(content: string) {
        this.content = content;
    }
    render() {
        console.log(this.content)
    }
}
```
![](media/15332846510823/15335449597070.jpg)

它还支持继承。
```js
class InputField extends Component {
    constructor(value: string) {
        const content = `<input type="text" value="${value}" />`;
        super(content);
    }
}
```
以下是转换结果
```js
var InputField = /** @class */ (function (_super) {
    __extends(InputField, _super);
    function InputField(value) {
        var _this = this;
        var content = "<input type=\"text\" value=\"" + value + "\" />";
        _this = _super.call(this, content) || this;
        return _this;
    }
    return InputField;
}(Component));
```
最终结果是ECMAScript 5代码，其中包含TypeScript库中的一些函数。 封装在__extends中的逻辑与我们在第一部分中讨论的逻辑相同。

随着Babel和TypeScript被广泛采用，标准类和基于类的继承成为构建JavaScript应用程序的标准方法。 这推动了浏览器中对类的本机支持的引入。

#### 原生支持

2014年，Chrome中引入了对课程的原生支持。这允许在不需要任何库或转换器的情况下执行类声明语法。

![](media/15332846510823/15335452419227.jpg)

本机实现类的过程就是我们所说的语法糖。这只是一种奇特的语法，可以编译为语言中已经支持的相同原语。您可以使用新的易于使用的类定义，但它仍将导致创建构造函数和分配原型。

![](media/15332846510823/15335453040418.jpg)

#### V8 Support


让我们看看ECMAScript 2015类的原生支持如何在V8中运行。正如我们在上一篇文章中所讨论的那样，首先必须将新语法解析为有效的JavaScript代码并添加到AST中。因此，作为类定义的结果，将一个类型为ClassLiteral的新节点添加到树中。

该节点存储了几个东西。首先，它将构造函数保存为单独的函数。它还包含类属性列表。它们可以是方法，吸气剂，制定者，公共领域或私人领域。此节点还存储对此类扩展的父类的引用，该类再次存储构造函数，属性列表和父类。


将这个新的ClassLiteral转换为代码后，它将再次转换为函数和原型。

原文地址：https://blog.sessionstack.com/how-javascript-works-the-internals-of-classes-and-inheritance-transpiling-in-babel-and-113612cdc220