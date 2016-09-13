# 给你的网页打上“码”

## 前言
很多时候我们希望自己的网页只是展示给别人看，并不希望别人截图转发。或者说即使被人转发了，我们也能追查到是谁转发的。这时候，网页上的水印效果就是不二的选择。本文旨在通过两种方式实现纯文字水印效果，图片水印实现方式较为简单，不在本文讨论范围内。

源码在这里 => [源码](https://github.com/un-defined/watermark) ，还是建议同学们自己敲着玩玩。

![](http://ww3.sinaimg.cn/mw690/a099ce0bgw1f7r6uk0i97j20ec09d75h.jpg)

## 原理
![](http://ww3.sinaimg.cn/mw690/a099ce0bgw1f7r6fd7gcej208o07wt8y.jpg)

如上图所示，我们将在要添加水印的容器（设为Layer_1）底层新建一个水印层（设为Layer_2），假设容器 z-index 为0，水印层为-1。为了使水印层至少能“包裹住”容器层，我们需要一个旋转了 45° * (2n + 1) 的正方形。假设容器的长和宽分别为A和B，根据简单的几何知识，得到水印层的边长为 L = A/2^(1/2) + B/2^(1/2)， 即 L = (A+B)*2^(1/2)/2。

![](http://ww2.sinaimg.cn/mw690/a099ce0bgw1f7r81e9rnrj208h07xq30.jpg)

为了达到这个效果，首先需要新建边长为 L = (A+B)*2^(1/2)/2 的正方形。

![](http://ww2.sinaimg.cn/mw690/a099ce0bgw1f7r81pwcuxj209u09f3yu.jpg)

然后分别向左向上平移 (L-A)/2 和 (L-B)/2 。

![](http://ww2.sinaimg.cn/mw690/a099ce0bgw1f7r81ti635j20az0acjs1.jpg)

最后以他们公共的中心旋转 45° * (2n + 1)，比如这里的逆时针旋转45°。


容器有了，接下来我们要向容器中填充水印文字，这就有两种实现方式了。概括来讲就是：

1. 向水印容器中插入水印文字
2. 用Canvas来绘制水印文字，以背景图片的形式插入到水印层

下面就两种方式讲解实现细节

### 方式一

``` bash
<style type="text/css">
    .container {
        margin-left: 300px;
        margin-top: 300px;
        width: 800px;
        height: 500px;
        border: 1px solid black;

        /*实际使用时，以下俩属性必须设置*/
        position: relative;
        overflow: hidden;
    }
    .watermark {
        position: absolute;
        line-height: 80px;
        transform: rotate(-45deg);
        -ms-transform: rotate(-45deg);          /* IE 9 */
        -moz-transform: rotate(-45deg);         /* Firefox */
        -webkit-transform: rotate(-45deg);      /* Safari 和 Chrome */
        -o-transform: rotate(-45deg);           /* Opera */
        border: 1px solid pink;
        z-index: -1;
        overflow: hidden;
        white-space: pre;
    }
</style>

;(function($){
    $.fn.setWaterMark = function(options){
        var _this = this;
        var settings = $.extend(true, {
            prefix: 'Watermark',        // 水印文字前缀
            suffixName: 'Admin',        // 后缀
            spaceCount: 4,              // 水印单元间隔空格个数
            colCount: 20,               // 每行水印单元个数
            rowCount: 20,               // 行数
            style: {},                  // 扩展样式
        }, options);

        var SQRT_2 = Math.sqrt(2),      // 常量根号二
            w = _this.width(),          // Layer_1 宽度
            h = _this.height(),         // Layer_1 高度
            L = (h + w)/SQRT_2,         // Layer_2 长度
            html = [];                  // 用于存储水印字符

        var $bgEl = $('<div>', {class: 'watermark'});       // 新建 Layer_2 对象

        var spaceStr = [],              // 保存若干连续空格
            singleMark = '';            // 水印单元
    }
    ...
})(jQuery);
```

这里我们实现一个 jQuery 插件，在 jQuery 实例上扩展 setWaterMark 方法。我们的水印单元是形如 "Watermark - Admin" 的字符串，若干个单元组成一行。之所以要在水印单元间添加若干空格，是为了实现交错的效果。

``` bash
for (var i = 0; i < settings.spaceCount; i++) {
    spaceStr.push('&nbsp;');
}

spaceStr = spaceStr.join('');
singleMark = settings.prefix + ' - ' + settings.suffixName + spaceStr;


for (var i = 0; i < Math.ceil( settings.rowCount/2 ); i++) {
    
    var singleLine = [],
        singleLineStr = '';

    for (var j = 0; j < settings.colCount; j++) {
        
        singleLine.push( singleMark );

    }

    singleLineStr = singleLine.join('');

    html.push( singleLineStr + '<br />' + settings.suffixName + spaceStr + singleLineStr + '<br />' );
}
```

以上代码根据 settings 中的配置生成了我们所需的水印字符串

``` bash
$bgEl.css( $.extend( settings.style, {
    'width': L + 'px',
    'height': L + 'px',
    'left': -(L-w)/2 + 'px',
    'top': -(L-h)/2 + 'px',
    'transform-origin': '50% 50%',          
    '-ms-transform-origin': '50% 50%',          /* IE 9 */
    '-moz-transform-origin': '50% 50%',         /* Firefox */
    '-webkit-transform-origin': '50% 50%',      /* Safari 和 Chrome */
    '-o-transform-origin': '50% 50%',           /* Opera */
}) );

$bgEl.html( html.join('') ).appendTo( _this );
```

通过设置 style 将 Layer_2 旋转平移至目标位置，最后将 Layer_2 添加到 Layer_1 中，这样我们的水印就生成了。


### 方式二

``` bash
var c = document.createElement('canvas');
var bodyEl = document.body;
var ctx;

if (!c.getContext) {
  console.log('你的破浏览器不支持canvas， 赶紧换一个！');
  return false;
}

// 动态添加不可见canvas到body上
c.width = args.cWidth || 200;
c.height = c.width;
bodyEl.appendChild(c);

ctx = c.getContext('2d');
ctx.rotate( -45*Math.PI/180 );
ctx.font = args.font || '20px Microsoft Yahei';
ctx.fillStyle = args.color || "lightgray";

var offsetX = -c.width/2;
var offsetY = c.width/(2*Math.sqrt(2));
var text = args.text || '水印文字';

ctx.fillText( text,  offsetX + 60, offsetY );
ctx.fillText( text,  offsetX - 60, 2*offsetY );
ctx.fillText( text,  offsetX + 60, 3*offsetY );
this.css('background-image', 'url("' + ctx.canvas.toDataURL() + '")');

// 将canvas移除
bodyEl.removeChild(c);
```
这种方式是利用 Canvas 创建一块画布，旋转并填充水印文字，然后将画布内容输出为 [data URI](https://developer.mozilla.org/en-US/docs/Web/API/HTMLCanvasElement/toDataURL) ，最后设置为外层容器的背景图片。因为默认背景图片是 x 轴及 y 轴平铺，所以能够自动铺满容器。

## 总结

两种方式都能实现水印效果，但是 Canvas 的实现方式兼容性有所欠缺，并且在水印文字的调节上不及方式一便捷，所以更推荐方式一。

写到后面会发现难点在于水印文字位置的调节，并没有找到一劳永逸的办法，同学们可以自由发挥。