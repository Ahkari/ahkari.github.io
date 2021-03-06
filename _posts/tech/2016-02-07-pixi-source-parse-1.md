---
layout: post
title: PIXI v3源码解析 其一
category: 技术
tags: pixiJS 源码解析 
description: pixi源码解析第一章
---

### 依赖分析
本文内容较多，直入主题。
请于github下载pixi源码，我以16.1月份v3版本的为准。

![pixi-最外层index](http://7xny7k.com1.z0.glb.clouddn.com/pixi1.png)

pixi遵循模块化开发理念，在commonJS模范下会首先按顺序加载图示模块，

然后实例化一个`loader`，

接着将`deprecation`中的那些旧版存在，而目前pixi已不再支持的特性做兼容性提示。

最后映射到全局属性`global`里。

我们先来看前后两个比较杂的模块
`./polyfill`和`./depreciation`

### 概念补习
PIXI源码中有一些方法，了解他们有助于源码阅读。

1. require()
require会从path路径索引目标js，如果目标是文件夹则默认取其中index.js的exports出的内容。
2. Object.assign(tar,orgin)
将orgin的**可枚举**属性拷贝到tar对象上，返回tar对象。 PIXI有这部分的功能增强。 
3. Object.create( orgin ) 
一般用于原型的继承, 生成一个以origin为原型的对象, 让原型式继承层级关系符合预期。

### require('./polyfill')
1. Object.assign.js 用来fix这个方法
2. requestAnimationFrame.js 用来fix这个全局渲染方法

        if (!(Date.now && Date.prototype.getTime)) { //如果Data,now()方法没有
            Date.now = function now() {
                return new Date().getTime(); //修复Data.now() 这个方法
            };
        }
        // performance.now
        if (!(global.performance && global.performance.now)) { //如果运行时间这个对象没有
            var startTime = Date.now(); //这个模块的闭包里存着检测开始的时间startTime
            if (!global.performance) {
                global.performance = {}; 
            }
            global.performance.now = function () {
                return Date.now() - startTime; // 修复这个performance方法, 让他能正常显示运行时间
            };
        }
        // requestAnimationFrame
        var lastTime = Date.now();
        var vendors = ['ms', 'moz', 'webkit', 'o']; //浏览器前缀
        for(var x = 0; x < vendors.length && !global.requestAnimationFrame; ++x) { //如果requestAnimationFrame这个方法没有
            global.requestAnimationFrame = global[vendors[x] + 'RequestAnimationFrame'];
            global.cancelAnimationFrame = global[vendors[x] + 'CancelAnimationFrame'] ||
                global[vendors[x] + 'CancelRequestAnimationFrame'];
        } //fix这个方法, 会被赋予第一个检测到的浏览器的requestAnimationFrame方法
        if (!global.requestAnimationFrame) { //如果还是没有,就得自己定义
            global.requestAnimationFrame = function (callback) {
                if (typeof callback !== 'function') {
                    throw new TypeError(callback + 'is not a function');
                }
                var currentTime = Date.now(),
                    delay = 16 + lastTime - currentTime; 
                if (delay < 0) {
                    delay = 0; //期望是[0,16]之间为执行间隔
                }
                lastTime = currentTime;
                return setTimeout(function () {
                    lastTime = Date.now();
                    callback(performance.now());
                }, delay); //按照delay来执行回调
            };
        }
        if (!global.cancelAnimationFrame) {
            global.cancelAnimationFrame = function(id) {
                clearTimeout(id);
            };
        }
3. Math.sign.js 用来fix这个判断正负的函数
    
        if (!Math.sign)
        {
            Math.sign = function (x) {
                x = +x;
                if (x === 0 || isNaN(x))
                {
                    return x;
                }
                return x > 0 ? 1 : -1;
            };
        }

### Object.assign(core, require('./deprecation'));
之前说过了, 这个`deprecation`是一些不推荐使用的内容了。

是用来兼容旧版的, 如果你在新版pixi里使用旧版有而新版没有的对象就会报错。

这个文件里会将每个可能出错的地方整理并给出合理的提示。这是一个成熟的框架所必须的素养。

我们整理下这个文件里的提示。来认识下pixi v3所做的改动。

1. core.SpriteBatch 这个对象已不存在，不要尝试new它，请用new ParticleContainer来替代。
2. core.AssetLoader 这个对象在v3中被分拆，请看new PIXI.loaders.Loader类。
3. core.Stege 这个舞台已被container完全取代, 已没有存在的必要了。所以直接指向core.Container
4. core.DisplayObjectContainer 直接缩短名字，就叫core.Container了。
5. core.Strip 放入mesh下，更改为mesh.Mesh。
6. core.Rope 同样，叫mesh.Rope。
7. core.MovieClip 改为extras.MovieClip。
8. core.TilingSprite 改为extras.TilingSprite。
9. core.BitmapText 改为extras.BitmapText。
10. core.blendModes 本命名空间下更名为core.BLEND_MODES。
11. core.scaleModes 本空间下更名为core.SCALE_MODES。
12. core.BaseTextureCache 移进同级的utils下, 变成core.utils.BaseTextureCache。
13. core.TextureCache 移进utils下变成core.utils.TextureCache。
14. core.Math 这个命名空间移除，相关方法直接进core里找。
15. core.Sprite.prototype.setTexture 这个原型方法不赞成使用，建议直接sprite.texture = texture。
16. extras.BitmapText.prototype.setText 这个方法也是，建议直接写myBitmapText.text = 'my text'。
17. core.Text.prototype.setText 同上，myText.text = 'my text'; 。
18. core.Text.prototype.setStyle 同上，myText.style = style; 。
19. core.Texture.prototype.setFrame 同上，myTexture.frame = frame。
20. filters.AbstractFilter 这个不够正式，请用core.AbstractFilter。
21. filters.FXAAFilter 同上，请用core.FXAAFilter。
22. filters.SpriteMaskFilter 同上，请用core.SpriteMaskFilter。
23. core.utils.uuid 这个方法更名为core.utils.uid();

### var core = module.exports = require('./core');
我们进入core核心部分。其下index.js里面有他所需的所有子模块的内容。

    var core = module.exports = Object.assign(require('./const'), require('./math'), {
        // utils工具类
        utils: require('./utils'),
        ticker: require('./ticker'),
        // display基础展示对象
        DisplayObject:          require('./display/DisplayObject'),
        Container:              require('./display/Container'),
        // sprites高级点的精灵对象
        Sprite:                 require('./sprites/Sprite'),
        ParticleContainer:      require('./particles/ParticleContainer'),
        SpriteRenderer:         require('./sprites/webgl/SpriteRenderer'),
        ParticleRenderer:       require('./particles/webgl/ParticleRenderer'),
        // text文本类
        Text:                   require('./text/Text'),
        // primitives形状元素
        Graphics:               require('./graphics/Graphics'),
        GraphicsData:           require('./graphics/GraphicsData'),
        GraphicsRenderer:       require('./graphics/webgl/GraphicsRenderer'),
        // textures材质类。是非形状元素的展示基础。
        Texture:                require('./textures/Texture'),
        BaseTexture:            require('./textures/BaseTexture'),
        RenderTexture:          require('./textures/RenderTexture'),
        VideoBaseTexture:       require('./textures/VideoBaseTexture'),
        TextureUvs:             require('./textures/TextureUvs'),
        // renderers - canvas渲染器
        CanvasRenderer:         require('./renderers/canvas/CanvasRenderer'),
        CanvasGraphics:         require('./renderers/canvas/utils/CanvasGraphics'),
        CanvasBuffer:           require('./renderers/canvas/utils/CanvasBuffer'),
        // renderers - webgl渲染器
        WebGLRenderer:          require('./renderers/webgl/WebGLRenderer'),
        ShaderManager:          require('./renderers/webgl/managers/ShaderManager'),
        Shader:                 require('./renderers/webgl/shaders/Shader'),
        ObjectRenderer:         require('./renderers/webgl/utils/ObjectRenderer'),
        RenderTarget:           require('./renderers/webgl/utils/RenderTarget'),
        // filters - webgl滤镜
        AbstractFilter:         require('./renderers/webgl/filters/AbstractFilter'),
        FXAAFilter:             require('./renderers/webgl/filters/FXAAFilter'),
        SpriteMaskFilter:       require('./renderers/webgl/filters/SpriteMaskFilter'),
        /**
         * This helper function will automatically detect which renderer you should be using.
         * WebGL is the preferred renderer as it is a lot faster. If webGL is not supported by
         * the browser then this function will return a canvas renderer
         * 这个方法用于webGl和canvas渲染之间的切换，如果主动关闭webGL或检测到浏览器不支持webGL这个方法就会切换到canvas渲染。
         */
        autoDetectRenderer: function (width, height, options, noWebGL)
        {
            width = width || 800;
            height = height || 600;
            if (!noWebGL && core.utils.isWebGLSupported())
            {
                return new core.WebGLRenderer(width, height, options);
            }
            return new core.CanvasRenderer(width, height, options);
        }
    });

在webGL和canvas间智能切换是pixi的特色之一。基于其之上的游戏引擎phaser也拿这个作为特色。

我们来一个个看核心库的内容。

### require('./const')
写着所有的静态变量。
需要关注的几个是：

* `TARGET_FPMS:0.06`期望是0.06秒一帧
* `RENDERER_TYPE:{ UNKNOWN:0 , WEBGL: 1 , CANVAS: 2}`设定渲染模式映射
* `BLEND_MODES:{...}`混合模式映射
* `DRAW_MODES:{...}`webGL绘制模式
* `SCALE_MODES:{...}`拉伸模式
* `RETINA_PREFIX: /@(.+)x/`声明这是种高分辨率的素材时要加的前缀
* `RESOLUTION:1`分辨率
* `FILTER_RESOLUTION:1`滤镜分辨率
* `DEFAULT_RENDER_OPTIONS:{...}`默认渲染配置（重要）
* `SHAPES:{...}`支持的形状的映射

### require('./math')
数学相关的计算。

目标路径是math文件夹，其中包含了各种形状之间的运算。虽然都是二维图形运算，但全部看明白也需要一点的初中数学功底。
    
1. PIXI.Point
用于表示一个二维坐标点的类。含有X和Y两个属性与四个方法，它是二维运算的一个基础类：
        
    * PIXI.Point.clone() 返回一个和当前点XY值一样的新二维坐标
    * PIXI.Point.copy(point) 将参数坐标点赋给当前点
    * PIXI.Point.quals(point) 比较两个坐标是否一模一样
    * PIXI.Point.set(x,y) 
        
2. PIXI.Matrix
二位矩阵的类，含有六个属性[a,b,c,d,e,f]。可以实现css3那样的所有变形效果。补充知识如下:

![二维矩阵运算](http://7xny7k.com1.z0.glb.clouddn.com/pixi1.jpg)

浏览器通过矩阵运算的方式运算matrix来实现css3，pixi自然也是一样。

我们根据上图总结出如下规律:

1, 4 是缩放变形的结果　　scale(sx,sy)可以由matrix(sx,0,0,sy,0,0)转变

5, 6 是平移变形的结果　　translate(tx,ty)可以由matrix(1,0,0,1,tx,ty)转换而来，

2, 3 是扭曲变形的结果　　skew(θx，θy)可以由matrix(1,tan(θy),tan(θx),1,0,0)转变过来

1, 2, 3, 4 是旋转变形的结果　　rotate(θ)可以有matrix(cosθ,sinθ,-sinθ,cosθ,0,0)转变而来

其原型方法：
    
    * PIXI.Matrix.formArray(array) 应用此数组作为矩阵新的属性
    * PIXI.Matirx.toArray() //返回九个值的矩阵数组
    * PIXI.Matirx.apply(pos,newPos) pos被应用了当前矩阵后会处于什么位置
    * PIXI.Matric.applyInverse(pos,newPos) 给应用了反向矩阵。
    * PIXI.Matric.translate(x,y) 改变tx，ty的值，用于位移变化。
    * PIXI.Matric.scale(x,y) 用于拉伸变换。
    * PIXI.Matric.rotate(angle) 用于旋转角度变换。
    * PIXI.Matric.append(matrix) 
    * PIXI.Matric.perpend(matrix)
    * PIXI.Matric.invert() 矩阵反转
    * PIXI.Matric.indentity() 矩阵重置
    * PIXI.Matric.clone() 矩阵克隆
    * PIXI.Matric.copy()
3. PIXI.Rectangle
矩形类。含有位置x，y，宽高width，height，以及类型type(根据常量设定，矩形为1）
    * PIXI.Rectangle.clone()
    * PIXI.Rectangle.contains(x,y) 判断目标x,y是否在当前矩阵中
4. PIXI.Circle 
圆形类。属性有位置x，y，半径radius，type为0
    * PIXI.Circle.clone()
    * PIXI.Circle.contains(x,y) 判断是否在圆之内
            
        Circle.prototype.contains = function (x, y)
        {
            if (this.radius <= 0)
            {
                return false;
            }
            var dx = (this.x - x),
                dy = (this.y - y),
                r2 = this.radius * this.radius;
            dx *= dx;
            dy *= dy;
            return (dx + dy <= r2);
        };

    * PIXI.Circle.getBounds() 返回刚好能把这个圆圈住的矩形范围   
5. PIXI.Ellipse
椭圆类。属性有位置x,y，宽高width，height，类型3

方法同圆形，究竟是后续要有椭圆的额外操作还是PIXI当前不支持椭圆呢，这个疑问我们阅读到后面才能知道。

2月12号回顾：这个问题已经解决，PIXI用了画四段贝赛尔曲线的方式来绘制椭圆。
6. PIXI.Polygon
多边形类。期望传入的参数是一堆表示每个顶点位置的数组。
    * PIXI.Polygon.clone() 
    * PIXI.Polygon.contains(x,y) 判断一个点是否在多边形内，运用了边内外遍历叠加的算法。

        Polygon.prototype.contains = function (x, y)
        {
            var inside = false;
            var length = this.points.length / 2;
            for (var i = 0, j = length - 1; i < length; j = i++)
            {
                var xi = this.points[i * 2], yi = this.points[i * 2 + 1],
                    xj = this.points[j * 2], yj = this.points[j * 2 + 1],
                    intersect = ((yi > y) !== (yj > y)) && (x < (xj - xi) * (y - yi) / (yj - yi) + xi);
                if (intersect)
                {
                    inside = !inside;
                }
            }
            return inside;
        };
7. PIXI.RoundedRectangle
圆角矩形类。属性:x,y,width,height,radius（圆角）。他除了多了一个圆角属性，和普通矩形完全一样，就连contains方法都是，究竟之后会不会有特殊处理，我们之后才能知道。

### utils : require('./utils')
一些小的工具方法，他require了node的事件注册模块'eventemitter3',异步模块'async'和自己的pluginTarget模块。

* PIXI.utils.EventEmitter 采用node的事件模式，是一种观察者模式。
* PIXI.utils.async node的异步模块。解决异步代码嵌套过深的问题。
* PIXI.utils.hex2rgb 16进制转为rgb色彩模式
* PIXI.utils.hex2string 16进制转为#XXXXXX颜色表示
* PIXI.utils.rgb2hex 反之
* PIXI.utils.canUseNewCanvasBlendModes 重要，用于判断当前canvas是否支持新的混合模式。
        
        canUseNewCanvasBlendModes: function ()
        {
            if (typeof document === 'undefined')
            {
                return false; //非浏览器环境直接false
            }
            var pngHead = 'data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAQAAAABAQMAAADD8p2OAAAAA1BMVEX/';
            var pngEnd = 'AAAACklEQVQI12NgAAAAAgAB4iG8MwAAAABJRU5ErkJggg==';
            var magenta = new Image();
            magenta.src = pngHead + 'AP804Oa6' + pngEnd;
            var yellow = new Image();
            yellow.src = pngHead + '/wCKxvRF' + pngEnd;
            var canvas = document.createElement('canvas');
            canvas.width = 6;
            canvas.height = 1;
            var context = canvas.getContext('2d');
            context.globalCompositeOperation = 'multiply'; //设置混合模式
            context.drawImage(magenta, 0, 0);
            context.drawImage(yellow, 2, 0);
            var data = context.getImageData(2,0,1,1).data;
            return (data[0] === 255 && data[1] === 0 && data[2] === 0); //检测颜色是否符合预期
        },
* PIXI.utils.getNextPowerOfTwo
* PIXI.utils.isPowerOfTwo
* PIXI.utils.getResolutionOfUrl
* PIXI.utils.sayHello 打个招呼~
* PIXI.utils.isWebGLSupported 重要，是否支持webGL
    
        isWebGLSupported: function ()
        {
            var contextOptions = { stencil: true };
            try
            {
                if (!window.WebGLRenderingContext) //不存在直接return
                {
                    return false;
                }
                var canvas = document.createElement('canvas'), 
                    gl = canvas.getContext('webgl', contextOptions) || canvas.getContext('experimental-webgl', contextOptions);
                return !!(gl && gl.getContextAttributes().stencil); //还要确实检测是否支持webGL模式
            }
            catch (e)
            {
                return false;
            }
        },

* PIXI.utils.TextureCache = {}  //缓存
* PIXI.utils.BaseTextureCache = {}  //
* PIXI.utils.pluginTarget.mixin(obj) 
    这是PIXI的插件拓展方式。传入一个对象。这个对象便赋予了`registerPlugin`注册插件`initPlugins`初始化插件`destroyPlugins`销毁插件的方法。具体插件实现由注册时传入的构造函数来定，这里PIXI只是提供了一个供发挥的接口。


### tickers : require('./ticker') 
时钟类。用于控制渲染帧率等。