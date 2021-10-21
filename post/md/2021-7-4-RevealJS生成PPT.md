---
layout: post
title: 2021-07-04-RevealJS 使用html或者markdown生成网页slide
date: 2021-07-04
categories: blog
tags: [应用]
description: RevealJS 使用html或者markdown生成网页slide
---

## 官网链接  
https://revealjs.com/installation/

## 安装  
方法一：直接[下载](https://github.com/hakimel/reveal.js/archive/master.zip)压缩包，然后编辑index.html文件即可。
方法二：用NodeJS修改源代码，做进一步扩充。  
这里为了方便采用方法一。  

## PPT构成
一个PPT的层级需要按照 .reveal > .slides > section 构成，这里section表示一张页面，并且可以被无限制地重复。  section元素里如果还嵌套另外的元素，那么将被展示为纵向可滑动的页面

```html
<html>
  <head>
    <link rel="stylesheet" href="dist/reveal.css">
    <link rel="stylesheet" href="dist/theme/white.css">
  </head>
  <body>
    <div class="reveal">
      <div class="slides">
        <section>Horizontal Slide 1</section>
        <section>
            <section>Vertical Slide 1</section>
            <section>Vertical Slide 2</section>
        </section>
      </div>
    </div>
    <script src="dist/reveal.js"></script>
    <script>
      Reveal.initialize();
    </script>
  </body>
</html>

```


## 用markdown写slide
### html  页面内嵌入markdown  
为了用Markdown创建slide，需要在section标签里添加"data-markdown"属性并且需要将内容用\<textarea data-template\>包裹起来，如下例子：  
```html
<section data-markdown>
  <textarea data-template>
    ## Slide 1
    A paragraph with some text and a [link](http://hakim.se).
    ---
    ## Slide 2
    ---
    ## Slide 3
  </textarea>
</section>
```

### 将markdown编写在独立的文件
该方法只支持通过上述方法二安装的方式，方法一这样做无法得到想要的结果。  
需要将所有页面写在同一个markdown中，并且需要用section包裹起来，section的一些参数如下：
* data-separator: 定义了水平slides的分隔符，默认是^\r?\n---\r?\n$，也就是“会车---回车”表示新的一页PPT。
* data-separator-vertical: 定义了垂直slides的分隔符，默认是disabled，所以如果要使用垂直分隔符需要手动指定分隔符。
* data-separator-notes: 是一个正则表达式，用来确定当前Slides开始。
* data-charset: 表示使用哪个字符集  

一个具体的例子：  
```html
<section data-markdown="example.md"
        data-separator="^\n\n\n"
        data-separator-vertical="^\n\n"
        data-separator-notes="^Note:"
        data-charset="iso-8859-15">
    <!--
        Note that Windows uses `\r\n` instead of `\n` as its linefeed character.
        For a regex that supports all operating systems, use `\r?\n` instead of `\n`.
    -->
</section>
```

### markdown 高亮代码
```C++ [2-3|5]
int a = 0;
int b = 0;
int c = 0;
int d = 0;
int e = 0;
```

### markdown 元素添加属性
增加Slide的Section元素的属性，因为有些section需要属性控制，比如背景图片等。  
方法：使用<!-- .slide: data-background="#ff0000" -->  
例子：  
```html
<section data-markdown>
  <script type="text/template">
  <!-- .slide: data-background="#ff0000" -->
    Markdown content
  </script>
</section>
```

### 元素级别的属性
```html
<section data-markdown>
  <script type="text/template">
    - Item 1 <!-- .element: class="fragment" data-fragment-index="2" -->
    - Item 2 <!-- .element: class="fragment" data-fragment-index="1" -->
  </script>
</section>
```  

## Slide背景  
通过对section元素添加data-background属性添加背景，四种类型的背景可以添加：color, image, video 和 iframe。

### 颜色背景  
具体伸缩变换，[参照](https://revealjs.com/backgrounds/)
```
<section data-background-color="aquamarine">
  <h2>🐟</h2>
</section>
<section data-background-color="rgb(70, 70, 255)">
  <h2>🐳</h2>
</section>
```  

### 图片背景
```html
<section data-background-image="http://example.com/image.png">
  <h2>Image</h2>
</section>
<section data-background-image="http://example.com/image.png" 
          data-background-size="100px" data-background-repeat="repeat">
  <h2>This background image will be sized to 100px and repeated</h2>
</section>
```


### 视频背景  
自动播放的视频背景，有如下属性：  
* data-background-video: 一个单一的视频源或者用逗号分割的一组视频资源  
* data-background-video-loop: 是否循环播放，默认false
* data-background-video-muted: 是否静音，默认false
* data-background-size: 是否填满整个屏幕  
* data-background-opacity: 透明度，默认为1，不透明

```html
<section data-background-video="https://static.slid.es/site/homepage/v1/homepage-video-editor.mp4" 
          data-background-video-loop data-background-video-muted>
  <h2>Video</h2>
</section>
```

### iframe背景  
嵌入一个网页作为背景，会占满100%的宽和高。这个frame是背景，所以默认情况下是不能交互的，为了能够让它交互，需要指定data-background-interactive为true。

```html
<section data-background-iframe="https://slides.com"
          data-background-interactive>
  <h2>Iframe</h2>
</section>
```  

Iframe 是lazy-loaded的，只有在可见的时候才加载。如果你想要预加载iframe，需要在section上增加data-preload属性。当然你也可以对所有iframe预加载，使用preloadIframes配置。  

### 背景转换  
默认情况下，我们使用cross fade来过渡slides的背景，也可以通过修改backgroundTransition来配置修改方法。

## Media  
### 自动播放  
```html
<video data-autoplay src="http://clips.vorwaerts-gmbh.de/big_buck_bunny.mp4"></video>
```
如果想要全局都autoplay，则需要进行设置：  
```javascript
Reveal.initialize({
	autoPlayMedia: true
})
```

注意：嵌入的HTML \<video\>/\<audio\>和YouTube/Vimeo iframes会自动暂停当离开这个slide。当然可以对该元素增加data-ignore属性取消这个默认。

### lazy load  
```html
<section>
  <iframe data-src="https://hakim.se" data-preload></iframe>
</section>
```


## 代码  
代码的data-trim属性会移除掉多余的空格
```html
<section>
  <pre><code data-trim data-noescape>
(def lazy-fib
  (concat
   [0 1]
   ((fn rfib [a b]
        (lazy-cons (+ a b) (rfib b (+ a b)))) 0 1)))
  </code></pre>
</section>
```

给代码增加行数：在\<code\>标签里增加data-line-numbers，如果要高亮，则指定行号即可。例子：  
```html
<pre><code data-line-numbers="3,8-10">
<table>
  <tr>
    <td>Apples</td>
    <td>$1</td>
    <td>7</td>
  </tr>
  <tr>
    <td>Oranges</td>
    <td>$2</td>
    <td>18</td>
  </tr>
</table>
</code></pre>
```  

一步一步地跳转高亮代码：  
```html
<pre><code data-line-numbers="3-5|8-10|13-15">
<table>
  <tr>
    <td>Apples</td>
    <td>$1</td>
    <td>7</td>
  </tr>
  <tr>
    <td>Oranges</td>
    <td>$2</td>
    <td>18</td>
  </tr>
  <tr>
    <td>Kiwi</td>
    <td>$3</td>
    <td>1</td>
  </tr>
</table>
</code></pre>
```  

如果代码含有<>两个字符会被解析成heml，那么可以使用($lt; $gt;)避开，或者在代码里增加\<script type="text/template"\>会自动处理：  
```html
<pre><code><script type="text/template">
sealed class Either<out A, out B> {
  data class Left<out A>(val a: A) : Either<A, Nothing>()
  data class Right<out B>(val b: B) : Either<Nothing, B>()
}
</script></code></pre>
```  

## 添加公式  
配置Latex，默认已经配置好，这里给出如何修改配置：  
```html
<script src="plugin/math/math.js"></script>
<script>
  Reveal.initialize({
    math: {
      mathjax: 'https://cdn.jsdelivr.net/gh/mathjax/mathjax@2.7.8/MathJax.js',
      config: 'TeX-AMS_HTML-full',
      // pass other options into `MathJax.Hub.Config()`
      TeX: { Macros: { RR: "{\\bf R}" } }
    },
    plugins: [ RevealMath ]
  });
</script>
```  
使用公式的例子：  
```html
<section>
  <h2>The Lorenz Equations</h2>
  \[\begin{aligned}
  \dot{x} &amp; = \sigma(y-x) \\
  \dot{y} &amp; = \rho x - y - xz \\
  \dot{z} &amp; = -\beta z + xy
  \end{aligned} \]
</section>  
```
在markdown中使用公式，必须先用反引号将公式括起来，比如：  
```markdown
`$$ J(\theta_0,\theta_1) = \sum_{i=0} $$`
```

## Fragment  
Fragment被用来在slides上高亮或者逐渐展示子元素，默认的fragment样式是开始不可见然后并淡入。这个样式可以通过增加不同的类来替换。其他样式见[网址](https://revealjs.com/fragments/)
```html
<p class="fragment">Fade in</p>
<p class="fragment fade-out">Fade out</p>
<p class="fragment highlight-red">Highlight red</p>
<p class="fragment fade-in-then-out">Fade in, then out</p>
<p class="fragment fade-up">Slide up while fading in</p>
```

通过对同一个元素进行Fragment嵌套，可以实现多种动画，步骤是从外嵌套到内，动画也是一样。
样例： 
```html
<span class="fragment fade-in">
  <span class="fragment highlight-red">
    <span class="fragment fade-out">
      Fade in > Turn red > Fade out
    </span>
  </span>
</span>
```

### Fragment Order
当指定data-frame-index后，可以指定html渲染的顺序（但是相对位置不会改变）

## 内部超链接跳转  
可以通过创建超链接在sildes内进行跳转，首先需要对每个section赋值一个id，然后通过\<a href="#/id"\>进行跳转，一个例子：  

```html
<section>
	<a href="#/grand-finale">Go to the last slide</a>
</section>
<section>
	<h2>Slide 2</h2>
</section>
<section id="grand-finale">
	<h2>The end</h2>
	<a href="#/0">Back to the first</a>
</section>
```

当然超链接跳转也可以根据section的编号，例如：
```html  
<a href="#/2">Go to 2nd slide</a>
<a href="#/3/2">Go to the 2nd vertical slide inside of the 3rd slide</a>
```  

导航链接：  
导航链接可以嵌入在任何.reveal容器内，表示下一张或者上一张slide
```html
<button class="navigate-left">Left</button>
<button class="navigate-right">Right</button>
<button class="navigate-up">Up</button>
<button class="navigate-down">Down</button>

<!-- Previous vertical OR horizontal slide -->
<button class="navigate-prev">Prev</button>

 <!-- Next vertical OR horizontal slide -->
<button class="navigate-next">Next</button>
```  


## 布局  
revealJS 提供了一些帮助class来控制布局。  

### Stack
r-stack帮助你将多个元素堆叠地放置在中心，这个搭配着fragment逐渐递增地显示元素。

一个例子：
```html
<div class="r-stack">
  <img class="fragment" src="https://placekitten.com/450/300" width="450" height="300">
  <img class="fragment" src="https://placekitten.com/300/450" width="300" height="450">
  <img class="fragment" src="https://placekitten.com/400/400" width="400" height="400">
</div>
```

如果想要显示该图片同时让后一张图片消失，则设置一下fragment属性即可，比如：  
```html
<div class="r-stack">
  <img class="fragment fade-out" data-fragment-index="0" src="https://placekitten.com/450/300" width="450" height="300">
  <img class="fragment current-visible" data-fragment-index="0" src="https://placekitten.com/300/450" width="300" height="450">
  <img class="fragment" src="https://placekitten.com/400/400" width="400" height="400">
</div>
```

### Fit Text  
r-fit-text 类能够使文字尽可能的大，这样不需要手动调整字体size。

```html
<h2 class="r-fit-text">BIG</h2>
```

```html
<h2 class="r-fit-text">FIT TEXT</h2>
<h2 class="r-fit-text">CAN BE USED FOR MULTIPLE HEADLINES</h2>
```  

### Stretch  
r-stretch 布局助手可让您调整元素（如图像或视频）的大小，以覆盖幻灯片中剩余的垂直空间。 例如，在下面的示例中，我们的幻灯片包含一个标题、一个图像和一个署名。 由于图像具有 .r-stretch 类，因此其高度设置为幻灯片高度减去标题和署名行的组合高度。  

```html
<h2>Stretch Example</h2>
<img class="r-stretch" src="/images/slides-symbol-512x512.png">
<p>Image byline</p>  
```


