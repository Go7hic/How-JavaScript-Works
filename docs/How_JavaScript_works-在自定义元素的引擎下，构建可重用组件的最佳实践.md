# JavaScript 是如何工作的: 在自定义元素的引擎下，构建可重用组件的最佳实践

![](https://cdn-images-1.medium.com/max/1600/0*2oRILfJJtmW07mbK)
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
![](https://cdn-images-1.medium.com/max/1600/0*v56OyrPtg_cZzzaZ)

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

```
var myDiv = document.querySelector('.my-custom-element');

myDiv.addEventListener('click', _ => {
  myDiv.innerHTML = '<b> I have been clicked </b>';
});
```

```
<div class="my-custom-element">
  I have not been clicked yet.
</div>
```

通过自定义元素API，所有这些逻辑都可以封装到元素本身中。下面的示例与上面的示例完全相同。

```
class MyCustomElement extends HTMLElement {
  constructor() {
    super();

    var self = this;

    self.addEventListener('click', _ => {
      self.innerHTML = '<b> I have been clicked </b>';
    });
  }
}

customElements.define('my-custom-element', MyCustomElement);
```

```
<my-custom-element>
  I have not been clicked yet
</my-custom-element>
```

乍一看，自定义元素方法似乎需要更多的JavaScript行。但在实际应用程序中，很少会出现这样的场景，即您创建的单个元素不会被重用。现代web应用程序的一个典型特点是，大多数元素都是动态创建的。因此，当使用JavaScript动态添加元素或之前在HTML结构中定义元素时，需要处理不同的情况。你可以用自定义元素把所有这些都取出来。

总之，定制元素使您的代码更容易理解和维护，并将其分解为小的、可重用的和封装的模块。

#### 要求
现在，在继续创建第一个定制元素之前，您应该知道必须遵循一些特殊规则。

- 名称中必须包含破折号(-)。通过这种方式，HTML解析器可以分辨哪些元素是定制的，哪些元素是内置的。它还确保不会与内置元素发生名称冲突(无论是现在还是将来添加其他元素时)。例如，`<my-custom-element>`是一个有效的名称，而`<myCustomElement>`和`<my_custom_element>`则不是。
- 禁止多次注册相同的标签名。这将导致浏览器抛出DOMException。不能覆盖自定义元素。
- 自定义元素不能自关闭。HTML解析器只允许少量的内置元素自闭(例如<img>， <link>， <br>)。

#### 功能
那么，如何使用定制元素呢?答案是——很多事情。

最好的特性之一是元素的类定义实际上引用了DOM元素本身。这意味着您可以直接使用它来附加事件侦听器、访问其属性、访问子节点等等。

```
class MyCustomElement extends HTMLElement {
  // ...

  constructor() {
    super();

    this.addEventListener('mouseover', _ => {
      console.log('I have been hovered');
    });
  }

  // ...
}
```

当然，这使您能够使用新内容覆盖元素的子节点。但这通常是不建议的，因为它可能导致意想不到的行为。作为一个自定义元素的用户，如果您自己在元素内的标记被其他标记替代，您会感到惊讶。

有一些钩子可以用于在元素生命周期的特定时间执行代码。

```
constructor
```

一旦元素被创建或升级，就会调用构造函数(稍后我们将对此进行讨论)。它最常用于状态初始化、附加事件侦听器、创建影子DOM等。要记住的一件事是，应该始终在构造函数中调用super()。

```
connectedCallback
```

每次将元素添加到DOM时，都会调用connectedCallback方法。它可以用来(也推荐)延迟一些工作，直到元素真正出现在页面上(例如，获取资源)。

```
disconnectedCallback
```
与connectedCallback类似，一旦从DOM中取出一个元素，就会调用disconnectedCallback方法。通常用于释放资源。要记住的一件事是，如果用户关闭标签，则不会调用disconnectedCallback。因此，首先要注意初始化的内容。

```
attributeChangedCallback
```
一旦元素的属性被添加、删除、更新或替换，就会调用此方法。在解析器创建元素时也会调用它。但是，请注意，这只适用于observedAttributes属性中具有白属性的属性。

```
addoptedCallback
```
在调用document. adoptnode(…)方法以将其移动到另一个文档时，就会调用addoptedCallback方法。

注意，上面的所有回调都是同步的。例如，在将元素添加到DOM后立即调用连接回调，而在此期间不会发生其他事情。

#### Property reflection

内置的HTML元素提供了一个非常方便的功能:属性反射。这意味着一些属性的值作为属性直接返回到DOM。例如id属性。
```
myDiv.id =“new-id”;
```
还会更新DOM到
```
< div id = "new-id " >…< / div >
```
它也适用于相反的方向。这非常有用，因为它允许您声明地配置元素。

定制元素并没有将这种功能开箱即用，但是有一种方法可以自己实现它。为了在定制元素中实现相同的行为，我们可以为属性定义getter和setter。

```
class MyCustomElement extends HTMLElement {
  // ...

  get myProperty() {
    return this.hasAttribute('my-property');
  }

  set myProperty(newValue) {
    if (newValue) {
      this.setAttribute('my-property', newValue);
    } else {
      this.removeAttribute('my-property');
    }
  }

  // ...
}
```

#### Extending elements

自定义元素API不仅允许创建新的HTML元素，还允许扩展现有的元素。对于内置元素和其他自定义元素，它都能很好地工作。它只是通过扩展它的类定义来实现的。

```
class MyAwesomeButton extends MyButton {
  // ...
}

customElements.define('my-awesome-button', MyAwesomeButton);
```
在内置元素的情况下，我们还需要向 `customelement .define(…)`函数添加第三个参数。这将告诉浏览器究竟扩展了哪个元素，因为许多内置元素共享同一个DOM接口。如果不指定要扩展的元素，浏览器就不知道要扩展什么功能。

```
class MyButton extends HTMLButtonElement {
  // ...
}

customElements.define('my-button', MyButton, {extends: 'button'});
```

扩展的本机元素也称为自定义内置元素。

您可以使用的经验法则是始终扩展现有的元素。循序渐进。这允许您保留所有以前的特性(属性、属性、函数)。

注意，定制的内置元素现在只被Chrome 67+支持。它也将在其他浏览器中实现，但Safari选择完全不实现它。

#### Upgrading elements
如上所述，我们使用`customelement .define(…)`方法注册一个定制元素。但这并不意味着这是你要做的第一件事。注册自定义元素可以推迟一段时间。甚至在元素本身被添加到DOM之后。这个过程称为元素升级。为了让您知道元素是什么时候真正定义的，浏览器提供了customelement . whendefined(…)方法。将元素的标记名传递给它，它将返回一个承诺，该承诺在元素注册后解析。

```
customElements.whenDefined('my-custom-element').then(_ => {
  console.log('My custom element is defined');
});
```
例如，您可能想要延迟一些东西，直到定义了所有子元素。如果您有嵌套的定制元素，这将非常有用。有时，父元素可能依赖于其子元素的实现。在本例中，您需要确保在父元素之前定义了子元素。

#### Shadow DOM
正如我们已经说过的，定制元素和影子DOM应该放在一起。前者用于将JavaScript逻辑封装到元素中，而后者用于为不受外部世界影响的DOM块创建隔离环境。我建议你看看我们之前的一篇关于影子DOM的博客文章，以便更好地理解这个概念。

要为定制元素使用 Shadow DOM，只需调用this.attachShadow

```
class MyCustomElement extends HTMLElement {
  // ...

  constructor() {
    super();

    let shadowRoot = this.attachShadow({mode: 'open'});
    let elementContent = document.createElement('div');
    shadowRoot.appendChild(elementContent);
  }

  // ...
});
```

#### Templates

我们在之前的一篇文章中简要介绍了模板，它们本身就值得一篇。在这里，我们将给出一个简单的示例，说明如何将模板合并到自定义元素的创建中。使用<template>，您可以声明一个DOM片段标记，该标记被解析，但不会在页面上呈现。
  
  ```
  <template id="my-custom-element-template">
  <div class="my-custom-element">
    <input type="text" class="email" />
    <button class="submit"></button>
  </div>
</template>
```

```
let myCustomElementTemplate = document.querySelector('#my-custom-element-template');

class MyCustomElement extends HTMLElement {
  // ...

  constructor() {
    super();

    let shadowRoot = this.attachShadow({mode: 'open'});
    shadowRoot.appendChild(myCustomElementTemplate.content.cloneNode(true));
  }

  // ...
});
```

现在，我们已经将定制元素与影子DOM和模板相结合，我们得到了一个独立于其自身作用域的元素，它很好地分离了HTML结构和JavaScript逻辑。

#### Styling
我们讲了HTML和JavaScript那么CSS呢。显然，我们需要一种样式化元素的方法。我们可以在影子DOM中添加CSS样式表，但是您可能会问，如何从外部将元素样式化为元素的用户。答案很简单——你用和内置元素一样的方式设计它们。

```
my-custom-element {
  border-radius: 5px;
  width: 30%;
  height: 50%;
  // ...
}
```
注意，从外部定义的样式具有更高的优先级，它将覆盖从元素定义的样式。

您知道有时您可以实际看到页面呈现，您会看到一些未经样式的内容(FOUC)。您可以通过为未定义的组件定义样式来避免这种情况，并在定义组件时使用某种转换。为此，您可以使用:defined selector

```
my-button:not(:defined) {
  height: 20px;
  width: 50px;
  opacity: 0;
}
```
#### 未知元素vs未定义定制元素

HTML规范非常灵活，允许声明任何您想要的标记。如果标签没有被浏览器识别，它将被解析为HTMLUnknownElement。
```
var element = document.createElement('thisElementIsUnknown');

if (element instanceof HTMLUnknownElement) {
  console.log('The selected element is unknown');
}
```
但是，这不适用于定制元素。还记得我们讨论过定义定制元素有特定的命名规则吗?原因是，如果浏览器看到自定义元素的有效名称，它将把它解析为HTMLElement，浏览器认为它是未定义的自定义元素。

```
var element = document.createElement('this-element-is-undefined');

if (element instanceof HTMLElement) {
  console.log('The selected element is undefined but not unknown');
}
```
虽然HTMLElement和HTMLUnknownElement之间可能没有任何视觉上的区别，但还有一些事情需要记住。解析器对它们进行了不同的处理。具有有效自定义元素名称的元素应该具有自定义实现。在这个实现被定义之前，它被当作一个空的div元素。而未定义的元素不实现任何内置元素的任何方法或属性。

#### 浏览器支持
定制元素的第一个版本是在Chrome 36+中引入的。它是所谓的自定义组件API v0，现在已被弃用，并被认为是一种不好的实践，尽管仍然可用。尽管如此，如果你想了解更多关于v0的信息，你可以在这篇博文中阅读。自Chrome 54和Safari 10.1(尽管部分)以来，自定义元素API v1就可以使用了。微软的Edge正处于原型阶段，Mozilla从v50开始就有了，但默认情况下是不可用的，需要显式启用。目前只有webkit浏览器完全支持它。但是，如上所述，有一个polyfill允许您跨所有浏览器使用自定义元素。是的，甚至是IE 11。

#### 检查可用性
为了确保浏览器支持定制元素，您可以简单地检查customElements属性是否存在于窗口对象中。

```
const supportsCustomElements = 'customElements' in window;

if (supportsCustomElements) {
  // You can safely use the Custom elements API
}
```

或者如果你正在使用polyfill库:
```
function loadScript(src) {
  return new Promise(function(resolve, reject) {
    const script = document.createElement('script');

    script.src = src;
    script.onload = resolve;
    script.onerror = reject;

    document.head.appendChild(script);
  });
}

// Lazy load the polyfill if necessary.
if (supportsCustomElements) {
  // Browser supports custom elements natively. You're good to go.
} else {
  loadScript('path/to/custom-elements.min.js').then(_ => {
    // Custom elements polyfill loaded. You're good to go.
  });
}
```
综上所述，web组件标准的自定义元素部分提供了以下内容:

- 将JavaScript行为和CSS样式与HTML元素相关联
- 允许您扩展已经存在的HTML元素(内置和其他自定义元素)
- 不需要库或框架就可以开始。您只需要普通的JavaScript、HTML和CSS以及可选的填充库，就可以支持较老的浏览器。
- 它构建的目的是与其他web组件特性(影子DOM、模板、插槽等)无缝协作。
- 与浏览器的开发工具紧密集成。
- 利用现有的可访问性特性。

毕竟，自定义元素与我们到目前为止所使用的元素并没有多大区别。这只是在开发web应用程序时使事情更方便的另一种方式。因此，它开启了以更快的速度构建非常复杂的应用程序的可能性。但复杂性越高，引入一个难以追踪和重现的问题的可能性就越大。这就是为什么对它们进行解密需要更多的上下文，而像SessionStack这样的工具就能发挥作用。

SessionStack被集成到web应用程序中，以收集数据，如用户事件、网络数据、异常、调试消息、DOM更改等，并将这些数据发送到我们的服务器。

然后，对收集到的数据进行处理，以创建类似视频的体验，这样您就可以看到您的用户与您的产品交互。这与SessionStack提供的所有技术信息一起，使您能够重现以前无法跟踪的问题。

因此，为了确保SessionStack始终能够生成完美的像素会话回放，我们需要跟上新兴的技术、框架和web标准。


  
