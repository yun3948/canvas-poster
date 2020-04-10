<template>
  <div>
    <div id="img-container" v-if="createImg">

    </div>

    <div v-else class="poster-container">
      <div class="posterBox">
        <!-- <p class="posterTit">选择图片和设置内容</p> -->
        <div class="editFixed">
          <p class="editone" @click="titShow">编辑文字</p>
          <p class="editimg" @click="editImg" v-if="!showImg">修改背景</p>
        </div>
        <div class="editbtn" v-if="!showTit">
          <!-- <el-button type="primary" class="editone" >编辑</el-button>
          <el-button class="editimg" @click="editImg">修改图片</el-button>-->
        </div>
        <p class="posterSave">
          <el-button type="primary" class="editimg" @click="saveImg">保存图片</el-button>
        </p>
        <div class="editBox">
          <transition name="slide-fade">
            <p class="edittow" v-if="showTit">
              <el-button-group>
                <el-button type="primary" icon="el-icon-user" @click="toggleName">姓名</el-button>
                <el-button type="primary" icon="el-icon-phone-outline" @click="togglePhone">手机</el-button>
                <el-button type="primary" icon="el-icon-picture" @click="toggleQrcode">直播二维码</el-button>
                <el-button type="primary" icon="el-icon-edit" @click="addText">自定义</el-button>
                <el-button type="primary" icon="el-icon-arrow-left" @click="titShow">收起编辑</el-button>
              </el-button-group>
            </p>
          </transition>
        </div>
        <div v-if="showImg" class="imgBox" style>
          <p
            style="width:100%;text-align:center;margin:10px 0 0 0;color: #8a8989;margin-top:26%"
          >左右滑动选择图片</p>
          <p class="thumb-list">
            <img v-for="(img,index) in imgs" :key="index" :src="img" @click="changeBg(img)" alt />
          </p>
          <p class="imgclose" @click="editImg">取消</p>
          <!-- <p></p> -->
        </div>
      </div>

      <div id="canvas" ref="canvas" style="position:relative;" width="100%"></div>
      <el-dialog title="文本编辑" :visible.sync="dialogVisible" width="90%">
        <div>
          <el-form ref="form" :model="form" label-width="80px">
            <el-form-item label="输入文本">
              <el-input v-model="form.text"></el-input>
            </el-form-item>
            <el-form-item label="选择颜色">
              <el-color-picker v-model="form.color"></el-color-picker>
            </el-form-item>
            <el-form-item label="选择字号">
              <el-select v-model="form.size" placeholder="请选择">
                <el-option
                  v-for="item in fontSizeList"
                  :key="item.value"
                  :label="item.label"
                  :value="item.value"
                ></el-option>
              </el-select>
            </el-form-item>
          </el-form>
        </div>
        <span slot="footer" class="dialog-footer">
          <el-button @click="dialogVisible = false">取 消</el-button>
          <el-button type="primary" @click="save_form">确 定</el-button>
        </span>
      </el-dialog>
    </div>
  </div>
</template>

<script>
import api from "../api.js";
import * as PIXI from "pixi.js";

// import Tink from "../library/pixi-think";
var timeEvent;
var app;
 
var vm;
var objText;
// eslint-disable-next-line no-unused-vars
const loader = new PIXI.Loader();
var resourcesArr = [];
export default {
  name: "poster",
  data() {
    return {
      showTit: false,
      showImg: true,
      id: 0,
      imgs: [],
      bg: "",
      name: "",
      phone: "",
      info: {},
      dialogVisible: false,
      form: {
        text: "",
        color: "#333333",
        size: 12
      },
      fontSizeList: [],
      createImg:false,
    };
  },
  created: function() {
    this.id = this.$route.params.id;
    if (this.$route.query.id) {
      this.id = this.$route.query.id;
    }

    this.load_data();
    this.load_poster();
  },

  mounted: function() {
    vm = this;
    this.init_canvas();
    requestAnimationFrame( animate );
  },

  methods: {
    // 点击显示隐藏编辑行
    titShow() {
      this.showTit = !this.showTit;
    },
    editImg() {
      this.showImg = !this.showImg;
    },

    init_font_size() {
      let fontSizeList = [];
      let fontArr = [12, 14, 16, 18, 24, 28, 32, 36];
      fontArr.forEach(function(i) {
        let tmp = {};
        tmp.label = i;
        tmp.value = i;
        fontSizeList.push(tmp);
      });
      this.fontSizeList = fontSizeList;
    },
    addText() {
      this.dialogVisible = true;
    },

    add_text(selftext, style) {
      let text = new PIXI.Text(selftext, style);
      //随机位置 页面中心位置
      let x = app.stage.width / 3 + (Math.floor(Math.random() * 100) + 1);
      let y = app.stage.height / 3 + (Math.floor(Math.random() * 100) + 1);
      text.position.set(x, y);
      text.interactive = true;
      text.buttonMode = true;

      addInteraction(text);

      app.stage.addChild(text);
      return text;
    },

    save_form() {
      //保存表单 添加文字
      let style = {
        wordWrap: true,
        wordWrapWidth: 100,
        align: "center"
      };
      style.fill = this.form.color;
      style.fontSize = this.form.size;

      if (objText) {
        objText.text = this.form.text;
        objText.style = style;
      } else {
        this.add_text(this.form.text, style);
      }

      this.dialogVisible = false;
    },
    init_canvas() {
      app = new PIXI.Application({
        width: 256,
        height: 256,
        antialias: true,

        transparent: true,
        backgroundColor: 0xcccccc
      });
      app.renderer.view.style.position = "absolute";
      app.renderer.view.style.display = "block";
      app.renderer.autoResize = true;
      app.renderer.resize(window.innerWidth, window.innerHeight);
      document.querySelector("#canvas").appendChild(app.view);
    },

    async load_poster() {
      const json = await api.load_poster_list();
      if (json.code == 200) {
        this.imgs = json.datas;
        this.load_imgs(this.imgs);
        console.log(this.imgs);
      }
    },

    async load_data() {
      const json = await api.load_poster_data({ id: this.id });
      if (json.code == 200) {
        this.info = json.datas;
        this.init_font_size();
        if (json.datas.qrcode) {
          // this.load_imgs(json.datas.qrcode);
        }
      } else {
        this.$message.error("获取信息失败！无法生成直播二维码！");
      }
    },

    load_imgs(imgs) {
      loader.add(imgs);
      loader.load((loader, resources) => {
        // eslint-disable-next-line no-const-assign
        resourcesArr = resources;
      });
    },

    changeBg(img) {
      // 删除所有
      app.stage.removeChildren();
      // 计算比例
      this.showImg = false;
      const bg = new PIXI.Sprite(resourcesArr[img].texture);

      //   let _width = bg.width;
      let height = bg.height;

      let scale = app.renderer.height / height;

      bg.scale.set(scale, scale);

      app.stage.addChild(bg);
    },

    toggleQrcode() {
      const qrcode = new PIXI.Sprite.from(this.info.qrcode);

      let x = app.stage.width / 3 + (Math.floor(Math.random() * 100) + 1);
      let y = app.stage.height / 3 + (Math.floor(Math.random() * 100) + 1);

      qrcode.position.set(x, y);
      
      qrcode.scale.set(0.2, 0.2);

      addInteraction(qrcode);

      app.stage.addChild(qrcode);
    },

    togglePhone() {
      if (this.phone != "") {
        this.phone.visible = !this.phone.visible;
        return false;
      }

      let phone = this.info.phone;
      let style = {
        wordWrap: true,
        wordWrapWidth: 100,
        align: "center",
        fill: "#000000"
      };
      let obj = this.add_text(phone, style);
      this.phone = obj;
    },

    toggleName() {
      if (this.name != "") {
        this.name.visible = !this.name.visible;
        return false;
      }
      let style = {
        wordWrap: true,
        wordWrapWidth: 100,
        align: "center",
        fill: "#000000"
      };

      let obj = this.add_text(this.info.name, style);
      this.name = obj;
    },

    saveImg() {
      this.createImg = true;
      var image = app.renderer.plugins.extract.image(app.stage);

      image.onload = () => {
        document.getElementById('img-container').appendChild(image);
      };
    }
  }
};



function animate() {

    requestAnimationFrame(animate);

    // render the stage
    app.renderer.render(app.stage);
}
 
function addInteraction(obj) {
  obj.interactive = true;
  obj.anchor.set(0.5);
  obj
    .on("mousedown", onDragStart)
    .on("touchstart", onDragStart)
    .on("mouseup", onDragEnd)
    .on("mouseupoutside", onDragEnd)
    .on("touchend", onDragEnd)
    .on("touchendoutside", onDragEnd)
    .on("mousemove", onDragMove)
    .on("touchmove", onDragMove);
}

function onDragStart(event) {
  // store a reference to the data
  // the reason for this is because of multitouch
  // we want to track the movement of this particular touch
  clearTimeout(timeEvent);
  let that = this;
  objText = that;

  timeEvent = setTimeout(() => {
    longpress(objText);
  }, 600);

  this.data = event.data;
  this.alpha = 0.5;
  this.dragging = true;
}

function longpress(obj) {
  //弹窗提示编辑或删除
  vm.$confirm("想要如何操作？", "提示", {
    distinguishCancelAndClose: true,
    confirmButtonText: "编辑",
    cancelButtonText: "删除"
  })
    .then(() => {
      vm.form.text = obj.text;
      vm.form.size = obj.style.fontSize;
      vm.form.color = obj.style.fill || "#333333";
      //弹窗修改
      vm.dialogVisible = true;
    })
    .catch(action => {
      if (action == "cancel") {
        // 删除
        app.stage.removeChild(obj);
        console.log("delete");
      } else {
        //关闭
      }
    });
}

function onDragEnd() {
  clearTimeout(timeEvent);
  this.alpha = 1;
  this.dragging = false;
  // set the interaction data to null
  this.data = null;
}

function onDragMove() {
  clearTimeout(timeEvent);
  timeEvent = 0;

 
  if (this.dragging) {
    var newPosition = this.data.getLocalPosition(this.parent);
    this.position.x = newPosition.x;
    this.position.y = newPosition.y;
  }
}
</script>

<style>
.posterBox {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  /* background: #fff; */
  z-index: 99;
}
.thumb-list img {
  height: 200px;
}
.posterTit {
  width: 100%;
  text-align: center;
  padding: 10px 0;
}
.editFixed {
  position: absolute;
  top: 7%;
  left: 0;
  /* background: #44b294; */
  border-bottom-right-radius: 5px;
  border-top-right-radius: 5px;
  color: #fff;
  /* padding: 6px; */
}
.editFixed p {
  /* float: left; */
}
.editBox {
  width: 100%;
  overflow: hidden;
  position: absolute;
  top: 7%;
  left: 0;
}
.editBox > p {
  /* float: left; */
}
.editbtn {
  width: 100%;
  text-align: center;
}
.editimg {
  /* width: 40%; */
  background: #44b294;
  text-align: center;
  color: #fff;
  border-radius: 5px;
  padding: 4px 6px;
  font-size: 14px;
  border-bottom-left-radius: 0;
  border-top-left-radius: 0;
}
.editone {
  /* width: 40%; */
  background: #44b294;
  text-align: center;
  color: #fff;
  border-radius: 5px;
  padding: 4px 6px;
  font-size: 14px;
  margin: 6px 0;
  border-bottom-left-radius: 0;
  border-top-left-radius: 0;
}
.editBox > p.edittow {
  width: 93%;
}
.slide-fade-enter-active {
  transition: all 0.3s ease;
}
.slide-fade-leave-active {
  transition: all 0.5s cubic-bezier(0.8, 1, 1, 0.5); /*cubic-bezier(<x1>, <y1>, <x2>, <y2>*/
}
.slide-fade-enter,
.slide-fade-leave-to {
  transform: translateX(-1000px);
  opacity: 0;
}

.thumb-list {
  width: 90%;
  margin: 10px auto;
  height: 222px;
  overflow-x: scroll;
  white-space: nowrap;
  padding: 6px 2%;
  border: 1px solid #f2f6fc;
  border-radius: 3px;
}
.thumb-list img {
  display: inline-block;
  margin: 6px;
}
.posterSave {
  width: 100%;
  margin: 20px auto;
  text-align: center;
  position: fixed;
  bottom: 0;
  left: 0;
}
.posterSave button {
}
.imgBox {
  background: rgb(243, 243, 243);
}
#canvas {
  /* margin-top: 20%; */
}
.imgclose{
  width: 100%;
  text-align: center;
  background: #eaeaea;
  color: #777;
  padding: 6px 0;
}
</style>