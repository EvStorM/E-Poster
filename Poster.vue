<template>
  <view class="canvas">
    <view ref="Content" class="theContent">
      <slot></slot>
    </view>
    <canvas canvas-id="myCanvas" id="myCanvas" @onReady="onCanvasReady" :style="{ width: width + 'px', height: height + 'px' }"></canvas>
  </view>
</template>
<!-- 
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
 -->
<script>
let timer = null;
export default {
  name: "Poster",
  props: {
    // 绘制队列
    list: {
      type: Array,
      required: true,
    },
    // 海报宽度(默认设备宽度放大两倍) 建议都放大两倍
    width: {
      type: [Number, String],
      default: uni.getSystemInfoSync().windowWidth * 2,
    },
    // 海报高度(默认设备高度放大两倍)
    height: {
      type: [Number, String],
      default: uni.getSystemInfoSync().windowHeight * 2,
    },
    //背景颜色
    backgroundColor: {
      type: String,
      default: "rgba(0,0,0,0)",
    },
  },
  emit: ["on-success", "on-error"],
  data() {
    return {
      posterUrl: "",
      renderList: [],
      ctx: null, //画布上下文
      counter: -1, //计数器
      drawPathQueue: [], //画图路径队列
    };
  },
  watch: {
    drawPathQueue(newVal, oldVal) {
      // 绘制单行文字
      const fillText = (textOptions) => {
        this.ctx.setFillStyle(textOptions.color);
        this.ctx.setFontSize(textOptions.size);
        this.ctx.setTextBaseline(textOptions.textBaseline || "top");
        this.ctx.fillText(textOptions.text, textOptions.x, textOptions.y);
      };
      // 绘制段落
      const fillParagraph = (textOptions) => {
        this.ctx.setFontSize(textOptions.size);
        let tempOptions = JSON.parse(JSON.stringify(textOptions));
        // 如果没有指定行间距则设置默认值
        tempOptions.lineSpace = tempOptions.lineSpace ? tempOptions.lineSpace : 10;
        // 获取字符串
        let str = textOptions.text;
        // 计算指定高度可以输出的最大行数
        let lineCount = Math.floor((tempOptions.height + tempOptions.lineSpace) / (tempOptions.size + tempOptions.lineSpace));
        // 初始化单行宽度
        let lineWidth = 0;
        let lastSubStrIndex = 0; //每次开始截取的字符串的索引

        // 构建一个打印数组
        let strArr = str.split("");
        let drawArr = [];
        let text = "";
        while (strArr.length) {
          let word = strArr.shift();
          text += word;
          let textWidth = this.ctx.measureText(text).width;
          if (textWidth > textOptions.width) {
            // 因为超出宽度 所以要截取掉最后一个字符
            text = text.substr(0, text.length - 1);
            drawArr.push(text);
            text = "";
            // 最后一个字还给strArr
            strArr.unshift(word);
          } else if (!strArr.length) {
            drawArr.push(text);
          }
        }

        if (drawArr.length > lineCount) {
          // 超出最大行数
          drawArr.length = lineCount;
          let pointWidth = this.ctx.measureText("...").width;
          let wordWidth = 0;
          let wordArr = drawArr[drawArr.length - 1].split("");
          let words = "";
          while (pointWidth > wordWidth) {
            words += wordArr.pop();
            wordWidth = this.ctx.measureText(words).width;
          }
          drawArr[drawArr.length - 1] = wordArr.join("") + "...";
        }
        // 打印
        for (let i = 0; i < drawArr.length; i++) {
          let _h = i > 0 ? tempOptions.size + tempOptions.lineSpace : 0;
          tempOptions.y = tempOptions.y + _h; // y的位置
          tempOptions.text = drawArr[i]; // 绘制的文本
          fillText(tempOptions);
        }
      };
      const roundRect = (x, y, w, h, r) => {
        // 开始绘制
        this.ctx.beginPath();
        // 因为边缘描边存在锯齿，最好指定使用 transparent 填充
        // 这里是使用 fill 还是 stroke都可以，二选一即可
        this.ctx.setFillStyle("transparent");
        // ctx.setStrokeStyle('transparent')
        // 左上角
        this.ctx.arc(x + r, y + r, r, Math.PI, Math.PI * 1.5);
        // border-top
        this.ctx.moveTo(x + r, y);
        this.ctx.lineTo(x + w - r, y);
        this.ctx.lineTo(x + w, y + r);
        // 右上角
        this.ctx.arc(x + w - r, y + r, r, Math.PI * 1.5, Math.PI * 2);

        // border-right
        this.ctx.lineTo(x + w, y + h - r);
        this.ctx.lineTo(x + w - r, y + h);
        // 右下角
        this.ctx.arc(x + w - r, y + h - r, r, 0, Math.PI * 0.5);

        // border-bottom
        this.ctx.lineTo(x + r, y + h);
        this.ctx.lineTo(x, y + h - r);
        // 左下角
        this.ctx.arc(x + r, y + h - r, r, Math.PI * 0.5, Math.PI);
        // border-left
        this.ctx.lineTo(x, y + r);
        this.ctx.lineTo(x + r, y);
        // 这里是使用 fill 还是 stroke都可以，二选一即可，但是需要与上面对应
        this.ctx.fill();
        // ctx.stroke()
        this.ctx.closePath();
        // 剪切
        this.ctx.clip();
      };
      // 图片自适应
      const adaptiveImg = (img, x, y, mode) => {
        const { imgW: w, imgH: h, width: dWidth, height: dHeight } = img;
        let dw = dWidth / w; //canvas与图片的宽比
        let dh = dHeight / h; //canvas与图片的高比
        // 裁剪图片中间部分
        if ((w > dWidth && h > dHeight) || (w < dWidth && h < dHeight)) {
          if (dw > dh) {
            this.ctx.drawImage(img.path, 0, (h - dHeight / dw) / 2, w, dHeight / dw, x, y, dWidth, dHeight);
          } else {
            this.ctx.drawImage(img.path, (w - dWidth / dh) / 2, 0, dWidth / dh, h, x, y, dWidth, dHeight);
          }
        } else {
          // 拉伸图片
          if (w < dWidth) {
            this.ctx.drawImage(img.path, 0, (h - dHeight / dw) / 2, w, dHeight / dw, x, y, dWidth, dHeight);
          } else {
            this.ctx.drawImage(img.path, (w - dWidth / dh) / 2, 0, dWidth / dh, h, x, y, dWidth, dHeight);
          }
        }
        // 裁剪图片中间部分
        // this.ctx.drawImage(img.path, sx, sy, sWidth, sHeight, x, y, dWidth, dHeight);
        // this.ctx.drawImage(img.path, x, y, temp.dWidth, temp.dHeight);
      };
      // 绘制背景
      this.ctx.setFillStyle(this.backgroundColor);
      this.ctx.fillRect(0, 0, this.width, this.height);
      /* 所有元素入队则开始绘制 */
      if (newVal.length === this.renderList.length) {
        if (newVal.length == 0) {
          this.$emit("on-error", {
            msg: "数据为空",
          });
          return;
        }
        try {
          // console.log('生成的队列：' + JSON.stringify(newVal));
          console.log("开始绘制...");
          for (let i = 0; i < this.drawPathQueue.length; i++) {
            for (let j = 0; j < this.drawPathQueue.length; j++) {
              let current = this.drawPathQueue[j];
              /* 按顺序绘制 */
              if (current.index === i) {
                /* 文本绘制 */
                if (current.type === "text") {
                  console.log("开始绘制...text");
                  fillText(current);
                  this.counter--;
                }
                /* 多行文本 */
                if (current.type === "textarea") {
                  console.log("开始绘制...textarea");
                  fillParagraph(current);
                  this.counter--;
                }
                /* 多行文本 */
                if (current.type === "square") {
                  console.log("开始绘制...square");
                  this.ctx.save(); // 保存上下文，绘制后恢复
                  this.ctx.beginPath(); //开始绘制
                  //画好了圆 剪切  原始画布中剪切任意形状和尺寸。一旦剪切了某个区域，则所有之后的绘图都会被限制在被剪切的区域内 这也是我们要save上下文的原因
                  // 设置旋转中心
                  let offsetX = current.x + Number(current.width) / 2;
                  let offsetY = current.y + Number(current.height) / 2;
                  this.ctx.translate(offsetX, offsetY);
                  let degrees = current.rotate ? Number(current.rotate) % 360 : 0;
                  this.ctx.rotate((degrees * Math.PI) / 180);
                  this.ctx.fillStyle = current.fillStyle;
                  this.ctx.fillRect(current.x - offsetX, current.y - offsetY, current.width, current.height);
                  this.ctx.closePath();
                  this.ctx.restore(); // 恢复之前保存的上下文
                  this.counter--;
                }
                /* 图片绘制 */
                if (current.type === "image") {
                  console.log("开始绘制...image");
                  if (current.area) {
                    // 绘制绘图区域
                    this.ctx.save();
                    this.ctx.beginPath(); //开始绘制
                    this.ctx.rect(current.area.x, current.area.y, current.area.width, current.area.height);
                    this.ctx.clip();
                    // 设置旋转中心
                    let offsetX = current.x + Number(current.width) / 2;
                    let offsetY = current.y + Number(current.height) / 2;
                    this.ctx.translate(offsetX, offsetY);
                    let degrees = current.rotate ? Number(current.rotate) % 360 : 0;
                    this.ctx.rotate((degrees * Math.PI) / 180);
                    this.ctx.drawImage(current.path, current.x - offsetX, current.y - offsetY, current.width, current.height);
                    this.ctx.closePath();
                    this.ctx.restore(); // 恢复之前保存的上下文
                  } else if (current.shape == "circle") {
                    this.ctx.save(); // 保存上下文，绘制后恢复
                    this.ctx.beginPath(); //开始绘制
                    //先画个圆   前两个参数确定了圆心 （x,y） 坐标  第三个参数是圆的半径  四参数是绘图方向  默认是false，即顺时针
                    let width = current.width / 2 + current.x;
                    let height = current.height / 2 + current.y;
                    let r = current.width / 2;
                    this.ctx.arc(width, height, r, 0, Math.PI * 2);
                    this.ctx.lineTo(current.x, current.y);
                    this.ctx.fill();
                    //画好了圆 剪切  原始画布中剪切任意形状和尺寸。一旦剪切了某个区域，则所有之后的绘图都会被限制在被剪切的区域内 这也是我们要save上下文的原因
                    this.ctx.clip();
                    // 设置旋转中心
                    let offsetX = current.x + Number(current.width) / 2;
                    let offsetY = current.y + Number(current.height) / 2;
                    this.ctx.translate(offsetX, offsetY);
                    let degrees = current.rotate ? Number(current.rotate) % 360 : 0;
                    this.ctx.rotate((degrees * Math.PI) / 180);
                    current.mode
                      ? adaptiveImg(current, current.x - offsetX, current.y - offsetY, current.mode)
                      : this.ctx.drawImage(current.path, current.x - offsetX, current.y - offsetY, current.width, current.height);
                    this.ctx.closePath();
                    this.ctx.restore(); // 恢复之前保存的上下文
                  } else if (typeof current.shape == "number") {
                    this.ctx.save(); // 保存上下文，绘制后恢复
                    this.ctx.beginPath(); //开始绘制
                    roundRect(current.x, current.y, current.width, current.height, current.shape);
                    //画好了圆 剪切 原始画布中剪切任意形状和尺寸。一旦剪切了某个区域，则所有之后的绘图都会被限制在被剪切的区域内 这也是我们要save上下文的原因
                    // 设置旋转中心
                    let offsetX = current.x + Number(current.width) / 2;
                    let offsetY = current.y + Number(current.height) / 2;
                    this.ctx.translate(offsetX, offsetY);
                    let degrees = current.rotate ? Number(current.rotate) % 360 : 0;
                    this.ctx.rotate((degrees * Math.PI) / 180);
                    current.mode
                      ? adaptiveImg(current, current.x - offsetX, current.y - offsetY, current.mode)
                      : this.ctx.drawImage(current.path, current.x - offsetX, current.y - offsetY, current.width, current.height);
                    this.ctx.closePath();
                    this.ctx.restore(); // 恢复之前保存的上下文
                  } else {
                    this.ctx.drawImage(current.path, current.x, current.y, current.width, current.height);
                  }
                  this.counter--;
                }
              }
            }
          }
        } catch (err) {
          console.log(err);
          this.$emit("on-error", err);
        }
      }
    },
    counter(newVal, oldVal) {
      if (newVal === 0) {
        this.ctx.draw(false, (draw) => {
          uni.canvasToTempFilePath(
            {
              canvasId: "myCanvas",
              success: (res) => {
                // 在H5平台下，tempFilePath 为 base64
                console.log("图片已保存至本地：", res.tempFilePath);
                this.posterUrl = res.tempFilePath;
                this.$emit("on-success", res.tempFilePath);
              },
              fail: (error) => {
                this.$emit("on-error", error);
              },
            },
            this
          );
        });
      }
    },
  },
  mounted() {
    // this.generateImg();
    // console.log('mounted');
  },
  methods: {
    // #ifdef MP-ALIPAY
    onCanvasReady() {
      const query = my.createSelectorQuery();
      query
        .select("#myCanvas")
        .node()
        .exec((res) => {
          const canvas = res[0].node;
          const ctx = canvas.getContext("2d");
        });
    },
    // #endif

    /**
     * @param {*} elClass 元素名称
     * @param {*} slot 是否采用slot方式
     * @param {*} startX  X偏移
     * @param {*} startY Y偏移
     * @return {*}
     */
    createForElRect(elClass = "Poster", slot = true, startX = 0, startY = 0) {
      uni
        .createSelectorQuery()
        .selectAll("." + elClass)
        .fields(
          {
            dataset: true,
            size: true,
            rect: true,
            value: true,
            scrollOffset: true,
            properties: ["src", "mode"],
            computedStyle: ["margin", "padding", "backgroundColor", "fontSize", "color", "fontWeight", "borderRadius"],
            context: true,
          },
          (res) => {
            let list = [];
            const sys = uni.getSystemInfoSync();
            let multiple = sys.windowWidth / this.width;
            res.forEach((val, index) => {
              let src = val.src || val.dataset.enode || "";
              let type = val.src ? "image" : val.dataset.etype || "text";
              let text = val.dataset.enode || "";
              let size = val.fontSize.replace("px", "") || 13;
              let shape = val.borderRadius == "50%" ? "circle" : val.borderRadius.replace("px", "") * 2;
              let x = (startX + val.left - (slot ? sys.screenWidth : 0)) / multiple;
              let y = (startY + val.top) / multiple;
              y = (startY + val.top - (slot ? 50 : 0)) / multiple;
              // #ifdef H5
              y = (startY + val.top) / multiple;
              // #endif
              list.push({
                type: type,
                shape,
                text,
                mode: "center",
                x,
                y,
                path: src,
                width: val.width / multiple,
                height: val.height / multiple,
                size: size / multiple,
                color: val.color,
              });
            });
            let canvas = uni.createCanvasContext("myCanvas", this);
            this.ctx = canvas;
            this.renderList = [...this.list, ...list];
            this.generateImg();
          }
        )
        .exec();
    },
    create() {
      let canvas = uni.createCanvasContext("myCanvas", this);
      this.ctx = canvas;
      this.renderList = this.list;
      this.generateImg();
    },
    async generateImg() {
      console.log("generateimg");
      this.counter = this.renderList.length;
      this.drawPathQueue = [];
      const getImgInfo = async (current) => {
        console.log("current", current);
        return new Promise((resolve, reject) => {
          uni.getImageInfo
            ? uni.getImageInfo({
                src: current.path,
                success: (res) => {
                  current.path = res.path;
                  current.imgW = res.width;
                  current.imgH = res.height;
                  // this.drawPathQueue.push(current);
                  resolve(current);
                },
              })
            : resolve(current);
        });
      };
      const delayedLog = async (v, i) => {
        let current = this.renderList[i];
        current.index = i;
        /* 如果是文本直接放入队列 */
        if (current.type === "text" || current.type === "textarea" || current.type === "square") {
          // this.drawPathQueue.push(current);
          return current;
        } else {
          return await getImgInfo(current);
          /* 图片需获取本地缓存path放入队列 */
        }
      };
      const processArray = async (array) => {
        // map array to promises
        const promises = array.map((v, i) => delayedLog(v, i));
        // wait until all promises are resolved
        const allData = await Promise.all(promises);
        console.log("Done!", allData);
        return allData;
      };
      this.drawPathQueue = await processArray(this.renderList);
    },
    saveImg() {
      uni.canvasToTempFilePath(
        {
          canvasId: "myCanvas",
          success: (res) => {
            // 在H5平台下，tempFilePath 为 base64
            uni.saveImageToPhotosAlbum({
              filePath: res.tempFilePath,
              success: () => {
                console.log("save success");
              },
            });
          },
        },
        this
      );
    },
  },
};
</script>

<style lang="scss" scoped>
.canvas {
  position: fixed;
  top: 100rpx;
  left: 750rpx;
  width: 100vw;
}
.theContent {
}
</style>
