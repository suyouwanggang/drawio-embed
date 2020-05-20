<h1 align="center">Welcome to drawio-embed</h1>
<p>
  <img alt="Version" src="https://img.shields.io/badge/version-1.0.0-blue.svg?cacheSeconds=2592000" />
</p>

> embed drawio into your website

## 简介

傻瓜式地在你的网页中嵌入流程图。在一键植入后，开发者只需关心：

1. 在需要的时候，调用命令唤起 drawio 流程图来编辑
2. 监听编辑完成后返回的图片数据

## 快速开始

开箱即用，只需 3 步

```js
import drawioEmbed from "drawio-embed";

// 初始化
const openDrawio = drawioEmbed();

// 监听返回的图片数据
window.addEventListener("drawioImageCreated", evt => {
  const { imageContent, imageType } = evt;
});

// 在需要时打开 drawio 开始编辑
openDrawio();
```

## 使用 UMD 方式引入

```html
<script
  defer
  src="https://raw.githubusercontent.com/imaoda/drawio-embed/master/umd/drawio-embed.min.js"
  onload="window.openDrawio = drawioEmbed()"
></script>
```

流程图初始化时会占用较多的网络资源，可选择滞后/按需加载

## DIY

黏贴下面代码到本地并用浏览器打开即可，或查看 [Demo](https://imaoda.github.io/drawio-embed/demo.html)

```html
<!DOCTYPE html>
<html>
  <body>
    <button onclick="openDrawio()">
      打开编辑 (实际使用时，需判断 drawio 是否初始化好)
    </button>
    <div>png图：</div>
    <img id="png" src="" />
    <div>svg图：<small>(不失真，但需考虑安全性兼容性)</small></div>
    <div id="svg"></div>
  </body>
  <script
    src="https://imaoda.github.io/drawio-embed/umd/drawio-embed.min.js"
    onload="openDrawio = drawioEmbed()"
  ></script>
  <script>
    const pngDom = document.querySelector("#png");
    const svgDom = document.querySelector("#svg");

    // 监听返回的图片
    window.addEventListener(
      "drawioImageCreated",
      ({ imageType, imageContent }) => {
        if (imageType === "png") pngDom.src = imageContent;
        if (imageType === "svg") svgDom.innerHTML = imageContent;
      }
    );
  </script>
</html>
```

## 部署自己的 drawio

默认调用 drawio 官网的流程图，初次访问较慢 _(不过初次加载后会建立 service worker 缓存)_。当然，如果追求速度和安全性，我们也可以自己部署一套，部署十分方便，只需 2 步，完成静态资源托管

1. git clone https://github.com/jgraph/drawio
2. 静态资源托管 `src/main/webapp` 路径

在初始化的时候，更改配置，指向自己部署的 drawio

```js
const openDrawio = drawioEmbed("https://xxx.com");
```

## Documentation

### 初始化

```js
import drawioEmbed from "drawio-embed";
const openDrawio = drawioEmbed();
```

执行 `drawioEmbed()` 将在你的页面中，插入一个 iframe，iframe 加载流程图，并挪动到可视区域之外

首次初始化之后，重复执行 `drawioEmbed()` 不会再执行，但依然会返回 `openDrawio` 的引用

### 打开 drawio

在页面中，通过初始化时的函数，来唤起 drawio 页面

```js
openDrawio();
```

此时 drawio 的 iframe 会全屏覆盖你的窗口。

如果需要在打开的时候，指定打开一个已有的 svg 图片，比如继续编辑上次的内容

```js
openDrawio("https://xxx.com/1.svg");
openDrawio("<svg>...</svg>");
```

注：这里的 svg 需使用 drawio 生成的 svg _(包含了附加的流程图信息)_，否则会提示无法打开

## 获取 drawio 导出的图片

在 drawio 编辑界面，点击右上角的「确定」按钮，可将编辑的流程图导出，并关闭 drawio 页面。代码中如何监听 drawio 发来的图片数据呢？可通过事件来捕获

1. 在页面初始化的时，为 `window` 增加特定的监听
2. 在触发事件时，提取其中的数据

```js
window.addEventListener("drawioImageCreated", ev => {
  const { imageType, imageContent } = ev;
  if (type === "png") {
    console.log("base64 格式的 png 图片信息", imageContent);
  } else {
    console.log("svg 标签文本", imageContent); // <svg>...</svg>
  }
});
```

一次用户保存，会在两个 eventLoop 里派发出两个 `drawioImageCreated` 事件，一个是导出 PNG，另一个是导出 SVG，开发者可根据实际需求，进行事件的监听

## 聊一聊 drawio 的初始化

借助本工具，会自动将 drawio 会以 iframe 的形式初始化，我们可以把它看成一个附属的应用程序，我们借助它来绘制图片以及创建图片数据。

iframe 形式的引入通常是耦合性较小的方式，并且我们也可以灵活方便的控制其加载时机。但同时还引入一个问题是，我们并不知道 iframe 是否加载完成，iframe 页面是否可以和我们的主页面进行通信。

因此在我们调用 `openDrawio` 时，有可能 iframe 还未加载完，为此，我们提供了一个错误回调

```js
openDrawio(er => {
  console.log(`drawio 还在初始化中，请稍后打开 ${er.msg}`);
});

openDrawio("https://xxx.com/1.svg", () => {
  console.log(`drawio 还在初始化中，请稍后打开`);
});
```

drawio 的初始化过程，分为以下几个阶段：

1. 父页面初始加载
2. 执行 `drawioIntegrator` 开始挂载 drawio iframe。此时 openDraw 方法可使用，但可能出现还在初始化的报错
3. drawio iframe 加载完毕，向 parent window 发出 `load` 事件。此时 openDraw 可以正常唤起编辑页面

> 向 window 添加 drawioImageCreated 事件可在任意阶段添加，不受初始化过程影响

## 聊一聊 drawio 与父页面的通信

## 个人 drawio 的部署与微调

我们可以直接 `git clone https://github.com/jgraph/drawio` 并部署 `src/main/webapp` 目录即可

如果部分内容需要微调，可加入 script，或者修改已有的 script 来实现调整，比如我们可以在 url 上新加入配置 `&retina=1`，表导出 2 倍图，而在源码中，我们需要将 `EditorUi.prototype.exportToCanvas` 的相应参数(第八个) 动态调整成 2 倍

我们在修改/新增原有 js 时，需留意，drawio 本身引入了 service-worker，可以对应做改动，其中 service-worker 通过 app.js 引用，不过其静态资源的 url 会有路径问题

#### 扩展新协议

我们可以修改 drawio，使其可以接受新的协议，通常如果你的新协议不涉及改动核心层，而是只是外围的 UI，那么这个很简单，比如我们扩展一个下载到本地的协议

自行扩展的协议名，比如 savelocal，在原有的协议处理过程时不识别，就直接被丢弃了，而被我们自己定义的流程接管

```js
window.addEventListener("message", function(e) {
  var data = e.data;
  var msg = {};
  try {
    msg = JSON.parse(data);
  } catch (error) {}
  // 扩展消息，保存到本地
  if (msg.action == "savelocal") {
    // do something
  }
});
```

## svg 信息携带

## svg 安全问题

## 最佳实践

## TypeScript

`drawio-embed` include [Typescript](https://www.typescriptlang.org/) definitions

```js
import drawioEmbed from "drawio-embed";
const openDrawio = drawioEmbed();
openDrawio.close();
```

## Author

👤 **wangyongfeng**

- Github: [@imaoda](https://github.com/imaoda)

## Show your support

Give a ⭐️ if this project helped you!

---

_This README was generated with ❤️ by [readme-md-generator](https://github.com/kefranabg/readme-md-generator)_
