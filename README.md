# evils-el-poster
uniApp的海报生成组件

# 新功能
可以通过直接获取页面元素内容,不必要自己将每个元素内容确定好定位确定好内容写成列表

## 示例

### 使用方法

 1. 将需要渲染并且有内容的元素都添加``Poster1``的样式名.
 2. 给需要渲染的元素添加``data-etype``,定义类型,不填则默认为text,
 3. 给需要渲染的元素添加``data-enode``,传入内容,此项必填
 4. 最后调用``this.$refs.Eposter.createForElRect(elClass, false)``,开始生成海报.第一个值为样式名称,第二个值为是否是``slot``内的元素.默认值为``true``,如果最后生成图片位置合适,还可以传入startX,startY,额外调整开始点的坐标.
### 示例1
```
    <view @click="getElRect1('Poster1')" class="ev-m-32 ev-fr-s-c">
      <view class="ev-fc-s-c">
        <image data-etype="image" :data-enode="info.avatar" class="ev-w-164 Poster1 ev-h-164 ev-round" :src="info.avatar" mode="aspectFit"> </image>
        <text :data-enode="info.name" class="ev-m-t-20 Poster1 ev-font-24 ev-color-6">{{ info.name }} </text>
      </view>
      <view class="ev-falign-self-s ev-m-l-20 ev-color-9">
        <text :data-enode="time" class="ev-font-24 Poster1">{{ time }} </text>
        <view data-etype="textarea" :data-enode="info.desc" class="ev-font-24 Poster1">
          {{ info.desc }}
        </view>
      </view>
    </view>
    <Eposter width="750" height="1334" :list="list" backgroundColor="rgb(255, 255, 255)" @on-success="onSuccess" @on-error="onError" ref="Eposter"> </Eposter>
```
### 示例2
 将内容写在组件内部,
```
    <button @click="getElRect('Poster')" class="ev-w-b80 ev-m-b-40">getElRect</button>
    <Eposter width="750" height="1334" @on-error="onError" :list="list" backgroundColor="rgb(255, 255, 255)" @on-success="onSuccess" ref="Eposter">
      <view class="ev-m-32 ev-fr-s-c">
        <view class="ev-fc-s-c">
          <image
            data-etype="image"
            :data-enode="info.avatar"
            class="ev-w-164 Poster ev-h-164 ev-round-32"
            :src="info.avatar"
            mode="aspectFit|aspectFill|widthFix"
          >
          </image>
          <text :data-enode="info.name" class="ev-m-t-20 Poster ev-font-24 ev-color-6">{{ info.name }} </text>
        </view>
        <view class="ev-falign-self-s ev-m-l-20 ev-color-9">
          <text :data-enode="time" class="ev-font-24 Poster">{{ time }} </text>
          <view data-etype="textarea" :data-enode="info.desc" class="ev-font-24 Poster">
            {{ info.desc }}
          </view>
        </view>
      </view>
    </Eposter>
```


## 来源
来自uniapp插件市场的 [zhangyuhao-poster](https://ext.dcloud.net.cn/plugin?id=4611#rating)项目,并做了很多的修改

## 修改

1. 新增了圆角生成功能

2. 新增了矩形的创建,对于一些基本的矩形/圆形内容可以直接代码生成

3. 将**setTimeout**延时保存图片的方式改为**draw**回调保存,减少了海报生成等待时间

4. 去掉了**mounted**生命周期的生成逻辑,改为**$refs** 调用内部的 **create()**方法.更灵活控制

5. 新增图片自适应功能

## 特别提示1

如果小程序线上不显示,可能是**uni.getImageInfo**API的问题,这个API需先配置download域名白名单才能生效,微信头像也同样需要配置域名白名单,在微信后台配置这两个域名即可

```
thirdwx.qlogo.cn
wx.qlogo.cn
```
## 特别提示2

支付宝小程序会有 ``this.callback is not a function`` 的报错,是属于``uniapp``的接口问题.

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
  mode:'center' 居中自适应
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

