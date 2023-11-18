微前端，前端这次词就不用多做解释了，这个概念的重点在于这个“微”字， 从字面意义上看，微是小的意思，小是相对于大的一个用于比较的形容词，所以通常是在项目庞大的情况下，才会考虑将它变小，去考虑将它拆分成若干个小项目。这就是做微前端所要达到的主要目标，将庞大的项目拆分成多个独立运行、独立部署和独立开发的小项目，使得项目利于维护和更新，然后在运行时，作为一个整体来呈现。

项目过于庞大的可能存在一系列问题，比如构建速度慢、应用加载慢、定位问题麻烦、项目可维护性差等等。

过往的案例中，通常会使用iframe作为微前端的一种解决方案，但iframe有一些明显的缺点，比如浏览器的前进后退，由于iframe的url不会显示在浏览器的地址栏，就会使前进后退看上去有点奇怪，因为如果iframe存在历史记录，就会先对iframe的历史记录进行前进后退操作，但在地址栏上不会体现出来，我们也不能通过地址栏在主应用中直接打开子应用中某个页面，并且刷新也存在问题（可以写一点代码看一下。。。）；还有iframe是与外部窗口隔离开的，如果有弹窗需要遮罩层，就会使样式很奇怪，因为遮罩层只会遮住iframe的部分；然后iframe与外部窗口通信也不是很方便。当然iframe还是有优点的，比如资源隔离，样式之间不会互相影响，可以加载外部页面的内容，这在早期也算比较好用的一种方案。

虽然以前没有微前端的概念，但其实很多平台类的应用都存在对微前端的实践，他们会在主应用中提供入口，让用户可以进入其他应用去使用其他功能，而不是把所有的功能、所有的东西都放在一个应用里面。

single-spa是近几年出来的微前端框架，带火了微前端的概念，我还没在项目中正式使用过，现在刚开始接触学习。它主要参考了近几年流行框架中路由的概念，来推出的一种方案。借助single-spa脚手架，我们可以搭建三类项目，一个是包含root config的项目，可以说它是将所有项目包裹起来的主项目，可以将它看作是一个应用容器，在root config项目中包含一个名为xxx-root-config的js文件和一个index.ejs文件，这个ejs文件就是所有子应用所共用的html页面，root config项目通常配合使用single-spa-layout来帮助更方便地对应用布局、注册应用和定义路由；第二类是普通的应用或者parcel组件，也就是普通的子应用，可以基于任意框架开发，parcel通常用于编写可以跨框架使用的组件，parcel和application的定义比较相似，只是使用上有所不同，application可以通过路由激活自动挂载，而parcel需要通过mountParcel或者mountRootParcel方法来手动挂载；第三类是utility module，工具包，用于提供一些通用功能，比如样式、api等，通常没有组件需要渲染。

相关的概念，比如：Root Config、Application、Parcel、Layout Engine，都可以在single-spa的官网上查阅。

知道了这些，就可以开始动手搭建简单的demo项目了。

我们先来创建一个root config类型的项目，暂时先不使用single-spa-layout，看看不使用的情况下，页面是长什么样子的。

>directory，就是项目创建在哪个目录下，默认是点，就是当前目录，为了方便管理所有微前端应用，我把这个项目创建在platform目录，这个目录它会自动创建，select type，应用的类型，选择single-spa root config，接下来是包管理器，选择yarn，当然选其他的也可以，是否使用ts，因为是简单的demo，就不用了，是否使用single-spa-layout，暂时选择不用，最后是organization name，就是组织名称，我用我的英文名becky。

（等待项目初始化。。。。。。。）

我们看一下项目的内容，整体结构的话和普通脚手架生成的项目并没有太大的不同，我们可以先关注src目录下的两个文件，index.ejs和becky-root-config.js。index.ejs就是之前说的子应用所共用的html页面，becky-root-config.js就是前面所说的root config文件，它的名字是由organization name和root config组合，使用横杆拼接起来的。

我们可以先运行一下这个项目，打开localhost:9000。页面上有一串welcome之类的文字，就是一个欢迎页面。但是我们刚刚看了，项目里就两个文件，那这些内容是哪里来的呢？我们可以再仔细看下两个文件的内容。

首先是index.ejs。

可以看到，这里使用了一些在普通项目里很少有看到使用的东西，比如类型为systemjs-importmap的script标签，它里面的内容是json，然后在其他script标签中，还使用了System.import方法。这涉及到两块内容：importmap和systemjs。

importmap直译过来是导入映射，与模块的使用有关，一般我们在项目中使用模块，会调用require方法，或者使用import语句或方法，引入的模块通常需要使用npm之类的包管理器进行管理。但是import map提供了一种支持，让我们可以直接在页面上管理模块，不需要通过打包构建。不过由于这个特性比较新，很多浏览器不支持，我这里写一个小的示例。

（编写一个页面。。。）

运行一下，可以看到，我们完全不需要构建就可以使用import语句导入模块。

Import maps 本质上是一个配置文件，可以让开发者将模块标识符映射到一到多个文件，描述了依赖的解析方式，某种程度上，Import maps 给浏览器端带来了包管理，但是目前支持 Import Maps 的浏览器还很少。

简单来说importmap的作用就是使浏览器端支持模块的解析，而不需要应用构建步骤，这使得前端开发更便捷了，但是import maps现在来使用的话存在一个缺点，就是需要所有模块都导出成 ESModule，当前社区当中的很多模块都没有导出成 ESModule，有些模块甚至没有经过编译，所以目前使用仍然有一定困难。

systemjs可以说是import maps的一种兼容方案，同样有模块管理的功能，在浏览器端实现了对 CommonJS、AMD、UMD 等各种模块的加载；它提供了一套自己的加载方法，我们可以对刚才的demo做一些改动，来看看效果。

（创建页面，复制内容。。。）

>首先将script的type改为systemjs-importmap，然后我们改为使用umd的模块，并在页面上使用script标签引入systemjs，在浏览器中引入system.js后，会去解析类型为systemjs-importmap的script标签里的import映射。最后对这个类型为module的script标签中的内容进行修改，将import语句都改为使用System.import方法。因为调用System.import得到的是一个promise，我们直接使用await来获取最终的内容。
>log一下变量，可以看到调用System.import方法直接获取的是整个模块，我们可以在页面上引入systemjs的use-default脚本，将模块的default部分提取出来，使用起来更方便。

再运行一下，可以看到，效果是一样的。

关于importmap和systemjs就先到这里不细讲了，因为具体的我也还没仔细看。

回到index.ejs，我们看到，使用System.import导入了一个`@becky/root-config`，再看上面的importmap，我们可以看到，这个标识符对应的就是becky-root-config.js文件。也就是说页面上引用了root config打包构建后的文件。

index.ejs文件的最后是一个import-map-overrides-full的标签，它是single-spa提供的一个开发工具，只要将localstorage中的devtools设置为true，就可以看到页面的右下角有一个图标。

（我们来设置一下。。。）

点击可以打开一个面板，上面展示了浏览器管理的两个模块，就是我们在importmap中定义的，通过这个工具，我们可以对应用中定义的importmap映射地址进行修改替换，方便本地调试。当然我们也可以通过安装浏览器插件的方式来进行调试，对于chrome可以安装一个single-spa-inspector的插件。

index.ejs看完了，再来看下root-config文件，这个文件很关键，我们通过这个文件来注册应用。可以看到顶部导入了两个方法，一个是registerApplication，见名知意，就是注册应用的方法，一个是start，就是启动项目的方法。

>可以看到，这里注册了一个子应用的示例，参数是一个对象，包含了三个属性，name、app和activeWhen，name就是名称，`@single-spa/welcome `，app就是对应的应用，可以看到是导入了一个single-spa官方的用作示例的欢迎页，最后是activeWhen，就是这个应用在什么条件下激活，默认是“/”，所以根路由被激活时就会显示，所以我们刚刚访问localhost:9000页面看到的内容就是它。
>activeWhen可以是一个字符串、或者函数，也可以是一个数组，包含函数或字符串类型的元素，本质上是一个函数，如果是字符串或者数组中的字符串元素，实际会被处理成用location进行判断，判断location.pathname是否是这个字符串开头的，如果是，路由就被激活，如果activeWhen是一个函数或者数组中包含的函数元素，则这个函数有一个默认传参是location，根据函数返回值判断路由对应的应用是否被激活。

可以看到single-spa是通过js文件接入子应用。

检查元素看一下，可以看到页面上有一个div元素，id为single-spa-application:后边再跟一个应用名@single-spa/welcome，div里边就是欢迎页的内容。

现在我们在root-config文件中注册一个简单的应用，比如写一个app1，使用registerApplication注册。因为是项目中的文件，我们直接调用require，或者()=>import，activeWhen我们就用“/app1”。运行一下，访问路由/app1，可以看到app1被挂载了，我们写的内容也显示在页面上了。

回到app1.js的内容，可以看到导出了几个方法，bootstrap、mount和unmount，通过single-spa官网对应用的定义，我们了解到，导出应用必须要定义这三个生命周期方法，来对应用的启动、挂载和卸载的过程做一些处理。当然如果需要的话，我们还可以定义unload生命周期方法。这些生命周期方法可以接收到一些属性，我们可以通过log来打印查看。

当然实际项目肯定没这么简单，我们再来创建几个项目，来作为子应用。同样使用single-spa的脚手架来创建。

>一样的，需要选择directory，就是项目创建在什么目录下，默认是点，我把这个项目创建在navbar目录，作为整个应用的导航栏，select type，应用的类型，选择application or parcel，使用的ui框架，选择react，接下来是包管理器，选择yarn，是否使用ts，简单的demo就不用了，暂时选择不用，organization name，还是使用我的英文名becky，最后是project name，在root config项目创建的时候脚手架给设置了默认项目名root config，普通的子应用我们可以自己设置项目名，还是设置navbar。

同样的步骤我们再创建几个项目。

在子项目中执行yarn:start命令，并不能直接访问目标页面，通过访问localhost:8081，我们可以看到一个页面，上面的文字提示我们需要在一个root config应用下来预览这个子应用页面，或者执行`yarn start:standalone`命令来预览，我们刚刚已经跑起来一个root config项目了，可以尝试直接把这个子应用注册到root config中。

先复制页面上提示我们复制的URL地址。

这次注册我们直接使用System.import，因为这个地址包含的内容是打包后的模块。

此时刷新页面后，会发现页面是空白的，控制台报了一个错误，提示无法解析‘react’这个标识符，这是因为我们的子应用使用了react框架，但是子应用使用webpack构建时是使用externals把react设置为外部模块，所以无法解析，此时我们可以选择在root config项目下的index.ejs文件中，使用importmap来引入react。

再次刷新页面，我们就可以看到页面有`@becky/navbar is mounted!`的字样，代表子应用挂载成功了。

为了管理方便，我们把navbar的模块也放到importmap中。

刚刚我们创建的子应用可以都这样直接调用registerApplication来注册，但是这样一个个注册不太方便，页面布局也不太直观，而且刚刚直接访问根路径时，我们看到navbar是在顶部，但是访问app1的时候，navbar又在app1的下面，为了方便布局和注册应用，我们可以使用single-spa-layout这个库。通过使用这个库，我们还可以定义应用加载时的过渡效果。

（先在root config项目中安装一下依赖。。。。）

然后对我们的index.ejs做一些修改。我们可以把官网的这段内容复制过来，然后做一点修改，把刚刚创建的几个子应用运行一下，并且把对应模块放到importmap中。

然后修改我们的root config文件。同样的把官网中的内容复制过来。

刷新页面，通过访问我们使用route标签定义的路由，可以激活不同的子应用。检查元素时，可以看到子应用挂载的布局与我们定义的single-spa-router中的内容是保持一致的。

此时一个简单的demo就完成了。

接下来我们可以修改一下navbar，就不用手动去改地址栏了。

打开navbar项目，直接查看src下面的文件，这个test文件是用于测试的，可以先不管，becky-navbar.js就是定义应用的主文件，可以看到使用了single-spa-react这个库，来创建single-spa应用，要修改navbar的展示内容，我们主要看rootComponent这个参数，它指定了应用的根组件，也就是说子应用的内容都是在这个组件里面，我们看到根组件就是这个root.component.js。

打开这个文件，我们做一点修改。首先我们看这个根组件，它接收了一个props参数，里面包含了跟应用相关的一些信息，比如props.name，就是应用名称，刚刚显示在页面上了。

完成修改后就可以看到效果了，点击不同的链接可以跳转不同的路由。

我们可以继续尝试在子应用中配置路由。比如students项目，我们给他添加路由功能。

>首先添加react-router-dom依赖，再对root.component进行改造。再添加几个react组件，并配置一张路由表。由于子应用的路由是基于主应用的路由，所以给BrowserRouter配置一个basename属性，/students。

改造完成后，重启students项目。再继续看，可以正常运行。

简单的demo就到此为止了。

最后我们再看一下webpack的配置。文件很简单，因为single-spa这个框架做了一层封装，我们可以把配置打印出来看一下。

（增加log代码，重启项目。。。）

主要看一下output和externals，externals配置的是引用外部的模块，可以看到single-spa、react、react-dom这些三方库都是引用的外部模块，也就是利用了我们在importmap中配置的模块映射，还有@becky开头的模块标识符也是引用了外部的模块，这些也是在importmap中配置了。

然后是output，看到输出的文件是becky-students.js，也就是importmap映射的子应用的文件。libraryTarget是system，他指定了使用system的模式处理模块，我们可以从network再看一下页面加载的becky-student.js的内容，可以看到文件的开头就调用了System.register注册了模块，这又涉及到了systemjs的内容。

视频的内容差不多就到这里了。

简单说了下微前端、以及single-spa的基础使用。有兴趣同学的可以继续查阅single-spa的官网文档、以及其他优秀的文章。

最后再说些微前端的缺点，比如子应用如果维护在不同的代码库里，这可能会造成代码分散，不利于整体的管理，对管理者要求更高，如果有新需求迭代，需要盘点确认涉及哪些子应用，还可能出现三方库版本不同所造成的维护问题。使用single-spa由于没有沙箱环境，可能会出现样式互相干扰的情况，需要去处理。由于我还没有具体的实践经验，那在实际使用中还可能出现一些问题，比如如何去拆分一个大型项目。

总之是否使用微前端，通过何种方式、何种框架来实践微前端，可以借鉴他人经验，根据实际情况来整体考量。