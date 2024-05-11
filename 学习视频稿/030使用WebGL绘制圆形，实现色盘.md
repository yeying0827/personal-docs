## 使用WebGL绘制圆形，实现色盘

### 前言

在Canvas2D中实现圆形的绘制比较简单，只要调用`arc`指令就能在Canvas画布上绘制出一个圆形，类似的，在SVG中我们也只需要一个`<circle>`标签就能在页面上绘制一个圆形。那么在WebGL中我们要怎么去绘制呢？WebGL只能绘制三种形状：点、线段和三角形，它没有提供直接绘制圆形的功能，当然也无法像SVG一样使用标签，所以我们是无法直接绘制圆形曲线的，这个时候我们可以借助相关的数学知识，来实现圆形的绘制。



### 参数方程

相信数学基础好的小伙伴一定能很快想到，我们可以使用参数方程去获取圆形曲线上的点的坐标，只要我们收集足够多的点，再通过绘制线段的方式将这些点连接起来，就能得到接近圆的图形，从视觉上看就是一个圆形了。其实圆形就是曲线中的一个特例，所以也就是说我们还可以通过参数方程绘制其他常见的曲线，比如圆、椭圆、抛物线、正余弦曲线等等。

以下是圆的参数方程：
$$
\begin{cases}
 x = x0 + r * cos(θ)\\
 y = y0 + r * sin(θ)\\
 \end{cases}
$$

在圆的参数方程中，可以使用圆心坐标、半径和夹角的正余弦值来表示横纵坐标的值。




### 具体实现

按照这个思路，我们就可以编写代码来绘制圆形曲线了。

在正式实现之前，在HTML中准备一个Canvas：

```html
<canvas ref="webglRef" width="256" height="256"></canvas>
```

在之后的代码中会用到我自己之前简单封装的一个WebGL的类，只是封装了一些繁琐的创建着色器程序的步骤，封装的比较粗糙。下面就开始具体的实现。

* 首先，定义函数获取圆形曲线的顶点集合。

  ```javascript
  const TAU_SEGMENTS = 60;
  const TAU = Math.PI * 2;
  // 获得圆形曲线顶点集合
  function arc(x0, y0, radius, startAng = 0, endAng = Math.PI * 2) {
    const ang = Math.min(TAU, endAng - startAng);
    const ret = ang === TAU ? []: [[x0, y0]];
    const segments = Math.round(TAU_SEGMENTS * ang / TAU);
    for (let i = 0; i <= segments; i ++) {
      const x = x0 + radius * Math.cos(startAng + ang * i / segments);
      const y = y0 + radius * Math.sin(startAng + ang * i / segments);
      ret.push([x, y]);
    }
    return ret;
  }
  ```

  x<sub>0</sub>和y<sub>0</sub>是圆心坐标，radius是半径，startAng和endAng表示圆弧的起始角度和结束角度，对于整个圆来说，就是从0到2π，这些参数都比较好理解。

  再来看arc这个函数的内部变量，ang好理解，就是结束角度和起始角度的差值；segments表示要在圆弧上取的点的总数，如果是整个圆就取60个点。

  接着就是遍历，获取segments数量的点的坐标，并存储在ret数组中。

* 这样，我们就可以调用`arc`函数来获取顶点集合了。

  ```javascript
  const vertices = arc(0, 0, 0.8);
  ```

  因为在WebGL中坐标系在视口的坐标范围默认是-1到1，要在视口中看到整个圆，这个圆的半径不能超过1，所以这里半径我取0.8，圆心为`(0, 0)`，然后获取到顶点集合。

* 创建WebGL程序并绘制。

  WebGL部分的代码就比较简单了，首先是两段GLSL代码，和常见的实现三角形的GLSL代码没什么太大区别：

  ```javascript
  const vertex = `
    attribute vec2 position;
  
    void main() {
      gl_PointSize = 1.0;
      gl_Position = vec4(position, 1, 1);
    }
  `;
  const fragment = `
    precision mediump float;
  
    void main() {
      gl_FragColor = vec4(0, 0, 0, 1);
    }
  `;
  ```

  因为通过参数方程获取到的是连续的点，所以我们可以通过`gl.LINE_LOOP`的绘图模式，将所有的点串联起来，这样就得到了一个视觉上的圆形曲线。

  ```javascript
  const gl = webglRef.value.getContext('webgl');
  const webgl = new WebGL(gl, vertex, fragment);
  webgl.drawSimple(vertices.flat(), 2, gl.LINE_LOOP);
  ```

  具体在封装的`drawSimple`方法中我调用了`gl.drawArrays`来绘制图形。

  ```javascript
  gl.drawArrays(gl.LINE_LOOP, 0, points.length / size);
  ```

实际操作下来能发现，其实绘制圆形曲线还比较简单，所以我们还可以尝试去实现色盘。

色盘是一个实心的圆，就不能通过线条的方式去绘制了，之前在《利用向量判断多边形边界》中我们有提到过，对于多边形我们可以把它们看做是由多个三角形组合而成的图形，因此我们可以对多边形进行三角剖分，也就是使用多个三角形的组合来表示一个多边形，把这些三角形都绘制到画布上就组成了多边形，而圆形我们就可以把它看做是一种特殊的多边形。

因为三角剖分算法比较复杂，我们可以直接调用现有的库来完成这个操作，之前使用的是`earcut`这个库，现在我们换一个叫`TESS2`的库，更详细的介绍可以查看它的[github仓库](https://github.com/memononen/tess2.js)，下面我们就调用TESS2的API来完成三角剖分操作。

```javascript
webgl.drawPolygonTess2(vertices);
// ↓↓ 
drawPolygonTess2(points, {
    color,
    rule = WINDING_ODD/*WINDING_NONZERO*/
} = {}) {
    const triangles = tess2Triangulation(points, rule);
    triangles.forEach(t => this.drawTriangle(t, {color}));
}
// ↓↓
function tess2Triangulation(points, rule = WINDING_ODD) {
    const res = tesselate({
        contours: [points.flat()],
        windingRule: rule,
        elementType: POLYGONS,
        polySize: 3,
        vertexSize: 2,
        strict: false
    });
    const triangles = [];
    for (let i = 0; i < res.elements.length; i += 3) {
        const a = res.elements[i];
        const b = res.elements[i + 1];
        const c = res.elements[i + 2];
        triangles.push([
            [res.vertices[a * 2], res.vertices[a * 2 + 1]],
            [res.vertices[b * 2], res.vertices[b * 2 + 1]],
            [res.vertices[c * 2], res.vertices[c * 2 + 1]],
        ])
    }
    return triangles;
}
```

这样我们就绘制了一个黑色的实心圆。

要实现色盘，我们需要使用HSV或者HSL的颜色表示形式，因为色相Hue的取值范围是0到360度，所以这两种颜色表示形式可以让我们直接把色值和角度关联起来，因此我们可以通过varying变量将坐标信息传递给片元着色器，然后在片元着色器中使用坐标信息计算hsv形式的像素色值。

```glsl
// vertex
attribute vec2 position;
varying vec2 vP;

void main() {
  gl_PointSize = 1.0;
  gl_Position = vec4(position, 1, 1);
  vP = position;
}
```

但是WebGL中还无法直接处理HSV的颜色表示形式，所以我们需要使用`hsv2rgb`函数来完成颜色向量的转换，这其中具体的转换算法我也并不是很懂，感兴趣的小伙伴可以自行研究。

```glsl
// fragment
#define PI 3.1415926535897932384626433832795
precision mediump float;

varying vec2 vP;

// hsv -> rgb
// 参数的取值范围都是 (0, 1)
vec3 hsv2rgb(vec3 c) {
  vec3 rgb = clamp(abs(mod(c.x * 6.0 + vec3(0.0, 4.0, 2.0), 6.0) - 3.0) - 1.0, 0.0, 1.0);
  rgb = rgb * rgb * (3.0 - 2.0 * rgb);
  return c.z * mix(vec3(1.0), rgb, c.y);
}

void main() {
  float x0 = 0.0;
  float y0 = 0.0;
  float h = atan(vP.y - y0, vP.x - x0);
  h = h / (PI * 2.0); // 归一化处理
  vec3 hsv_color = vec3(h, 1.0, 1.0);
  vec3 rgb_color = hsv2rgb(hsv_color);
  gl_FragColor = vec4(rgb_color, 1.0);
}
```

在上述代码中，我们调用atan函数计算得到以`(0,0)`为圆心的弧度值，再除以`2π`得到一个归一化的值，然后将这个归一化的值通过`hsv2rgb`函数转化RGB颜色向量。

这样我们就使用WebGL实现了一个色盘。如果我们想要颜色的过渡显得更自然，还可以设置使饱和度随着半径增大而增大。

```glsl
void main() {
  // ...
  float r = sqrt((vP.x - x0) * (vP.x - x0) + (vP.y - y0) * (vP.y - y0)); // 计算半径

  vec3 hsv_color = vec3(h, r * 1.2, 1.0);
  // ...
}
```

好啦，那看到这里的小伙伴应该都知道如何绘制圆形，如何实现色盘了吧，可以自己动手实践一下。

