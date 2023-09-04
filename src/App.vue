<template>
  <div class="maintenancePlanAdd">
    <div class="child-panel-title">
      <h2>图形标注</h2>
    </div>
    <div class="panel-body">
      <div class="demo">
        <canvas id="canvas" :width="width" :height="height"></canvas>
        <div class="draw-btn-group">
          <div style="margin:5px 20px;border:1px solid #333;cursor: pointer;" :class="{active:drawType==''}" title="自由选择" @click="drawTypeChange('')">
            <span>自由选择</span>
          </div>
          <div style="margin:5px 20px;border:1px solid #333;cursor: pointer"  :class="{active:drawType=='rectangle'}" title="画矩形" @click="drawTypeChange('rectangle')">
            <span>画矩形</span>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
<script>
export default {
  name: "App",
  data() {
    return {
      width: 1280,
      height: 720,
      rect: [],
      canvas: {},
      showMenu: false,
      x: "",
      y: "",

      mouseFrom: {},
      mouseTo: {},
      drawType: null,  //当前绘制图像的种类
      canvasObjectIndex: 0,
      textbox: null,
      rectangleLabel: "warning",
      drawWidth: 2, //笔触宽度
      color: "#E34F51", //画笔颜色
      drawingObject: null, //当前绘制对象
      moveCount: 1, //绘制移动计数器
      doDrawing: false, // 绘制状态

      //polygon 相关参数
      polygonMode: false,
      pointArray: [],
      lineArray: [],
      activeShape: false,
      activeLine: "",
      line: {},

      delectKlass: {},
      imgFile: {},
      imgSrc: "",
    };
  },
  watch: {
    drawType() {
      this.canvas.selection = !this.drawType;
    },
    width() {
      this.canvas.setWidth(this.width)
    },
    height() {
      this.canvas.setHeight(this.height)
    },
  },
  methods: {
    // 开始绘制时，指定绘画种类
    drawTypeChange(e) {
      this.drawType = e;
      this.canvas.skipTargetFind = !!e
      if (e == "pen") {
        // isDrawingMode为true 才可以自由绘画
        this.canvas.isDrawingMode = true;
      } else {
        this.canvas.isDrawingMode = false;
      }
    },
    // 鼠标按下时触发
    mousedown(e) {
      // 记录鼠标按下时的坐标
      var xy = e.pointer || this.transformMouse(e.e.offsetX, e.e.offsetY);
      this.mouseFrom.x = xy.x;
      this.mouseFrom.y = xy.y;
      this.doDrawing = true;
      /**   在当前重叠面积中 寻找最小的区域  */
      const searchMinArea = (opt)=>  {
        let { x: curX, y: curY } = opt.pointer;
        let minArea = null;
        const objList = this.canvas.getObjects();
        objList.forEach((item) => {
          /**   无标题 && 有标题  */
            if (
              curX > item.left &&
              curX < item.left + item.width &&
              curY > item.top &&
              curY < item.top + item.height
            ) {
              if (minArea) {
                if (minArea.width * minArea.height > item.width * item.height) {
                  minArea = item;
                }
              } else {
                minArea = item;
              }
              /**   先所有的都降为 0 层  */
              item.moveTo(0);
            }
        });
        return minArea;
      };
      let minArea = searchMinArea(e);
      /**  存在最小区域 && 选中状态  */
      if (minArea && this.drawType === '') {
        /**   先取消选中效果，要不然无法选中底下的  */
        this.canvas.discardActiveObject();
        /**   抬高一层  */
        minArea.moveTo(1);
        /**   手动设置选中效果。  */
        this.canvas.setActiveObject(minArea);
        minArea.set({
          hasControls: true,
        });
      }
      if (this.drawType == "text") {
        this.drawing();
      }

      if (this.textbox) {
        this.textbox.enterEditing();
        this.textbox.hiddenTextarea.focus();
      }
      // 绘制多边形
      if (this.drawType == "polygon") {
        this.canvas.skipTargetFind = false;
        try {
          // 此段为判断是否闭合多边形，点击红点时闭合多边形
          if (this.pointArray.length > 1) {
            // e.target.id == this.pointArray[0].id 表示点击了初始红点
            if (e.target && e.target.id == this.pointArray[0].id) {
              this.generatePolygon();
            }
          }
          //未点击红点则继续作画
          if (this.polygonMode) {
            this.addPoint(e);
          }
        } catch (error) {
        }
      }
    },
    // 鼠标松开执行
    mouseup(e) {
      var xy = e.pointer || this.transformMouse(e.e.offsetX, e.e.offsetY);
      this.mouseTo.x = xy.x;
      this.mouseTo.y = xy.y;
      this.drawingObject = null;
      this.moveCount = 1;
      if (this.drawType != "polygon") {
        this.doDrawing = false;
      }
    },

    //鼠标移动过程中已经完成了绘制
    mousemove(e) {
      if (this.moveCount % 2 && !this.doDrawing) {
        //减少绘制频率
        return;
      }
      this.moveCount++;
      var xy = e.pointer || this.transformMouse(e.e.offsetX, e.e.offsetY);
      this.mouseTo.x = xy.x;
      this.mouseTo.y = xy.y;
      // 多边形与文字框特殊处理
      if (this.drawType != "text" || this.drawType != "polygon") {
        this.drawing(e);
      }
      if (this.drawType == "polygon") {
        if (this.activeLine && this.activeLine.class == "line") {
          var pointer = this.canvas.getPointer(e.e);
          this.activeLine.set({ x2: pointer.x, y2: pointer.y });

          var points = this.activeShape.get("points");
          points[this.pointArray.length] = {
            x: pointer.x,
            y: pointer.y,
            zIndex: 1
          };
          this.activeShape.set({
            points: points
          });
          this.canvas.renderAll();
        }
        this.canvas.renderAll();
      }
    },
    deleteObj() {
      this.canvas.getActiveObjects().map(item => {
        this.canvas.remove(item);
      });
    },
    transformMouse(mouseX, mouseY) {
      return { x: mouseX / 1, y: mouseY / 1 };
    },
    // 绘制多边形开始，绘制多边形和其他图形不一样，需要单独处理
    drawPolygon() {
      this.drawType = "polygon";
      this.polygonMode = true;
      //这里画的多边形，由顶点与线组成
      this.pointArray = new Array();  // 顶点集合
      this.lineArray = new Array();  //线集合
      this.canvas.isDrawingMode = false;
    },
    addPoint(e) {
      var random = Math.floor(Math.random() * 10000);
      var id = new Date().getTime() + random;
      var circle = new fabric.Circle({
        radius: 5,
        fill: "#ffffff",
        stroke: "#333333",
        strokeWidth: 0.5,
        left: (e.pointer.x || e.e.layerX) / this.canvas.getZoom(),
        top: (e.pointer.y || e.e.layerY) / this.canvas.getZoom(),
        selectable: false,
        hasBorders: false,
        hasControls: false,
        originX: "center",
        originY: "center",
        id: id,
        objectCaching: false
      });
      if (this.pointArray.length == 0) {
        circle.set({
          fill: "red"
        });
      }
      var points = [
        (e.pointer.x || e.e.layerX) / this.canvas.getZoom(),
        (e.pointer.y || e.e.layerY) / this.canvas.getZoom(),
        (e.pointer.x || e.e.layerX) / this.canvas.getZoom(),
        (e.pointer.y || e.e.layerY) / this.canvas.getZoom()
      ];

      this.line = new fabric.Line(points, {
        strokeWidth: 2,
        fill: "#999999",
        stroke: "#999999",
        class: "line",
        originX: "center",
        originY: "center",
        selectable: false,
        hasBorders: false,
        hasControls: false,
        evented: false,

        objectCaching: false
      });
      if (this.activeShape) {
        var pos = this.canvas.getPointer(e.e);
        var points = this.activeShape.get("points");
        points.push({
          x: pos.x,
          y: pos.y
        });
        var polygon = new fabric.Polygon(points, {
          stroke: "#333333",
          strokeWidth: 1,
          fill: "#cccccc",
          opacity: 0.3,

          selectable: false,
          hasBorders: false,
          hasControls: false,
          evented: false,
          objectCaching: false
        });
        this.canvas.remove(this.activeShape);
        this.canvas.add(polygon);
        this.activeShape = polygon;
        this.canvas.renderAll();
      } else {
        var polyPoint = [
          {
            x: (e.pointer.x || e.e.layerX) / this.canvas.getZoom(),
            y: (e.pointer.y || e.e.layerY) / this.canvas.getZoom()
          }
        ];
        var polygon = new fabric.Polygon(polyPoint, {
          stroke: "#333333",
          strokeWidth: 1,
          fill: "#cccccc",
          opacity: 0.3,

          selectable: false,
          hasBorders: false,
          hasControls: false,
          evented: false,
          objectCaching: false
        });
        this.activeShape = polygon;
        this.canvas.add(polygon);
      }
      this.activeLine = this.line;

      this.pointArray.push(circle);
      this.lineArray.push(this.line);
      this.canvas.add(this.line);
      this.canvas.add(circle);
    },
    generatePolygon() {
      var points = new Array();
      this.pointArray.map((point, index) => {
        points.push({
          x: point.left,
          y: point.top
        });
        this.canvas.remove(point);
      });
      this.lineArray.map((line, index) => {
        this.canvas.remove(line);
      });
      this.canvas.remove(this.activeShape).remove(this.activeLine);
      var polygon = new fabric.Polygon(points, {
        stroke: this.color,
        strokeWidth: this.drawWidth,
        fill: "rgba(255, 255, 255, 0)",
        opacity: 1,
        hasBorders: true,
        hasControls: false
      });
      this.canvas.add(polygon);

      this.activeLine = null;
      this.activeShape = null;
      this.polygonMode = false;
      this.doDrawing = false;
      this.drawType = null;
    },
    drawing(e) {

      if (this.drawingObject) {
        this.canvas.remove(this.drawingObject);
      }
      var canvasObject = null;
      var left = this.mouseFrom.x,
        top = this.mouseFrom.y,
        mouseFrom = this.mouseFrom,
        mouseTo = this.mouseTo;
      switch (this.drawType) {
        case "rectangle": //长方形
          // 按shift时画正方型
          if (e.e.shiftKey) {
            mouseTo.x - left > mouseTo.y - top ? mouseTo.y = top + mouseTo.x - left : mouseTo.x = left + mouseTo.y - top
          }
          var path =
            "M " +
            mouseFrom.x +
            " " +
            mouseFrom.y +
            " L " +
            mouseTo.x +
            " " +
            mouseFrom.y +
            " L " +
            mouseTo.x +
            " " +
            mouseTo.y +
            " L " +
            mouseFrom.x +
            " " +
            mouseTo.y +
            " L " +
            mouseFrom.x +
            " " +
            mouseFrom.y +
            " z";
          canvasObject = new fabric.Path(path, {
            left: left,
            top: top,
            stroke: this.color,
            strokeWidth: this.drawWidth,
            fill: "rgba(255, 255, 255, 0)",
            hasControls: false
          });
          //也可以使用fabric.Rect
          break;
    

        default:
          break;
      }

      if (canvasObject) {
        // canvasObject.index = getCanvasObjectIndex();\
        this.canvas.add(canvasObject); //.setActiveObject(canvasObject)
        this.drawingObject = canvasObject;
      }
    }
  },
  mounted() {
    this.canvas = new fabric.Canvas("canvas", {
      // skipTargetFind: false, //当为真时，跳过目标检测。目标检测将返回始终未定义。点击选择将无效
      // selectable: false,  //为false时，不能选择对象进行修改
      // selection: false   // 是否可以多个对象为一组
    });
    window.watrixFabric =  this.canvas;
    this.canvas.selectionColor = "rgba(0,0,0,0.05)";
    this.canvas.on("mouse:down", this.mousedown);
    this.canvas.on("mouse:move", this.mousemove);
    this.canvas.on("mouse:up", this.mouseup);

    document.onkeydown = e => {
      // 键盘 delect删除所选元素
      if (e.keyCode == 46) {
        this.deleteObj();
      }
      // ctrl+z 删除最近添加的元素
      if (e.keyCode == 90 && e.ctrlKey) {
        this.canvas.remove(
          this.canvas.getObjects()[this.canvas.getObjects().length - 1]
        );
      }
    };
    // var originalRender = fabric.Textbox.prototype._render;
    // fabric.Textbox.prototype._render = function(ctx) {
    //   originalRender.call(this, ctx);
    //   //Don't draw border if it is active(selected/ editing mode)
    //   if (this.active) return;
    //   if(this.showTextBoxBorder){
    //     var w = this.width,
    //       h = this.height,
    //       x = -this.width / 2,
    //       y = -this.height / 2;
    //     ctx.beginPath();
    //     ctx.moveTo(x, y);
    //     ctx.lineTo(x + w, y);
    //     ctx.lineTo(x + w, y + h);
    //     ctx.lineTo(x, y + h);
    //     ctx.lineTo(x, y);
    //     ctx.closePath();
    //     var stroke = ctx.strokeStyle;
    //     ctx.strokeStyle = this.textboxBorderColor;
    //     ctx.stroke();
    //     ctx.strokeStyle = stroke;
    //   }
    // }
    // fabric.Textbox.prototype.cacheProperties = fabric.Textbox.prototype.cacheProperties.concat('active');
  }
};
</script>

<style scoped>
.el-container {
  flex-direction: column;
}
img,
input {
  display: none;
}
.demo {
  display: flex;
  flex-direction: column;
  align-items: center;
}
canvas {
  border: 1px dashed black;
}

.draw-btn-group div i{
  display: flex;
      background-repeat: no-repeat;
      background-size: 80%;
      background-position: 50% 50%;
      height: 30px;
      width: 30px;
    }
.draw-btn-group{
  display: flex;
}
.icon-1 {
    background-image: url("./assets/icons/draw/1.png");
}
.icon-pentagram {
  background-image: url("./assets/icons/draw/pentagram.png");
}
.icon-2 {
  background-image: url("./assets/icons/draw/2.png");
}
.icon-3 {
  background-image: url("./assets/icons/draw/3.png");
}
.icon-4 {
  background-image: url("./assets/icons/draw/4.png");
  background-size: 75%;
}
.icon-5 {
  background-image: url("./assets/icons/draw/5.png");
  background-size: 70%;
}
.icon-6 {
  background-image: url("./assets/icons/draw/6.png");
}
.icon-7 {
  background-image: url("./assets/icons/draw/7.png");
  background-size: 80%;
}
.icon-del {
  background-image: url("./assets/icons/draw/del.png");
  background-size: 90%;
}
.icon-img {
  background-image: url("./assets/icons/draw/img.png");
  background-size: 80%;
}
.icon-back {
  background-image: url("./assets/icons/draw/back.png");
  background-size: 75%;
}
.icon-save {
  background-image: url("./assets/icons/draw/save.png");
  background-size: 80%;
}
.icon-mouse {
  background-image: url("./assets/icons/draw/mouse.png");
  background-size: 60%;
}

.active{
  background: #eee;
  background-position: 50% 50%;
  background-repeat: no-repeat;
}
</style>
