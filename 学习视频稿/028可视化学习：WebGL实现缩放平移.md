## 可视化学习：WebGL实现缩放平移

### 前言

在上篇文章中，我们使用WebGL实现了网格背景，当时有提到说使用WebGL来实现的好处之一，是网格背景可以与画布上的其他元素更好地融合，比如一起缩放平移，那么在WebGL中怎么实现缩放和平移呢？现在我们已经实现了网格背景，接下来我们就用网格背景作为例子来了解一下WebGL中的缩放和平移。

这里需要用到我之前学过的变换矩阵，不熟悉变换矩阵的小伙伴可以看我之前的文章《CSS transform与仿射变换》，或者查阅其他相关资料；简单来说就是使用矩阵来表示顶点坐标的变换。



### 具体实现

接下来我们在代码中来具体实现这个效果。

#### 1. 改造顶点着色器

首先我们先改造顶点着色器的GLSL代码，对顶点应用变换矩阵。

```glsl
attribute vec2 a_vertexPosition;
attribute vec2 uv;

varying vec2 vUv;

uniform int scale;
uniform vec2 offset;

mat3 translateMatrix = mat3( // 平移矩阵
  1.0, 0.0, 0.0, // 第一列
  0.0, 1.0, 0.0, // 第二列
  offset.x, offset.y, 1.0 // 第三列
);

mat3 scaleMatrix = mat3( // 缩放矩阵
  float(scale), 0.0, 0.0,
  0.0, float(scale), 0.0,
  0.0, 0.0, 1.0
);

void main() {
  gl_PointSize = 1.0;
  vUv = uv;
  vec3 pos = scaleMatrix * translateMatrix * vec3(a_vertexPosition, 1.0);
  gl_Position = vec4(pos, 1.0);
}
```

代码中我们增加接收两个uniform常量`scale`和`offset`，表示缩放比例和平移的偏移量；并且定义两个矩阵：平移矩阵`translateMatrix`和缩放矩阵`scaleMatrix`。需要注意GLSL这里矩阵的写法是列主序的，也就是说这里的第一行其实是平常我们认为的第一列，也就是说假如有平移矩阵乘以原坐标：`translateMatrix * vec3(a_vertexPosition, 1.0)`，我们会得到如下结果：
$$
\begin{bmatrix}
 1 & 0 & offset.x \\
 0 & 1 & offset.y \\
 0 & 0 & 1
\end{bmatrix} \times
\begin{bmatrix}
 x_0 \\
 y_0 \\
 1
\end{bmatrix}  =
\begin{bmatrix}
 x_0 + offset.x\\
 y_0 + offset.y \\
 1
\end{bmatrix}
$$


将两个矩阵相乘表示叠加效果，再与坐标向量相乘后，我们就能得到映射的新顶点坐标。

#### 2. 改造js代码

顶点着色器中增加接收的两个参数需要js代码传递过去，接下来我们就对js代码进行改造。

首先是设置这两个变量的初始值，scale等于1表示原始大小，offset设置的初始偏移量为0，也就是不偏移。

```javascript
// ...
renderer.uniforms.scale = 1;
renderer.uniforms.offset = [0.0, 0.0];
// ...
```

然后我们添加对鼠标滚轮事件以及鼠标按下和放开事件的监听。

```javascript
// ...
const addEvent = () => {
  patternPracticeRef.value.addEventListener('mousewheel', wheelEventHandler);
  patternPracticeRef.value.addEventListener('mouseup', mouseUpHandler);
  patternPracticeRef.value.addEventListener('mousedown', mouseDownHandler);
}
addEvent();
// ...
```

接下来我们就来完成这三个事件监听的回调函数。

我们先来完成滚轮事件的监听回调，这个比较简单。`e.wheelDeltaY > 0`表示滚轮向下滚动，实现放大效果，这里我使用了整数表示放大倍数，这是因为之前我有使用浮点数来实现过，但是存在一些问题，比如绘制出来的网格线会有可能粗细不同，甚至会缺少一些线，这估计是由于精度原因导致的，我还没去进一步的研究。

```javascript
const wheelEventHandler = e => {
  e.preventDefault();
  if (e.wheelDeltaY > 0) { // 向下滚，放大
    if (renderer.uniforms.scale <= 50) {
      renderer.uniforms.scale += 1;
    }
  } else { // 向上滚，缩小
    if (renderer.uniforms.scale > 1) {
      renderer.uniforms.scale -= 1;
    }
  }
}
const mouseDownHandler = e => {
  // ...
}
const mouseUpHandler = e => {
	// ...
}
```

此时我们在页面上进行测试一下，就可以看到缩放的效果实现了。

接着我们来实现平移的效果，因为偏移量需要根据鼠标的初始位置和移动结束的位置计算得到，所以我们增加一个对象类型的变量lastPos来记录鼠标的初始位置，并在鼠标按下的事件回调中对它赋值。

```javascript
// ...
const lastPos = {};
// ...
const mouseDownHandler = e => {
  e.preventDefault();
  // 记录初始位置
  lastPos.x = e.offsetX;
  lastPos.y = e.offsetY;
  // 绑定事件
  patternPracticeRef.value.addEventListener('mousemove', mouseMoveHandler);
}
```

在鼠标按下后，我们对鼠标的移动事件开启监听，便于实时更新位置信息。

```javascript
const mouseMoveHandler = e => {
  e.preventDefault();
  // 计算坐标差值并转换为Canvas差值
  const { offsetX: x, offsetY: y } = e;
  const translateX = (x - lastPos.x) / patternPracticeRef.value.width;
  const translateY = (lastPos.y - y) / patternPracticeRef.value.height;
  // 设置偏移量
  renderer.uniforms.offset = [translateX, translateY];
}
```

因为在页面上y轴向下，与WebGL中y轴向上是相反的，所以这里计算y方向的偏移量使用`lastPos.y - y`。我们通过除以canvas的宽高，就能转换得到偏移量在WebGL里的对应数值。

接着我们在鼠标按键松开事件监听的回调中，对移动监听进行移除。

```javascript
const mouseDownHandler = e => {
	e.preventDefault();
  // 解绑事件
  patternPracticeRef.value.addEventListener('mousemove', mouseMoveHandler);
}
```

此时在页面上测试，能看到我们可以对网格进行拖动平移了，但很明显这其中还存在问题，当我们完成第一次平移后，想再点击鼠标来平移，就会发现在按下鼠标的那一刻网格会发生闪烁，这是因为在上一次平移完成后，图片的中心点已经改变了，而在GLSL代码中的偏移矩阵中，偏移量的设置是相对`(0, 0)`点的，所以会闪烁一下，也就是说我们需要考虑中心点的影响，在第二次平移时要在新的中心点的基础上去进行平移，在这里我使用一个lastCenter变量来存储中心点的位置。

```javascript
const lastPos = {}, lastCenter = {};
```

并在鼠标按键松开的事件回调中对画布中心点的信息进行更新。

```javascript
const mouseUpHandler = e => {
  e.preventDefault();
  // 计算坐标差值并转换为Canvas差值
  const { offsetX: x, offsetY: y } = e;
  const translateX = (x - lastPos.x) / patternPracticeRef.value.width;
  const translateY = (lastPos.y - y) / patternPracticeRef.value.height;
  // 更新新的中心点
  lastCenter.x = translateX + lastCenter.x;
  lastCenter.y = translateY + lastCenter.y;
  // 解除事件绑定
  patternPracticeRef.value.removeEventListener('mousemove', mouseMoveHandler);
}
```

同时在鼠标移动过程中偏移量的更新我们也要把中心点的坐标考虑进去。

```javascript
const mouseMoveHandler = e => {
  e.preventDefault();
  // 计算坐标差值并转换为Canvas差值
  const { offsetX: x, offsetY: y } = e;
  const translateX = (x - lastPos.x) / patternPracticeRef.value.width;
  const translateY = (lastPos.y - y) / patternPracticeRef.value.height;
  // 设置偏移量
  renderer.uniforms.offset = [translateX + lastCenter.x, translateY + lastCenter.y];
}
```

这个时候我们再去测试一下，可以看到就是正常的了。



### 总结

到这里为止我们就实现了在WebGL中的缩放和平移，这里主要就是借助了鼠标事件的监听和变换矩阵的应用。

