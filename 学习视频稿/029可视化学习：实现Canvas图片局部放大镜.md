## 实现Canvas图片局部放大镜

### 前言

最近我在可视化课程中学习了如何在Canvas中利用像素处理来实现滤镜效果，在这节课程的结尾留了一道局部放大镜的题目，提示我们用像素处理的方式去实现这个效果，最终实现随着鼠标移动将图片局部放大，本着把学到的内容落地实践的想法，我就去思考了一番，但很不幸，我思考了好几天也没思考出结果，因为刚开始我想的一直是在一个Canvas上来操作，但是一来我对Canvas API还并不是很熟悉，二来我对像素处理还不够熟练，然后第三是如果原图的部分像素被处理了，那下一次放大就会有问题，因此我最终放弃了这个思路，选择了再增加一个Canvas来完成最终的效果，以下就是利用这种方式实现图片局部放大的效果。



### 像素处理

在实现这个效果之前，我们先来了解一下如何处理像素，有些小伙伴可能不太清楚，所以这里简单说一下，在屏幕上我们知道所有显示的内容都是由像素点组成的，那么在处理像素之前，我们需要先获取到像素信息，那么Canvas就是提供了一个API叫做getImageData让我们可以获取到画布上的像素信息，最终这个API返回的是一个**ImageData类型**的值，关于这个API的具体描述可以参考对应的[MDN页面](https://developer.mozilla.org/zh-CN/docs/Web/API/CanvasRenderingContext2D/getImageData)。

ImageData类型的数据包含三个属性，包括data、width、height。width和height简单来说，就是被提取像素信息的区域的宽高，最主要的像素信息是在这个data属性中。data属性指向一个数组类型的值，准确来说是Uint8ClampedArray的实例，Uint8ClampedArray表示**8 位无符号整型固定数组**，也就是说其中的元素是0到255之间的整数，我们知道一个像素的颜色信息可以使用rgba四个分量表示，那么我们就得出在data数组中每四个元素就能表示一个像素点的信息，因此data数组的长度就是`width * height * 4`，通过操作这个数组的元素就能对像素进行处理。

了解完像素处理，我们就可以开始进行具体的实现了。



### 具体实现

```html
<canvas ref="canvasRef" width="0" height="0"></canvas>
<canvas ref="magnifier" width="0" height="0"></canvas><!-- 放大镜 -->
```

#### 1. 准备工作

在实现放大效果之前，我们需要先把图片加载到Canvas上：

```javascript
(async function() {
  const img = await loadImage('src/assets/girl1.jpg');
  canvasRef.value.width = img.width;
  canvasRef.value.height = img.height;
  context.drawImage(img, 0, 0);
}());
```

这里`loadImage`方法是通过Image对象来异步加载图片，然后通过drawImage方法将图片绘制到画布上。

接着设置一个要放大的区域，也就是以鼠标坐标为中心，多少半径以内的内容要被放大，这里我设置一个变量originSize用于存储原图大小，并设置一个5倍的放大倍数。

```javascript
let originSize = 40; // 原图大小
let zoom = 5; // 放大倍数

(async function() {
  // ...
  magnifier.value.width = originSize * zoom;
  magnifier.value.height = originSize * zoom;
}());
```

用作于放大镜的magnifier，我们使用`originSize * zoom`来设置它的宽高。

#### 2. 鼠标移动事件监听

接下来就是主要的代码实现。

首先是添加鼠标移动事件的监听：

```javascript
const addEvent = () => {
  canvasRef.value.addEventListener('mousemove', mouseDownHandler);
};

addEvent();
```

然后我们就来实现`mouseDownHandler`函数。

* 首先我们获取鼠标坐标在Canvas中的相对坐标，并通过`Math.floor`取整

  ```javascript
  const mouseDownHandler = e => {
    // 相对于画布的坐标
    const center = {
      x: Math.floor(e.pageX - left),
      y: Math.floor(e.pageY - top)
    };
  };
  ```

* 然后利用`getImageData`方法获取指定区域的像素信息，这里我们用到了[`OffscreenCanvas`](https://developer.mozilla.org/zh-CN/docs/Web/API/OffscreenCanvas)，它提供了一个可以脱离屏幕渲染的 canvas 对象，可以提升渲染性能；这样我们就得到了待放大区域的像素信息。

  ```javascript
  const mouseDownHandler = e => {
    // 相对于画布的坐标
    // ...
    // 待放大区域的imageData
    const originImageData = getImageData(img, [center.x - originSize / 2, center.y - originSize / 2, originSize, originSize]);
  };
  ```

* 现在我们需要一个ImageData类型的变量，用于存储放大后的像素信息，因为最终要渲染到magnifier这个Canvas上，我们就使用magnifier的2d上下文对象调用`createImageData`方法来创建一个ImageData对象，关于这个方法的使用具体可查看[MDN文档](https://developer.mozilla.org/zh-CN/docs/Web/API/CanvasRenderingContext2D/createImageData)。

  ```javascript
  const mouseDownHandler = e => {
    // 相对于画布的坐标
    // ...
    // 待放大区域的imageData
    // ...
    // 构建一个imageData
    const areaImageData = mContext.createImageData(magnifier.value.width, magnifier.value.height);
  };
  ```

* 接下来就是具体的像素遍历和处理，按照areaImageData的宽高来进行遍历，这里迭代的增量使用`+zoom`是因为，当我们放大zoom倍数之后，原图1个像素的信息在magnifier使用`zoom*zoom`个像素来放大，也就是`zoom*zoom`个像素点的色值和原图中对应的那个像素的色值是一样的。在我们这段代码中设置zoom为5，也就是放大后使用5*5=25个像素点表示之前的一个像素点。

  ```javascript
  const mouseDownHandler = e => {
    // 相对于画布的坐标
    // ...
    // 待放大区域的imageData
    // ...
    // 构建一个imageData
    // ...
    let count = 0;
    for (let j = 0; j < originSize * zoom; j += zoom) {
      for (let i = 0; i < originSize * zoom; i += zoom) {
  
        // ...
  
      }
    }
  };
  ```

* 所以我们继续使用两个for循环k和m，把areaImageData的data数组中的对应元素赋值为原图对应像素的色值，完成赋值后我们就可以通过putImageData方法将像素信息渲染到magnifier画布上。

  ```javascript
  const mouseDownHandler = e => {
    // 相对于画布的坐标
    // ...
    // 待放大区域的imageData
    // ...
    // 构建一个imageData
    // ...
    let count = 0;
    for (let j = 0; j < originSize * zoom; j += zoom) {
      for (let i = 0; i < originSize * zoom; i += zoom) {
  
        for (let k = j; k < j + zoom; k ++) {
          for (let m = i; m < i + zoom; m ++) {
            const index = (k * originSize * zoom + m) * 4;
            areaImageData.data[index] = originImageData.data[count];
            areaImageData.data[index + 1] = originImageData.data[count + 1];
            areaImageData.data[index + 2] = originImageData.data[count + 2];
            areaImageData.data[index + 3] = originImageData.data[count + 3];
  
          }
        }
        count += 4;
  
      }
    }
    mContext.putImageData(areaImageData, 0, 0);
  };
  ```

至此我们就实现了基本的局部放大，但现在放大镜不在原图Canvas的上方，并且放大镜是一个正方形，我们继续简单优化一下。

#### 3. 简单优化

* 首先因为我对Canvas API还不太熟悉，所以我现在通过css把放大镜改为圆形，并加上一个阴影`box-shadow`来优化视觉效果。

  ```css
  #magnifier {
    position: absolute;
    box-shadow: 0 0 10px 4px rgba(12, 12, 12, .5);
    border-radius: 50%;
  }
  ```

* 然后给两个Canvas外层加一个div容器，把放大镜设置绝对定位，把它放到鼠标坐标的位置，在鼠标移动过程中更新放大镜的位置。

  ```html
  <div class="canvas-container" ref="containerRef" :style="{width: containerWidth + 'px'}">
    <canvas ref="canvasRef" width="0" height="0"></canvas>
    <canvas ref="magnifier" width="0" height="0" id="magnifier" :style="position"></canvas>
  </div>
  ```

  ```javascript
  const position = reactive({
    left: 0,
    top: 0
  });
  const containerWidth = ref(0);
  
  containerWidth.value = img.width;
  // 在鼠标移动过程中更新放大镜的位置
  position.top = (center.y - originSize * zoom / 2) + 'px';
  position.left = (center.x - originSize * zoom / 2) + 'px';
  ```

  ```css
  .canvas-container {
    position: relative;
    overflow: hidden;
  }
  ```

* 这个时候放大镜的位置就和我们预想的一致了，但是现在还有一个问题，就是放大镜在原图的上方，在移动的过程中会看到放大镜的渲染有点卡顿，这是因为鼠标移动事件是加在原图Canvas上的，当鼠标悬浮在放大镜上时，这个移动事件的监听就不连贯了，此时我们可以考虑把鼠标移动监听加改为加在外层容器上，这样就能看到移动过程中放大镜的渲染是比较流畅了。

  ```javascript
  const addEvent = () => {
    containerRef.value.addEventListener('mousemove', mouseDownHandler);
  };
  ```

至此就完成了简单的局部放大效果，虽然还存在一些问题吧。



### 总结

以上的实现比较简单粗暴，就是遍历imageData然后赋值，不算什么很高明的思路，就当作是抛砖引玉吧。



[最终效果](https://yeying0827.github.io/visualization-demos/#/demo-magnifier)

[完整代码](https://github.com/yeying0827/visualization-demos/blob/main/src/pages/DemoMagnifier.vue)

