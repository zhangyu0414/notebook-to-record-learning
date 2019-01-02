# 思考如何不在通过图片服务去生成图片

## 前言

这段时间的工作中，因为业务中频繁的使用图片模板的json数据去渲染图片，导致渲染时长比较久，在部分业务操作时候需要调用公司的图片服务，😔😔，╮(╯▽╰)╭哎，后来想尝试用html2canvas来转为canvas再去导出图片，但是发现html2canvas转化我们特效字体的时候出现一些不理想效果，但是这个框架近期有做更新，并且对于每个css样式都有做测试用例，他们自己实现了一套css解析，然后就再GitHub上找到 domtoimage这个js插件是通过浏览器内置的api做css解析，效果会比html2canvas好，但是拓展性不是特别高，所以想做一个这两个插件的深度分析。


## domtoimage 和html2canvas 对比

#### HTML to image有两种方案比较流行，一个是html2canvas，一个是dom-to-image。它们的设计初衷其实都是将已有DOM结构转成图片类型。对比来看

* 流行度上，html2canvas流行度更高，资料更好找，但更新缓慢
* 格式支持上，dom-to-image可以将图转成SVG等更多格式，html2canvas只能输出canvas，需要用户自行处理
* 清晰度上，dom-to-image可以导出SVG，html2canvas则需要hack的方式（设置更大的canvas绘制再等比缩放）
* 实现原理上，都是通过遍历DOM树，读取格式化数据，dom-to-image通过浏览器解析CSS语法，因此支持度更高；html2canvas则自己实现了CSS解析
* 速度上，相同模板dom-to-image是稍微快一点包括导出img的时间大约为1000ms上下，而html2canvas生成canvas就会在1100ms左右。

#### 渲染图片的HTML模板在通常情况下，不应该展示给用户。即生成过程短暂停留的DOM需要用户不可见。不可见的方式大致有下面几种：

* display: none，这种情况，两个方案度都输出空白图片
* visibility: hidden，在输出图片时，DOM结构会短暂闪现，两种方案都输出空白图片
* 将DOM移出视口，html2canvas可以正确输出图片，dom-to-image不行


## 已知缺陷

* 对部分CSS属性支持度有限，如box-shadow，-webkit-line-clamp，background-position等。
* 使用时需要额外的卸载操作。
* domtoimage无法兼容图形遮罩mask-box-image，和svg 的阴影drop-shadow。

## domtoimage 和 html2canvas 工作原理

### domtoimage
使用`svg`的一个特性，允许在`<foreignobject>`标签中包含任意的`html`内容。（主要是 XMLSerializer | MDN这个`api`将`dom`转为`svg`） 所以，为了渲染那个`dom`节点，你需要采取以下步骤：
1.  递归 `clone` 原始的 `dom` 节点
2.  获取 节点以及子节点 上的 `computed style`，并将这些样式添加进新建的style标签中（不要忘记了clone 伪元素的样式）
3.  嵌入网页字体

*   找到所有的`@font-face`
*   解析URL资源，并下载对应的资源
*   base64编码和内联资源 作为 `data:` URLS引用
*   把上面处理完的`css rules`全部都放进`<style>`中，并把标签加入到clone的节点中去
4.  内嵌图片(都转成dataUrl)
*   内联图片src 的url 进 `<img>`元素
*   背景图片 使用 background css 属性，类似fonts的使用方式
5.  序列化 clone 的 dom 节点 为 `svg`
6.  将xml包装到`<foreignobject>`标签中，放入`svg`中，然后将其作为`data: url`
7.  将png内容或原始数据作为`uint8array`获取，使用svg作为源创建一个`img`标签，并将其渲染到新创建的`canvas`上，然后把`canvas`转为`base64`
8.  完成

### html2canvas

1.  遍历页面中的所有元素，提取DOM节点，填充到一个rederList，并附加是否为顶层元素/包含内容的容器等信息
2.  通过提取的css属性和元素的层级信息将rederList排序，计算出一个canvas的renderQueue
3.  遍历renderQueue，将css样式转为setFillStyle可识别的参数，依据nodeType调用相对应canvas方法，将节点对应到 canvas 上。

补充：html2canvas 上/tests文件下面有各个样式的兼容情况，并且有各种样式的render time，开发同学可以直接上面进行测试，所以html2canvas更适合拿来做尝试

## 分析 domtoimage 内部 toPng 方法的原理

> 尽量挑最核心的讲，希望不会显得很繁琐，了解核心思想就好

![](https://user-gold-cdn.xitu.io/2018/3/6/161f8d2e90d57677?w=915&h=404&f=png&s=73283)

下面介绍几个核心函数：

*   toPng （包装了draw函数，没啥意义）
*   Draw （dom => canvas）
*   toSvg （dom => svg）
*   cloneNode （clone dom树和css样式）
*   makeSvgDataUri （dom => svg => data:uri）

调用顺序为

    toPng 调用 Draw
    Draw 调用 toSvg
    toSvg 调用 cloneNode
    

**toPng方法：**

```
// 里面其实就是调用了 draw 方法，promise返回的是一个canvas对象
function toPng(node, options) {
    return draw(node, options || {})
        .then(function (canvas) {
            return canvas.toDataURL();
        });
}
```

    

**Draw方法**
```
function draw(domNode, options) {
    // 将 dom 节点转为 svg（data: url形式的svg）
    return toSvg(domNode, options)    
        // util.makeImage 将 canvas 转为 new Image(uri)
        .then(util.makeImage)
        .then(util.delay(100))
        .then(function (image) {
            var canvas = newCanvas(domNode);
            canvas.getContext('2d').drawImage(image, 0, 0);
            return canvas;
        });

    // 创建一个空的 canvas 节点
    function newCanvas(domNode) {
        var canvas = document.createElement('canvas');
        canvas.width = options.width || util.width(domNode);
        canvas.height = options.height || util.height(domNode);
		  ......
        return canvas;
    }
}

```
    

**toSvg方法**

```
  function toSvg (node, options) {
    options = options || {}
    // 设置一些默认值，如果option是空的话
    copyOptions(options)

    return (
      Promise.resolve(node)
        .then(function (node) {
          // clone dom 树
          return cloneNode(node, options.filter, true)
        })
        // 把字体相关的csstext 全部都新建一个 stylesheet 添加进去
        .then(embedFonts)
        // 处理img和background url('')里面的资源，转成dataUrl
        .then(inlineImages)
        // 把option 里面的一些 style 放进stylesheet里面
        .then(applyOptions)
        .then(function (clone) {
          // node 节点序列化成 svg
          return makeSvgDataUri(
            clone,
            // util.width 就是 getComputedStyle 获取节点的宽
            options.width || util.width(node),
            options.height || util.height(node)
          )
        })
    )
	  // 设置一些默认值
    function applyOptions (clone) {
		......
      return clone
    }
  }

```
    

**cloneNode 方法**
```
  function cloneNode (node, filter, root) {
    if (!root && filter && !filter(node)) return Promise.resolve()

    return (
      Promise.resolve(node)
        .then(makeNodeCopy)
        .then(function (clone) {
          return cloneChildren(node, clone, filter)
        })
        .then(function (clone) {
          return processClone(node, clone)
        })
    )
    // makeNodeCopy
    // 如果不是canvas 节点的话，就clone
    // 是的话，就返回 canvas转image的 img 对象
    function makeNodeCopy (node) {
      if (node instanceof HTMLCanvasElement) { return util.makeImage(node.toDataURL()) }
      return node.cloneNode(false)
    }
    // clone 子节点 （如果存在的话）
    function cloneChildren (original, clone, filter) {
      var children = original.childNodes
      if (children.length === 0) return Promise.resolve(clone)

      return cloneChildrenInOrder(clone, util.asArray(children), filter).then(
        function () {
          return clone
        }
      )
      // 递归 clone 节点
      function cloneChildrenInOrder (parent, children, filter) {
        var done = Promise.resolve()
        children.forEach(function (child) {
          done = done
            .then(function () {
              return cloneNode(child, filter)
            })
            .then(function (childClone) {
              if (childClone) parent.appendChild(childClone)
            })
        })
        return done
      }
    }
    
    // 处理添加dom的css，处理svg
    function processClone (original, clone) {
      if (!(clone instanceof Element)) return clone

      return Promise.resolve()
        // 读取节点的getComputedStyle，添加进css中
        .then(cloneStyle)
        // 获取伪类的css，添加进css
        .then(clonePseudoElements)
        // 读取 input textarea 的value
        .then(copyUserInput)
        // 设置svg 的 xmlns
        // 命名空间声明由xmlns属性提供。此属性表示<svg>标记及其子标记属于名称空间为“http://www.w3.org/2000/svg”的XML方言
        .then(fixSvg)
        .then(function () {
          return clone
        })


```
    

> 下面是这篇的重点 把 `html` 节点序列化成 `svg`

```
  // node 节点序列化成 svg
  function makeSvgDataUri (node, width, height) {
    return Promise.resolve(node)
      .then(function (node) {
        node.setAttribute('xmlns', 'http://www.w3.org/1999/xhtml')

        // XMLSerializer 对象使你能够把一个 XML 文档或 Node 对象转化或“序列化”为未解析的 XML 标记的一个字符串。
        // 要使用一个 XMLSerializer，使用不带参数的构造函数实例化它，然后调用其 serializeToString() 方法：
        return new XMLSerializer().serializeToString(node)
      })
      // escapeXhtml代码是string.replace(/#/g, '%23').replace(/\n/g, '%0A')
      .then(util.escapeXhtml)
      .then(function (xhtml) {
        return (
          '<foreignObject x="0" y="0" width="100%" height="100%">' +
          xhtml +
          '</foreignObject>'
        )
      })
      // 变成svg
      .then(function (foreignObject) {
        return (
          '<svg xmlns="http://www.w3.org/2000/svg" width="' +
          width +
          '" height="' +
          height +
          '">' +
          foreignObject +
          '</svg>'
        )
      })
      // 变成 data: url
      .then(function (svg) {
        return 'data:image/svg+xml;charset=utf-8,' + svg
      })
  }


```

参考链接
----

*   [CSSStyleDeclaration.setProperty() - Web API 接口 | MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/CSSStyleDeclaration/setProperty)
    
*   [dom-to-image](https://github.com/tsayen/dom-to-image)
    
*   [XML DOM - XMLSerializer 对象](http://www.w3school.com.cn/xmldom/dom_xmlserializer.asp)

## 简洁的domtoimage

domtoimage 主要代码才700多行，方法和属性都比较少，下载之后看一下就知道怎么用，有些什么功能。虽然html2canvas 代码3000多行，调用其实也是不难，但相对来说代码确实比domtoimage多了很多。

### domtoimage 运用场景
#### 主要的方法有：

domtoimage.toPng(...);将节点转化为png格式的图片

```
var node = document.getElementById('my-node');

domtoimage.toPng(node)
    .then(function (dataUrl) {
        var img = new Image();
        img.src = dataUrl;
        document.body.appendChild(img);
    })
    .catch(function (error) {
        console.error('oops, something went wrong!', error);
    });
```

domtoimage.toJpeg(...);将节点转化为jpg格式的图片

```
domtoimage.toJpeg(document.getElementById('my-node'), { quality: 0.95 })
    .then(function (dataUrl) {
        var link = document.createElement('a');
        link.download = 'my-image-name.jpeg';
        link.href = dataUrl;
        link.click();
    });

```

domtoimage.toSvg(...);将节点转化为svg格式的图片，生成的图片的格式都是base64格式。

```
function filter (node) {
    return (node.tagName !== 'i');
}

domtoimage.toSvg(document.getElementById('my-node'), {filter: filter})
    .then(function (dataUrl) {
        /* do something */
    });

```

domtoimage.toBlob(...);将节点转化为二进制格式，这个可以直接将图片下载，是不是非常方便

```
domtoimage.toBlob(document.getElementById('my-node'))
    .then(function (blob) {
        window.saveAs(blob, 'my-node.png');
    });

```

domtoimage.toPixelData(...);获取原始像素值，以Uint8Array 数组的形式返回，每4个数组元素表示一个像素点，即rgba值。这个方法也是挺实用的，可以用于WebGL中编写着色器颜色。

```
var node = document.getElementById('my-node');

domtoimage.toPixelData(node)
    .then(function (pixels) {
        for (var y = 0; y < node.scrollHeight; ++y) {
          for (var x = 0; x < node.scrollWidth; ++x) {
            pixelAtXYOffset = (4 * y * node.scrollHeight) + (4 * x);
            /* pixelAtXY is a Uint8Array[4] containing RGBA values of the pixel at (x, y) in the range 0..255 */
            pixelAtXY = pixels.slice(pixelAtXYOffset, pixelAtXYOffset + 4);
          }
        }
    });

```

### domtoimage 主要的属性有：

filter ： 过滤器节点中默写不需要的节点；

bgcolor ： 图片背景颜色；

height, width ： 图片宽高；

style ：传入节点的样式，可以是任何有效的样式；

quality ： 图片的质量，也就是清晰度；

cacheBust ： 将时间戳加入到图片的url中，相当于添加新的图片；

imagePlaceholder ： 图片生成失败时，在图片上面的提示，相当于img标签的alt；

[上面的这些摘自 GitHub](https://github.com/tsayen/dom-to-image)

# 对此想补充下html2canvas单例测试结果
1. position + index 这块转化正常、耗时300ms
2. boder上支持不太好
3. transform支持效果的不太好、可以尝试用canvas 去模拟 scale 和 translate、然后进行裁剪
4. 对于text multiple underline线会变细、shadow若color设置为透明，无论设置其他样式转化后都会是透明，或者在字体上加上background-image 和 -webkit-text-fill-color来做文字特效 与 -webkit-text-stroke，这块未做css解析
5. canvas svg img background这几个兼容都不错
6. overflow-transform 无法支持
7. clip 这个支持效果还行
8. text-shadow可以支持，但是文字都会向下移动，其实这块对于文字转化后跟现有位置有所偏差
9. 其实也很明显，html2canvas源码里就很清楚了
![](http://thyrsi.com/t6/648/1546424182x2890211750.png)
他就只有支持了这些属性，如果需要支持更多css属性，需要对此做干预了！

# 总结

* domtoimage 性能还是很不错，优于html2canvas，代码少，性能高，应用简单。
* 感觉这两个插件都很相似，主要但是html2canvas是自己实现了CSS解析，所以在可干预情况下，优于domtoimage，我们可以在它css解析的程度上添加适合业务的更优的css解析方式。
