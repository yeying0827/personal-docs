## WebGL的基础使用

### 引言

继续复习可视化的内容。WebGL和其他Web端的图形系统存在很大的不同，是OpenGL ES规范在浏览器的实现，它最大的不同就在于它更接近底层，可以由开发者直接操作GPU来实现绘图，性能很好，可以充分利用GPU并行计算的能力，并且WebGL还支持3D物体的渲染；WebGL最大的缺点应该就在于它的使用比较复杂，不易掌握，不同于一般的Web API使用，想要掌握好WebGL，还需要了解与WebGL相关的GLSL语言。



### 着色器

想要在WebGL中绘图，离不开着色器的使用，着色器是什么呢，我觉得可以简单理解为，着色器定义了如何去处理画布上的坐标点。在WebGL中有两类着色器，一类是顶点着色器，一类是片元着色器（或者也可以说片段着色器）。

顶点着色器可以认为是，声明了需要处理的坐标点；片元着色器就是定义了将这个待处理的坐标点渲染为什么颜色。当然这只是我目前学下来的一个理解，一种感觉，不一定准确。

这两类着色器都是由GPU调用的。

GPU会根据着色器程序，以及传入的数据，对批量的坐标点并行进行处理，最后渲染为一个图形。WebGL最大的特点就在于对批量的坐标点应用同一个着色器程序。所以通常来说，要绘制的坐标点和绘制的颜色，尤其是坐标点，一般不会直接写在GLSL代码中，而是由GPU从缓存中读取相关信息；所以在GPU读取之前，我们需要通过代码将数据写入缓存。



### 基础使用

接下来我们通过绘制一个简单的三角形来体会WebGL的使用，来了解如何使用WebGL来绘图。

首先我们在页面上准备一个canvas画布。

```html
<canvas width="512" height="512"></canvas>
```

```css
canvas {
  width: 512px;
  height: 512px;
  border: 1px solid #eee;
}
```

接下来我们就开始编写JavaScript代码。

#### 1. 获取WebGL上下文

首先是最基本的，获取WebGL上下文。

```javascript
const canvas = document.querySelector('canvas');
const gl = canvas.getContext('webgl');
```

#### 2. 编写着色器程序

然后是最关键的，编写着色器程序。

* 编写着色器程序的**第一步，是编写GLSL代码**，来定义两个着色器。

定义着色器有两种方式，可以直接通过一段字符串定义，也可以通过使用自定义type属性的script元素把GLSL代码包含在网页中；以下我们通过字符串来定义。

```javascript
const vertex = `
	attribute vec2 position;
	
	void main() {
		gl_Position = vec4(position, 0.0, 1.0);
	}
`;
```

vertex变量定义的是顶点着色器的GLSL代码。

attribute表明变量是专门用于接收顶点数据的；

vec2是变量类型，表示变量是一个包含两个元素的数组，两个元素分别代表x和y坐标；

position不用说，就是变量的标识。

在运行着色器程序时，会对每一个待处理的顶点执行main函数，将position通过vec4创建一个包含4个元素的数组，把2D坐标转换为3D坐标，并赋值给gl_Position。gl_Position是内建变量，是四维向量，为什么3D坐标要用四维向量表示呢？这是因为顶点有可能会需要做一些坐标变换的操作。

gl_Position就是最终要渲染的点的坐标。

```javascript
const fragment = `
	precision mediump float;
	
	void main() {
		gl_FragColor = vec4(1.0, 0.0, 0.0, 1.0);
	}
`;
```

fragment变量定义的是片元着色器的GLSL代码。

`precision mediump float;`这是对精度的描述，不添加会报错；

gl_FragColor也是内建变量，是四维向量，代表待渲染的点的颜色信息。vec4中的四个元素分别对应RGB三个颜色通道的色阶和alpha通道，与CSS中的RGBA不同的点在于，这里的RGB的取值在0.0到1.0之间。

* 接下来我们就来**创建着色器程序**。

先是分别定义两个GLSL代码对应的shader对象，并把GLSL代码传递给shader对象，然后编译这两个shader对象，这两个着色器。

```javascript
const vertexShader = gl.createShader(gl.VERTEX_SHADER);
gl.shaderSource(vertexShader, vertex);
gl.compileShader(vertexShader);
const fragmentShader = gl.createShader(gl.FRAGMENT_SHADER);
gl.shaderSource(fragmentShader, fragment);
gl.compileShader(fragmentShader);
```

然后是创建Program对象并关联这两个shader对象，将两个shader对象链接到着色器程序。

```javascript
const program = gl.createProgram();
gl.attachShader(program, vertexShader);
gl.attachShader(program, fragmentShader);
gl.linkProgram(program);
```

最后使WebGL上下文使用这个program程序。

```javascript
gl.useProgram(program);
```

至此，GPU就可以通过这个着色器程序来使用两个着色器。

#### 3. 将顶点数据存入缓冲区

现在就可以使用着色器程序来绘制我们的三角形了。

对于三角形，我们知道用三个顶点就可以确定一个三角形。

在定义三角形的顶点之前，我们需要先了解WebGL中的坐标系。和Canvas2D不同，在默认情况下，WebGL的坐标系原点`(0, 0)`在画布的中心，并且画布的左下角是`(-1, -1)`，右上角是`(1, 1)`，也就是说x和y坐标的取值范围都是-1到1。

接着我们还需要了解，WebGL中顶点信息是保存在TypedArray中的，TypedArray翻译过来可以叫做定型数组或者类型数组，我们知道在JavaScript中，普通数组中的元素并没有限制，我们可以通过push方法，插入任意类型的值，但是在定型数组中就不能这么做了，定型数组可以简单认为就是数组中所有元素的类型是被指定了的。

JavaScript通过定型数组向GPU传输数据，某种程度上也是防止GPU接收到意外类型的数据吧，并且这样GPU也不用花费额外的时间去进行数据类型的判断和转换，性能效率更高。

好了，了解完这些之后，我们就来**定义三角形的三个顶点**。

```javascript
const points = new Float32Array([
  -1, -1,
  0, 1,
  1, -1,
]);
```

points变量引用了一个Float32Array类型的数组，Float32Array就是定型数组的其中一种，代表数组中的元素都是32位的浮点数。

可以看到，我们在这个数组中放了6个元素，每两个元素代表一个点的x和y坐标。

然后我们就将这些点写入WebGL的缓冲区。我的理解是，这三个点其实并不是独立的点，WebGL会将这三个点在后续用于划定要处理的范围；后续还会通过绘图模式来进一步确定要处理的点。

定义好顶点后，我们就**将这些顶点数据写入缓冲区提供给WebGL使用**。

首先在使用前我们需要先创建buffer对象，也就是缓冲区对象。

```javascript
const bufferId = gl.createBuffer();
```

然后将这个缓冲区指定为WebGL的操作对象，`gl.ARRAY_BUFFER`代表这个缓冲区存储的是顶点数据。

```javascript
gl.bindBuffer(gl.ARRAY_BUFFER, bufferId);
```

最后将数据写入缓冲区，以供GPU读取。

```javascript
gl.bufferData(gl.ARRAY_BUFFER, points, gl.STATIC_DRAW);
```

`gl.STATIC_DRAW`代表数据加载一次，可以在多次绘制中使用。

#### 4. 将缓冲区数据读取到GPU

完成了数据的写入后，GPU就可以从缓冲区读取数据，所以我们需要**告诉GPU去哪里读取数据**。

因为上面缓冲区中存储的是顶点数据，所以这些数据是在顶点着色器中使用；又因为在顶点着色器的GLSL代码中，我们指定了变量position用于接收顶点数据，所以我们需要先**获取position变量的地址**。

```javascript
const vPosition = gl.getAttribLocation(program, 'position');
```

接着创建一个指针，指向刚刚绑定到`gl.ARRAY_BUFFER`的缓冲区，并保存到vPosition中。

```javascript
gl.vertexAttribPointer(vPosition, 2, gl.FLOAT, false, 0, 0);
```

2表示顶点数据是2个为一组读取，这是因为在顶点着色器的GLSL代码中，我们是通过vec2类型接收的；

`gl.FLOAT`代表读取的数据类型；

最后2个0，代表的是从缓冲区中读取数据时的偏移量，这个例子中数据都是连续写入的，所以不用管，都设置为0就可以。

最后是激活这个变量，这样在顶点着色器中就能通过变量position读取到points定型数组中对应的值了。

```javascript
gl.enableVertexAttribArray(vPosition);
```

#### 5. 执行着色器程序完成绘制

着色器程序和数据都准备好了之后，GPU就可以调用着色器程序并完成图形的绘制了。

首先，我们先调用`gl.clear`将当前画布的内容清除，就类似于Canvas2D中的clearReact。

```javascript
gl.clear(gl.COLOR_BUFFER_BIT);
```

我们可以直接使用`gl.COLOR_BUFFER_BIT`，也可以自己指定颜色。

然后我们就可以开始绘制了。

```javascript
gl.drawArrays(gl.TRIANGLES, 0, points.length / 2);
```

drawArrays的第一个参数是绘图模式，代表绘图时所使用的图元，图元我理解就是图形的单元，就一个图形是由若干个单元组成的，比如这个代码中的`gl.TRIANGLES`就代表这个图形由三角形组成，它就是一个三角形，那么WebGL所要渲染的点就是整个三角形区域内的点。

如果我们设置为`gl.LINE_LOOP`，就代表这个图形由封闭的线段组成，会形成一个封闭图形，它默认最后一个点和第一个点连接在一起，那么WebGL所要渲染的点就是顶点所形成的封闭线段上的点。

如果我们设置为`gl.LINES`，就代表这个图形由一个个线段组成，那么WebGL所要渲染的点就是每一条线段上的点。

drawArrays的第二个参数是缓冲区的起始偏移量，这里我们从0开始读取。

最后一个参数是顶点的个数，由于points中每两个元素一组作为一个顶点坐标，所以数组长度除以2就是顶点的个数。

至此，就完成了三角形的绘制。

可以看到，这个三角形是红色的，这是因为我们在片元着色器中的定义，就是`gl_FragColor`，我们写死了`vec4(1.0, 0.0, 0.0, 1.0);`，这就相当于我们在CSS中写的RGBA(255, 0, 0, 1)；在WebGL中，RGB色阶通道的取值也和透明度一样，在0.0到1.0之间。我们可以通过修改`gl_FragColor`来修改三角形的颜色。

由于我们在代码中把`gl_FragColor`写死了，所以在对所有点并行执行片元着色器时，渲染了同样的颜色，所以三角形整体是红色的。