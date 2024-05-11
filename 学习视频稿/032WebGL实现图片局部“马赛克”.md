## WebGL实现简易的局部“马赛克”

### 前言

接触过Canvas的小伙伴应该都知道，在Canvas2D中我们要加载一个图片很简单，通过调用`drawImage` API就能将图像绘制到画布上，当然在WebGL中我们也可以绘制图像，在绘制时我们需要用到WebGL中的纹理对象，在之前WebGL实现网格背景的文章中，我使用了一个叫做纹理坐标的配置，现在要完成纹理的加载我们也需要用到纹理坐标，并且我们可以通过对纹理坐标处理实现简单的”马赛克“效果。通过对纹理的使用学习，我感觉自己对纹理坐标的认知，和之前学习网格背景时，有点不一样了，这大概就是学习的过程吧，不断更新自己的认知。

接下来我会介绍纹理的基础使用，并在此基础上实现简单的局部“马赛克”效果。



### 创建纹理并绑定到上下文

在Shader中使用纹理之前，我们需要先在JavaScript中创建纹理对象。

```javascript
// 创建纹理对象
const texture = gl.createTexture();
// 坐标翻转
gl.pixelStorei(gl.UNPACK_FLIP_Y_WEBGL, true);
// 将纹理绑定到当前上下文
gl.bindTexture(gl.TEXTURE_2D, texture);
// 在图片加载完毕后，指定纹理图像
gl.texImage2D(
						gl.TEXTURE_2D,
						level,
						internalFormat,
						srcFormat,
						srcType,
						image,
				);
// 设置纹理的一些参数
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR);
```

上述代码中`pixelStorei`方法设置了像素存储格式，它的作用是将图片坐标进行翻转，因为GIF、JPEG和PNG图片使用的坐标系统以左上角为原点，X轴水平向右，Y轴垂直向下，而纹理坐标是左下角为原点，Y轴垂直向上，两者Y轴的方向相反，所以为了使图片在WebGL中正常显示，需要这一步操作。

还有一点需要注意的是，在WebGL中使用的图片需要和当前页面同源。



### 使用纹理

完成纹理的创建后，我们就可以通过纹理单元编号激活指定纹理，来在WebGL中使用。

```javascript
// 按照单元编号激活纹理
gl.activeTexture(gl.TEXTURE0);
// 将纹理绑定到上下文
gl.bindTexture(gl.TEXTURE_2D, texture);
// 获取Shader中纹理变量
const loc = gl.getUniformLocation(program, "tMap");
// 将对应的纹理单元写入Shader变量
gl.uniform1i(loc, 0);
```

默认情况下WebGL会使用第一个纹理，所以如果我们只有一个纹理的话，不调用`activeTexture`方法也能正常使用纹理。

激活纹理后，我们就可以将对应的纹理单元写入Shader变量，这样就可以在Shader中使用纹理了，可以看到这里向Shader传递的并不是纹理对象，而是纹理单元编号。



### 加载纹理

等到JavaScript传递纹理信息后，Shader就可以使用这个纹理了。

```glsl
precision mediump float; // 添加如下精度描述
uniform sampler2D tMap; // 纹理相关

varying vec2 vUv;

void main() {
  gl_FragColor = texture2D(tMap, vUv);
}
```

在片元着色器中，我们使用sampler2D类型的变量接收纹理信息。

可以看到在GLSL代码中，我们使用了一个叫texture2D的函数，这个函数的作用是根据纹理坐标，从纹理中提取颜色。

通过对纹理的简单使用，我目前的理解是，纹理坐标是与顶点坐标存在一个对应的关系。

```javascript
// ...
let vertices = new Float32Array([ // 顶点坐标
      [-1, -1],
      [-1, 1],
      [1, 1],
      [1, -1]
    ].flat()),
// ...
let vertices = new Float32Array([ // 纹理坐标
      [0, 0],
      [0, 1],
      [1, 1],
      [1, 0]
    ].flat()),
```

两组坐标是对应的，所以在顶点着色器中虽然我们只是指定了顶点，但根据对应关系，此时顶点对应的纹理坐标也是已知的。

因为前面对纹理的参数设置（`gl.CLAMP_TO_EDGE`），纹理是拉伸铺在整个纹理坐标上。（我还没深入学习纹理的参数，这里我只是根据单词意思做的猜测。）

所以就可以通过`texture2D`函数根据纹理坐标提取到图像对应位置的像素信息了，也就是颜色色值，并将它赋值给gl_FragColor常量、给片元上色。

至此我们就实现了简单的纹理加载，将图像绘制到WebGL的画布上了。

接下去我们就来实现图片的局部“马赛克”效果。

因为对于纹理的具体使用步骤我们已经知道了，所以在接下去的例子中，我就使用课程提供的`gl-renderer`库来简化纹理的加载使用操作，专注于效果的实现。



### 实现局部“马赛克”

在处理照片时，我们常常需要将一些敏感的或者是不想展示的信息使用马赛克的效果处理掉，那么在WebGL中我们要怎么去实现呢？

* 首先我们设置马赛克效果的中心点，对应的是纹理坐标的值。

    ```javascript
    renderer.uniforms.center = [-2.0, -2.0];
    ```
    
    初始中心点随意设置一个在0-1之外的位置。
    
* 接着设置马赛克的范围。

    ```javascript
    const radiusPX = 100;
    renderer.uniforms.radiusX = radiusPX / canvasRef.value.width;
    renderer.uniforms.radiusY = radiusPX / canvasRef.value.height;
    ```
    
    我们将马赛克的半径范围设置为100px，并将它转换为WebGL内对应的数值，使用uniform传递给Shader。
    
* 然后我们添加鼠标点击事件的监听。

    ```javascript
    const clickHandler = e => {
      e.preventDefault();
      const {width, height} = canvasRef.value.getBoundingClientRect();
      const {offsetX: x, offsetY: y} = e;
      // 转换为纹理坐标上的值
      const center = [];
      center[0] = x / width;
      center[1] = (height - y) / height;
      renderer.uniforms.center = center;
    };
    ```

    offsetX和offsetY分别表示鼠标位置距离元素左边和顶部的距离，所以`height-y`表示鼠标位置距离元素底部多远，通过分别除以宽和高获得鼠标在WebGL纹理坐标上的值。

    这样通过监听鼠标点击事件，我们就可以动态更新马赛克的位置。

* 完成uniform的传递后，我们就可以在片元着色器中使用了。

    首先我们对片元对应的纹理坐标进行缩放，X坐标放大50倍，Y坐标放大27.7倍，与画布的宽高比例一致，得到50乘以27.7，也就是1350个20x20大小的方格；同时获取到X和Y坐标的整数部分，整数部分相当于片元所在方格在横纵坐标方向的索引。

    ```glsl
    vec2 st = vUv * vec2(50, 27.7);
    vec2 uv = floor(st);
    ```

    接着根据原始纹理坐标的位置与中心点的距离，我们使用椭圆的公式来判断片元是否在马赛克范围内。（因为画布宽高不一样，所以这里我们用椭圆公式判断。）

    ```glsl
    // 中心点坐标
    float x0 = center.x;
    float y0 = center.y;
    
    if (pow(abs(vUv.x - x0), 2.0) / pow(radiusX, 2.0) + pow(abs(vUv.y - y0), 2.0) / pow(radiusY, 2.0) <= 1.0) {
      color = texture2D(tMap, vec2(uv.x / 50.0, uv.y / 27.7));
    } else {
      color = texture2D(tMap, vUv);
    }
    ```

    如果是在范围内的片元，我们就将方格的索引进行缩放，对应到原始纹理坐标上的值，因为一个方格内所有的片元对应的索引值一致，所以按照这个值提取到的颜色是一样的，也就是一个方格内是一种颜色。

    如果不在马赛克范围内，就按照普通的提取纹理色值的方式。

* 最后就是将颜色值赋值给常量`gl_FragColor`，完成了给片元上色。

到这里我们就实现了一个简单的马赛克效果，可以通过点击鼠标给图片指定位置添加马赛克效果，是一个圆形的样子，我们可以通过使用不同的公式判断，呈现不同的马赛克形状，比如正方形。



### 总结

在WebGL中纹理也是比较重要的内容，可以让我们使用图片，最早我是在JavaScript高级程序设计这本书中接触到纹理的，但是因为书里给出的代码并不完整，并且我当时也没去深入了解，所以当时的代码并没有跑起来，现在我通过学习一个可视化教程才知道说纹理要怎么去用，了解到通过不同参数的设置可以实现不同的纹理表现，呈现不同的视觉效果，本期内容只是简单的纹理使用，对纹理感兴趣的小伙伴可以再自己深入研究。
