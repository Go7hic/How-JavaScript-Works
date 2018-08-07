# JavaScript 是如何工作的: 在自定义元素的引擎下，构建可重用组件的最佳实践

![](media/15335456893446/15335457574971.jpg)
这是专门探索JavaScript及其构建组件的系列文章的第19篇。 在识别和描述核心元素的过程中，我们还分享了一些我们在构建SessionStack时使用的经验法则，这是一个JavaScript应用程序，需要强大且高性能，以帮助用户实时查看和重现其Web应用程序缺陷。

#### 概述
在我们之前的一篇文章中，我们讨论了Shadow DOM API和一些其他概念，这些概念都是更大图片的一部分 - Web组件。 Web组件标准背后的整体思想是通过创建小型，模块化和可重用的元素来扩展HTML的内置功能。这是一个相对较新的W3C标准，已经得到了所有主流浏览器的认可，可以在生产环境中看到......当然还有一个polyfill库的帮助（我们将在后面讨论）。

您可能已经知道，浏览器为我们提供了一些非常重要的工具来构建网站和Web应用程序。我们谈论的是HTML，CSS和JavaScript。您使用HTML来构建应用程序，使用CSS使其看起来很漂亮，使用JavaScript来实现操作。但是，在引入Web组件之前，没有简单的方法将JavaScript行为与HTML结构相关联。

在这篇文章中，我们将探讨Web组件的基础 - 自定义元素。简而言之，自定义元素API允许您创建（顾名思义）具有内置JavaScript逻辑和CSS样式的自定义HTML元素。许多人将自定义元素与shadow DOM混淆。但它们是两个完全不同的概念，它们实际上是互补的而不是可互换的。

一些框架（例如Angular，React）试图通过引入他们自己的概念来解决同样的问题。您可以将自定义元素与Angular的指令或React的组件进行比较。但是，自定义元素是浏览器的原生元素，只需要vanilla JavaScript，HTML和CSS。当然，这并不一定意味着它是典型JavaScript框架的替代品。现代框架为我们提供的不仅仅是能够模拟自定义元素的行为。所以这两者可以并肩工作。

#### API

在我们深入研究之前，让我们快速了解一下API的实际情况。 customElements全局对象为您提供了一些方法：

- define（tagName，constructor，options） - 定义一个新的自定义元素。采用三个参数：自定义元素的有效标记名称，自定义元素的类定义和选项对象。目前只支持一个选项：extends，它是一个字符串，指定要扩展的内置元素的名称。用于创建自定义内置元素。
- get（tagName） - 如果定义了元素，则返回自定义元素的构造函数，否则返回undefined。采用单个参数：自定义元素的有效标记名称。
- whenDefined（tagName） - 返回一个定义自定义元素后解析的promise。如果元素已经定义，则会立即解析。如果标记名称不是有效的自定义元素名称，则拒绝承诺。采用单个参数：自定义元素的有效标记名称

#### 如何创建自定义元素
创建自定义元素实际上是件小事。 您需要做两件事：为元素创建一个类定义，该类定义应该扩展HTMLElement类并在您选择的名称下注册该元素。


```js
class MyCustomElement extends HTMLElement {
  constructor() {
    super();
    // …
  }

  // …
}

customElements.define('my-custom-element', MyCustomElement);
```

或者，如果您愿意，可以使用匿名类，以防您不想混淆当前范围
```js
customElements.define('my-custom-element', class extends HTMLElement {
  constructor() {
    super();
    // …
  }

  // …
});
```

从示例中可以看出，使用`customElements.define（...）`方法注册自定义元素。

#### 自定义元素解决的问题是什么

那实际上是什么问题。 Div汤是其中的一部分。 什么是你可能会问的div汤 - 它是现代网络应用程序中一个非常常见的结构，你有多个嵌套的div元素（div内div的div等等）。
```html
<div class="top-container">
  <div class="middle-container">
    <div class="inside-container">
      <div class="inside-inside-container">
        <div class="are-we-really-doing-this">
          <div class="mariana-trench">
            …
          </div>
        </div>
      </div>
    </div>
  </div>
</div>
```

使用这种结构，因为它使浏览器按原样呈现页面。 但是，它使HTML不可读并且很难维护。

例如，我们可能有一个看起来像这样的组件
![](media/15335456893446/15335510265806.jpg)

传统上，HTML可能如下所示。
```html
<div class="primary-toolbar toolbar">
  <div class="toolbar">
    <div class="toolbar-button">
      <div class="toolbar-button-outer-box">
        <div class="toolbar-button-inner-box">
          <div class="icon">
            <div class="icon-undo">&nbsp;</div>
          </div>
        </div>
      </div>
    </div>
    <div class="toolbar-button">
      <div class="toolbar-button-outer-box">
        <div class="toolbar-button-inner-box">
          <div class="icon">
            <div class="icon-redo">&nbsp;</div>
          </div>
        </div>
      </div>
    </div>
    <div class="toolbar-button">
      <div class="toolbar-button-outer-box">
        <div class="toolbar-button-inner-box">
          <div class="icon">
            <div class="icon-print">&nbsp;</div>
          </div>
        </div>
      </div>
    </div>
    <div class="toolbar-toggle-button toolbar-button">
      <div class="toolbar-button-outer-box">
        <div class="toolbar-button-inner-box">
          <div class="icon">
            <div class="icon-paint-format">&nbsp;</div>
          </div>
        </div>
      </div>
    </div>
  </div>
</div>
```
但想象一下，如果我们能让它看起来像这样
```html
<primary-toolbar>
  <toolbar-group>
    <toolbar-button class="icon-undo"></toolbar-button>
    <toolbar-button class="icon-redo"></toolbar-button>
    <toolbar-button class="icon-print"></toolbar-button>
    <toolbar-toggle-button class="icon-paint-format"></toolbar-toggle-button>
  </toolbar-group>
</primary-toolbar>
```
如果你问我，第二个例子要好得多。它更易于维护，可读，并且对浏览器和开发人员都有意义。它更简单。

另一个问题是可重用性。我们作为开发人员的工作不仅需要编写工作代码，还需要编写可维护代码。使一些代码可维护的一件事是能够轻松地重用一段代码而不是一次又一次地编写代码。

我会给你一个简单的例子，但你会明白这个想法。假设我们有以下元素：
```js
<div class="my-custom-element">
  <input type="text" class="email" />
  <button class="submit"></button>
</div>
```
如果我们需要在其他地方使用它，我们需要重新编写相同的HTML。现在假设我们需要进行需要应用于每个元素的更改。我们需要在代码中找到每个位置并一次又一次地执行完全相同的更改。无赖...

如果我们能做到以下几点，那不是更好吗？
```js
<my-custom-element></my-custom-element>
```

但是现代Web应用程序不仅仅是静态HTML。你需要与它互动。这来自JavaScript。通常，您可能会创建一些元素，然后附加您需要的任何事件侦听器，以便它们对用户的输入作出反应。它们是否被点击，拖动，悬停等等。