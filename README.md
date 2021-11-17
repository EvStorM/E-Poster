# E-Poster
uniApp的海报生成组件
## 来源
来自uniapp插件市场的 [zhangyuhao-poster](https://ext.dcloud.net.cn/plugin?id=4611#rating)项目

## 修改

1. 新增了圆角生成功能

2. 新增了矩形的创建,对于一些基本的矩形/圆形内容可以直接代码生成

3. 将**setTimeout**延时保存图片的方式改为**draw**回调保存,减少了海报生成等待时间

4. 去掉了**mounted**生命周期的生成逻辑,改为**$refs**调用内部的**create()**方法.更灵活控制

   

## 特别提示

如果小程序线上不显示,可能是**uni.getImageInfo**API的问题,这个API需先配置download域名白名单才能生效,微信头像也同样需要配置域名白名单,在微信后台配置这两个域名即可

```
thirdwx.qlogo.cn
wx.qlogo.cn
```



## 示例

```vue
// 使用    
<poster
      :list="posterlist"
      :width="posterWidth"
      backgroundColor="#FFF"
      :height="posterHeight"
      @on-error="posterError"
      @on-success="posterSuccess"
      ref="poster"
    />
```

```javascript
        let list = [];
        list.push({
          type: 'text',
          x: 55,
          y: 72,
          text: this.shareInfo.nickname || '',
          width: 480,
          height:  90,
          size: 32,
          color: '#333'
        });
        list.push({
          type: 'textarea',
          x: 55,
          y: 72,
          text: this.shareInfo.title || '',
          width: 480,
          height:  90,
          size: 32,
          color: '#333'
        });
        list.push({
          type: 'image',
          path: 'share/shareInfo.png',
          x: 0,
          y: 736,
          width: 600,
          height: 377.25,
          shape:12
        });
          list.push({
            type: 'square',
            x: 445,
            y: 394 ,
            width: 90,
            height: 40 ,
            shape: 6,
            fillStyle: 'rgba(0, 0, 0, 0.5)'
          });
        this.posterlist = list;
// 延迟0毫秒调用,主要是滞后调用,等待数组创建完成       
setTimeout(() => {
          this.$refs.poster.create();
        }, 0);
```

## 说明

```
list参数说明：
 图片渲染：
 type: 'image',
 x: X轴位置,
 y: Y轴位置,
 path: 图片路径,
 width: 图片宽度,
 height: 图片高度,
 rotate: 旋转角度
 shape: 形状，默认无，可选值：circle 圆形 如果是number,则生成圆角矩形
 area: {x,y,width,height}  // 绘制范围，超出该范围会被剪裁掉 该属性与shape暂时无法同时使用，area存在时，shape失效
矩形渲染：
 type: 'square',
 x: X轴位置,
 y: Y轴位置,
 width: 图片宽度,
 height: 图片高度,
 rotate: 旋转角度
 shape: 形状，默认无，可选值：circle 圆形 如果是number,则生成圆角矩形
 fillStyle:填充的背景样式
 area: {x,y,width,height}  // 绘制范围，超出该范围会被剪裁掉 该属性与shape暂时无法同时使用，area存在时，shape失效
 文字渲染：
 type: 'text',
 x: X轴位置,
 y: Y轴位置,
 text: 文本内容,
 size: 字体大小,
 textBaseline： 基线 默认top  可选值：'top'、'bottom'、'middle'、'normal'
 color: 颜色
 多行文字渲染：
 type: 'textarea',
 x: X轴位置,
 y: Y轴位置,
 width：换行的宽度
 height: 高度，溢出会展示“...”
 lineSpace: 行间距
 text: 文本内容,
 size: 字体大小,
 textBaseline： 基线 默认top  可选值：'top'、'bottom'、'middle'、'normal'
 color: 颜色
```

