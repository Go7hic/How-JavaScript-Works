# JavaScript 是如何工作的: Shadow DOM的内部结构+如何构建自包含的组件

![](media/15324122760389/15324123119848.jpg)
这是专门探索JavaScript及其构建组件的系列文章的第17篇。 在识别和描述核心元素的过程中，我们还分享了一些我们在构建SessionStack时使用的经验法则，这是一个JavaScript应用程序，需要强大且高性能，以帮助用户实时查看和重现其Web应用程序缺陷。

#### 概述
Web Components是一套不同的技术，允许您创建可重用的自定义元素。它们的功能远离其他代码，您可以在Web应用程序中使用它们。

有4个Web组件标准：
- 暗影DOM
- HTML模板
- 自定义元素
- HTML导入
在本文中，我们将重点关注Shadow DOM。

Shadow DOM被设计为用于构建基于组件的应用程序的工具。 它为您可能遇到的Web开发中的常见问题提供了解决方案：
- Isolated DOM：组件的DOM是自包含的（例如，`document.querySelector（）`不会返回组件的shadow DOM中的节点）。 这也简化了Web应用程序中的CSS选择器，因为DOM组件是隔离的，它使您能够使用更通用的id /类名称而无需担心命名冲突。

- Scoped CSS：在shadow DOM中定义的CSS是作用域的。 样式规则不会泄漏，页面样式不会干扰它。

- Composition：为组件设计基于声明，基于标记的API。

#### Shadow DOM
本文假设您已经熟悉DOM及其API的概念。 如果没有，你可以在这里阅读一篇关于它的详细文章 - https://developer.mozilla.org/en-US/docs/Web/API/Document_Object_Model/Introduction。

除了两个不同之外，Shadow DOM只是一个普通的DOM：
- 与您创建和使用DOM的方式相比，如何创建/使用它与页面的其余部分相关
- 它与页面其余部分的关系如何

通常，您创建DOM节点并将它们作为子节点附加到另一个元素。 在shadow DOM的情况下，您创建一个附加到元素的范围DOM树，但它与实际的子元素分开。 这个作用域子树称为阴影树。 它附加的元素是它的影子主机。 您添加到阴影树的任何内容都将成为托管元素的本地元素，包括`<style>`。 这就是`shadow DOM`实现CSS样式范围的方式。

#### Creating Shadow DOM

shadow root 是附加到“host”元素的文档片段。 附加阴影根的那一刻是元素获得阴影DOM的时刻。 要为元素创建shadow DOM，请调用element.attachShadow（）：


```js
var header = document.createElement('header');
var shadowRoot = header.attachShadow({mode: 'open'});
var paragraphElement = document.createElement('p');

paragraphElement.innerText = 'Shadow DOM';
shadowRoot.appendChild(paragraphElement);
```

#### Composition in Shadow DOM (Shadow DOM中的组合)
组合是Shadow DOM最重要的特征之一。

编写HTML时，组合是构建Web应用程序的方式。 您可以组合并嵌套不同的构建块（元素），例如<div>，<header>，<form>等，以构建Web应用程序所需的UI。 其中一些标签甚至可以互相协作。

组合定义了为什么元素（如<select>，<form>，<video>等）是灵活的，并接受特定的HTML元素作为子元素，以便对它们做一些特殊的事情。

例如，<select>知道如何将<option>元素呈现到具有预定义项的下拉窗口小部件中。

Shadow DOM引入了以下可用于实现组合的功能。

#### Light DOM

这是组件用户写入的标记。 这个DOM存在于组件的shadow DOM之外。 这是元素的实际孩子。 想象一下，您已经创建了一个名为<better-button>的自定义组件，它扩展了原生HTML按钮，您希望在其中添加图像和一些文本。 这是它的样子：
```js
<extended-button>
  <!-- the image and span are extended-button's light DOM -->
  <img src="boot.png" slot="image">
  <span>Launch</span>
</extended-button>
```

“扩展按钮”是您定义的自定义组件，而其中的HTML称为Light DOM，由组件的用户添加。

这里的Shadow DOM是您创建的组件（“扩展按钮”）。 Shadow DOM是组件的本地，它定义了内部结构，作用域CSS，并封装了您的实现细节。

#### Flattened DOM tree (扁平的DOM树)
浏览器将轻量级DOM（由用户创建的DOM）分配到您的影子DOM并定义了自定义组件的结果呈现最终产品。 平顶树是您最终在DevTools中看到的以及页面上呈现的内容。

```js
<extended-button>
  #shadow-root
  <style>…</style>
  <slot name="image">
    <img src="boot.png" slot="image">
  </slot>
  <span id="container">
    <slot>
      <span>Launch</span>
    </slot>
  </span>
</extended-button>
```

#### Templates
当您必须在网页上重复使用相同的标记结构时，最好使用某种模板而不是一遍又一遍地重复相同的结构。 以前这是可能的，但HTML `<template>`元素（在现代浏览器中得到很好的支持）使它变得更容易。 此元素及其内容不会在DOM中呈现，但仍可使用JavaScript引用它。

让我们看一个简单的例子：
```js
<template id="my-paragraph">
  <p> Paragraph content. </p>
</template>
```
在您使用JavaScript引用它之前，它不会出现在您的页面中，然后使用以下内容将其附加到DOM：

```js
var template = document.getElementById('my-paragraph');
var templateContent = template.content;
document.body.appendChild(templateContent);
```

到目前为止，还有其他技术可以实现类似的行为，但正如前面提到的那样，将本机覆盖在内是非常好的。 它也有相当不错的浏览器支持：
![](media/15324122760389/15324156373324.jpg)

模板本身很有用，但它们使用自定义元素可以更好地工作。 我们将在本系列的另一篇文章中定义自定义元素，暂时您应该知道浏览器中的`customElement` API允许您使用自定义渲染定义自己的标记。
让我们定义一个Web组件，它使用我们的模板作为其shadow DOM的内容。 我们称之为<my-paragraph>：

```js
customElements.define('my-paragraph',
 class extends HTMLElement {
   constructor() {
     super();

     let template = document.getElementById('my-paragraph');
     let templateContent = template.content;
     const shadowRoot = this.attachShadow({mode: 'open'}).appendChild(templateContent.cloneNode(true));
  }
});
```
这里要注意的关键点是我们将模板内容的一个克隆附加到阴影根，它是使用`Node.cloneNode（）`方法创建的。

因为我们将其内容附加到shadow DOM，所以我们可以在模板中的一个`<style>`元素中包含一些样式信息，然后将其封装在自定义元素中。 如果我们只是将它附加到标准DOM，这将无法工作。

例如，我们可以将模板更改为以下内容：

```js
<template id="my-paragraph">
  <style>
    p {
      color: white;
      background-color: #666;
      padding: 5px;
    }
  </style>
  <p>Paragraph content. </p>
</template>
```
现在，我们使用模板定义的自定义组件可以像这样使用：

```js
<my-paragraph></my-paragraph>
```

#### Slots

模板有一些缺点，主要是静态内容，它不允许我们渲染我们的变量/数据，以使其按照您习惯的标准HTML模板的方式工作。

这是<slot>进入图片的地方。
This is where the <slot> comes into the picture.

您可以将插槽视为占位符，允许您将自己的HTML放在模板中。 这允许您创建通用HTML模板，然后通过添加插槽使其可自定义。

让我们看看上面的模板与插槽的外观如何：

```js
<template id="my-paragraph">
  <p> 
    <slot name="my-text">Default text</slot> 
  </p>
</template>
```

如果在元素包含在标记中时未定义插槽的内容，或者浏览器不支持插槽，则<my-paragraph>将仅包含后备内容“默认文本”。

要定义插槽的内容，我们应该在<my-paragraph>元素中包含一个HTML结构，其中slot属性的值等于我们希望它填充的插槽的名称。

和以前一样，这可以是你喜欢的任何东西：

```js
<my-paragraph>
 <span slot="my-text">Let's have some different text!</span>
</my-paragraph>
```
可插入插槽的元素称为Slotable; 当一个元素插入一个插槽时，它被称为插槽。

请注意，在上面的示例中，我们插入了一个<span>元素，它是一个开槽元素。 它有一个属性`slot`，它等于“my-text”，它与模板槽定义中`name`属性的值相同。

在浏览器中呈现后，上面的代码将创建以下Flatatened DOM树：

```js
<my-paragraph>
  #shadow-root
  <p>
    <slot name="my-text">
      <span slot="my-text">Let's have some different text!</span>
    </slot>
  </p>
</my-paragraph>
```
注意＃shadow-root元素 - 它只是Shadow DOM存在的指示符。
#### Styling

使用shadow DOM的组件可以由主页面设置样式，可以定义自己的样式，或者以CSS自定义属性的形式提供钩子，以便用户覆盖默认值。
##### Component-defined styles
Scoped CSS是Shadow DOM的最大特色之一：
- 外部页面中的CSS选择器不适用于组件内部。
- 组件内定义的样式不会影响页面的其余部分。 它们的范围是主机元素。

Shadow DOM中使用的CSS选择器本地应用于组件。 实际上，这意味着我们可以再次使用常见的id / class名称，而不必担心页面上其他地方的冲突。 简单的CSS选择器意味着更好的性能。

让我们看一下定义一些样式的＃shadow-root：

```html
#shadow-root
<style>
  #container {
    background: white;
  }
  #container-items {
    display: inline-flex;
  }
</style>

<div id="container"></div>
<div id="container-items"></div>
```
上面示例中的所有样式都是＃shadow-root的本地样式。

您还可以使用<link>元素在＃shadow-root中包含样式表，这些样式表也是本地的。

##### The :host pseudo-class
`：host`允许您选择并设置托管阴影树的元素的样式：

```css
<style>
  :host {
    display: block; /* by default, custom elements are display: inline */
  }
</style>
```
在涉及到以下内容时，您应该注意以下事项：host - 父页面中的规则具有比以下内容更高的优先级：元素中定义的主机规则。 这允许用户从外部覆盖您的顶级样式。 此外，：host仅在阴影根的上下文中工作，因此您不能在Shadow DOM之外使用它。

函数形式：host（<selector>）允许您在主机与<selector>匹配时定位主机。 这是组件封装响应用户交互或状态的行为以及基于主机设置内部节点样式的好方法： 

```css
<style>
  :host {
    opacity: 0.4;
  }
  
  :host(:hover) {
    opacity: 1;
  }
  
  :host([disabled]) { /* style when host has disabled attribute. */
    background: grey;
    pointer-events: none;
    opacity: 0.4;
  }
  
  :host(.pink) > #tabs {
    color: pink; /* color internal #tabs node when host has class="pink". */
  }
</style>
```
##### 主题和元素：host-context（<selector>）伪类
`：host-context（<selector>`）伪类与主机元素匹配（如果后者或其任何祖先与<selector>匹配）。

这种情况的常见用途是主题。 例如，许多人通过将类应用于<html>或<body>来进行主题化：

```html
<body class="lightheme">
  <custom-container>
  …
  </custom-container>
</body>
```
`:host-context(.lightheme)` would style <fancy-tabs> when it’s a descendant of `.lightheme`:


```css
:host-context(.lightheme) {
  color: black;
  background: white;
}
```

：host-context（）可用于主题，但更好的方法是使[用CSS自定义属性创建样式钩子](https://developers.google.com/web/fundamentals/web-components/shadowdom#stylehooks)。

##### 从外部设置组件的主机元素的样式
您可以通过使用它们的标记名称作为选择器从外部设置组件的宿主元素，如下所示：

```css
custom-container {
  color: red;
}
```
外部样式的优先级高于Shadow DOM中定义的样式。

```css
custom-container {
  width: 500px;
}
```
它将覆盖组件的规则：

```css
:host {
  width: 300px;
}
```

样式化组件本身只会让你到目前为止。 但是如果要为组件的内部构造样式会发生什么？ 为此，我们需要CSS自定义属性。

##### 使用CSS自定义属性创建样式钩子

如果组件的作者使用CSS自定义属性提供样式挂钩，则用户可以调整内部样式。

这个想法与<slot>类似，但适用于样式。

我们来看看下面的例子：
```css
<!-- main page -->
<style>
  custom-container {
    margin-bottom: 60px;
     - custom-container-bg: black;
  }
</style>

<custom-container background>…</custom-container>
```
在其Shadow DOM中：

```css
:host([background]) {
  background: var( - custom-container-bg, #CECECE);
  border-radius: 10px;
  padding: 10px;
}
```

在这种情况下，组件将使用黑色作为背景值，因为用户提供了它。 否则，它将默认为#CECECE。

作为组件作者，您负责让开发人员了解他们可以使用的CSS自定义属性。 将其视为组件公共接口的一部分。

#### Slots JavaScript API

Shadow DOM API提供了使用插槽的实用程序。

##### slotchange事件
当插槽的分布式节点发生更改时，slotchange事件将触发。 例如，如果用户从轻DOM添加/删除子项。
```js
var slot = this.shadowRoot.querySelector('#some_slot');
slot.addEventListener('slotchange', function(e) {
  console.log('Light DOM change');
});
```
要监视light DOM的其他类型的更改，可以在元素的构造函数中使用MutationObserver。 我们之前已经讨论过[MutationObserver的内部结构以及如何使用它](https://blog.sessionstack.com/how-javascript-works-tracking-changes-in-the-dom-using-mutationobserver-86adc7446401)。

#### assignedNodes() 方法
知道哪些元素与插槽相关联可能很有用。 调用slot.assignedNodes（），可以为您提供插槽所呈现的元素。 {flatten：true}选项还将返回插槽的后备内容（如果没有分发节点）。

我们来看下面的例子：
```js
<slot name=’slot1’><p>Default content</p></slot>
```

我们假设这是一个名为`<my-container>`的组件.

让我们看一下这个组件的不同用法以及调用assignedNodes（）的结果：

在第一种情况下，我们将自己的内容添加到插槽中：
```js
<my-container>
  <span slot="slot1"> container text </span>
</my-container>
```

调用assignedNodes（）将导致[<span slot =“slot1”>容器文本</ span>]。 请注意，结果是一个节点数组。

在第二种情况下，我们将内容留空：
```js
<my-container> </my-container>
```
调用assignedNodes（）的结果将返回一个空数组[]。

但是，如果您为同一元素传递{flatten：true}参数，则会得到默认内容：[<p>默认内容</ p>]。

此外，要访问插槽内的元素，可以调用assignedNodes（）以查看元素分配给哪个组件插槽。

#### Event Model
有趣的是要注意当Shadow DOM中发生的事件冒泡时会发生什么。

调整事件的目标以维持Shadow DOM提供的封装。 重新定位事件时，它看起来像是来自组件本身，而不是作为组件一部分的Shadow DOM中的内部元素。

以下是从Shadow DOM传播的事件列表（有些事件没有）：

- Focus Events: blur, focus, focusin, focusout
- Mouse Events: click, dblclick, mousedown, mouseenter, mousemove, etc.
- Wheel Events: wheel
- Input Events: beforeinput, input
- Keyboard Events: keydown, keyup
- Composition Events: compositionstart, compositionupdate, compositionend
- DragEvent: dragstart, drag, dragend, drop, etc.

#### Custom Events
默认情况下，自定义事件不会传播到Shadow DOM之外。 如果要调度自定义事件并希望它传播，则需要添加bubbles：true和composition：true作为选项。

让我们看看如何调度这样的事件：
```js
var container = this.shadowRoot.querySelector('#container');
container.dispatchEvent(new Event('containerchanged', {bubbles: true, composed: true}));
```

浏览器支持

要检测Shadow DOM是否是受支持的功能，请检查是否存在attachShadow：
```js
const supportsShadowDOMV1 = !!HTMLElement.prototype.attachShadow;
```
![](media/15324122760389/15330313379242.jpg)

通常，Shadow DOM的行为方式与DOM完全不同。 我们从SessionStack库的经验中得到了第一手的例子。 我们的库集成到Web应用程序中以收集用户事件，网络数据，异常，调试消息，DOM更改等数据，并将此数据发送到我们的服务器。

之后，我们处理收集的数据，以便您使用SessionStack重播产品中发生的问题。
使用Shadow DOM产生的困难如下：我们必须监视每个DOM更改，以便以后能够正确地重放它。 我们通过使用MutationObserver来做到这一点。 但是，Shadow DOM不会在全局范围内触发MutationObserver事件，这意味着我们需要以不同方式处理这些组件。 我们发现现在有越来越多的网络应用利用Shadow DOM，所以看来这项技术前景光明.

#### 参考

- https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_shadow_DOM
- https://developers.google.com/web/fundamentals/web-components/shadowdom
- https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_templates_and_slots
- https://www.html5rocks.com/en/tutorials/webcomponents/shadowdom-201/#toc-style-host