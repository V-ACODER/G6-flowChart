<template>
  <div>
    <header class="tool-bar">
      <button @click="runGraph()" v-show="!running">运行</button>
      <button @click="endRunning()" v-show="running">结束</button>
      <button @click="zoomGraph(-1)">缩小</button>
      <button @click="zoomGraph(1)">放大</button>
      <button @click="setFitView()">自动适应</button>
      <button v-if="isFullScreen" @click="setFullScreen(false)">收起</button>
      <button v-else @click="setFullScreen(true)">全屏</button>
      <button @click="addNewWindow()" :disabled="windows.length > 5">新建窗口</button>
    </header>
    <div class="editor-container">
      <ul class="left-menu" id="leftMenu">
        <li
          v-for="(item, index) in list"
          :key="index"
          class="left-item"
          draggable
          @dragstart="handleDragstart"
          @dragend="handleDragEnd($event,item)"
        >
          <span
            class="pannel-type-icon"
            :style="{background:'url('+item.icon+') no-repeat left top', backgroundSize: '100%'}"
          ></span>
          {{item.label}}
        </li>
      </ul>

      <div class="main-section">
        <el-tabs type="card" @tab-click="checkoutWindow" v-model="activeWindow">
          <el-tab-pane>
            <span slot="label">窗口1</span>
            <div id="graph-container" class="graph-container" :name="0"></div>
          </el-tab-pane>
          <el-tab-pane v-for="(w, index) in windows" :key="index" :label="'窗口'+w">
            <span slot="label">
              窗口{{w}}
              <i class="el-icon-circle-close" @click="closeWindow(w)"></i>
            </span>
            <div :id="'graph-container'+(index+1)" class="graph-container" :name="index+1"></div>
          </el-tab-pane>
        </el-tabs>
      </div>
      <div class="right-panel" id="rightPanel">
        <el-tabs v-model="activeName" type="card" @tab-click="handleClick">
          <el-tab-pane v-for="(item, index) in rightMenu" :key="index" :label="item" :name="item">名称</el-tab-pane>
        </el-tabs>
      </div>

      <!-- 右击菜单 -->
      <ul class="el-scrollbar__view el-select-dropdown__list context-menu" id="contextMenu">
        <li
          class="el-select-dropdown__item"
          v-for="menu in menus"
          :key="menu.key"
          @click="handleClick(menu)"
        >{{menu.name}}</li>
      </ul>
    </div>

    <!-- 重命名弹框 -->
    <el-dialog title="重命名" :visible.sync="dialogFormVisible">
      <el-input v-model="labelName" :placeholder="labelName" autocomplete="off"></el-input>
      <div slot="footer" class="dialog-footer">
        <el-button @click="dialogFormVisible = false">取 消</el-button>
        <el-button type="primary" @click="modifyLabelName">确 定</el-button>
      </div>
    </el-dialog>
  </div>
</template>
<script>
import G6 from "@antv/g6";
import uniqueId from "lodash/uniqueId";
export default {
  data() {
    return {
      graph: null,
      offsetX: 0,
      offsetY: 0,
      list: [
        {
          label: "数据源写入",
          icon: "/images/database.png",
          rightMenu: ["字段设置", "参数设置"]
        },
        {
          label: "数据归一化",
          icon: "/images/database.png",
          rightMenu: ["执行调优"]
        },
        {
          label: "XGboost",
          icon: "/images/database.png",
          rightMenu: ["字段设置"]
        }
      ],
      data: {
        type: Object,
        default: () => {}
      },
      running: false,
      menus: [
        { key: 1, name: "重命名" },
        { key: 2, name: "删除" }
      ],
      selectedItem: null,
      selectedEdge: null,
      rightMenu: [],
      labelName: "",
      dialogFormVisible: false,
      activeName: "",
      isFullScreen: false,
      documentWidth: 0,
      documentHeight: 0,
      windows: [],
      activeWindow: 0
    };
  },
  created() {},
  mounted() {
    this.$nextTick(() => {
      this.init();
    });
  },
  methods: {
    handleDragstart(e) {
      this.offsetX = e.offsetX;
      this.offsetY = e.offsetY;
    },
    handleDragEnd(e, item) {
      let data = {};
      Object.assign(data, item);
      data.offsetX = this.offsetX;
      data.offsetY = this.offsetY;
      const graph = this.graph;
      const xy = graph.getPointByClient(e.x, e.y);
      data.x = xy.x;
      data.y = xy.y;
      data.id = data.type + uniqueId();
      graph.addItem("node", data);
    },

    init() {
      this.documentHeight = document.documentElement.clientHeight;
      const height = this.documentHeight - 85;
      this.documentWidth = document.documentElement.clientWidth;
      const width = this.documentWidth - 408;
      const _this = this;
      // 封装点击添加边的交互
      G6.registerBehavior("click-add-edge", {
        // 设定该自定义行为需要监听的事件及其响应函数
        getEvents() {
          return {
            "node:click": "onClick", // 监听事件 node:click，响应函数是 onClick
            mousemove: "onMousemove", // 监听事件 mousemove，响应函数是 onMousemove
            "edge:click": "onEdgeClick" // 监听事件 edge:click，响应函数是 onEdgeClick
          };
        },
        // getEvents 中定义的 'node:click' 的响应函数
        onClick(ev) {
          const type = ev.shape.attrs.cursor;
          if (type === "crosshair") {
            const node = ev.item;
            // 鼠标当前点击的节点的位置
            const point = { x: ev.x, y: ev.y };
            const model = node.getModel();
            if (this.addingEdge && this.edge) {
              this.graph.updateItem(this.edge, {
                target: model.id
              });

              this.edge = null;
              this.addingEdge = false;
            } else {
              // 在图上新增一条边，结束点是鼠标当前点击的节点的位置
              this.edge = this.graph.addItem("edge", {
                source: model.id,
                target: point
              });
              this.addingEdge = true;
            }
          }
        },
        // getEvents 中定义的 mousemove 的响应函数
        onMousemove(ev) {
          // 鼠标的当前位置
          const point = { x: ev.x, y: ev.y };
          if (this.addingEdge && this.edge) {
            // 更新边的结束点位置为当前鼠标位置
            this.graph.updateItem(this.edge, {
              target: point
            });
          }
        },
        // getEvents 中定义的 'edge:click' 的响应函数
        onEdgeClick(ev) {
          _this.selectedEdge = ev.item;
          _this.resetSelectedEdge();
          // 拖拽过程中，点击会点击到新增的边上
          // if (this.addingEdge && this.edge == currentEdge) {
          //   this.graph.removeItem(this.edge);
          //   this.edge = null;
          //   this.addingEdge = false;
          // }
        }
      });

      const dashArray = [
        [0, 1],
        [0, 2],
        [1, 2],
        [0, 1, 1, 2],
        [0, 2, 1, 2],
        [1, 2, 1, 2],
        [2, 2, 1, 2],
        [3, 2, 1, 2],
        [4, 2, 1, 2]
      ];

      const lineDash = [4, 2, 1, 2];
      const interval = 9; // lineDash 的和
      G6.registerEdge(
        "line-dash",
        {
          afterDraw(cfg, group) {
            // 获得该边的第一个图形，这里是边的 path
            const shape = group.get("children")[0];
            // 获得边的 path 的总长度
            const length = shape.getTotalLength();
            let totalArray = [];
            // 计算出整条线的 lineDash
            for (let i = 0; i < length; i += interval) {
              totalArray = totalArray.concat(lineDash);
            }

            let index = 0;
            // 边 path 图形的动画
            shape.animate(
              () => {
                // 每一帧的操作，入参 ratio：这一帧的比例值（Number）。返回值：这一帧需要变化的参数集（Object）。
                const cfg = {
                  lineDash: dashArray[index].concat(totalArray)
                };
                // 每次移动 1
                index = (index + 1) % interval;
                // 返回需要修改的参数集，这里只修改了 lineDash
                return cfg;
              },
              {
                repeat: true, // 动画重复
                duration: 3000 // 一次动画的时长为 3000
              }
            );
          }
        },
        "cubic"
      ); // 该自定义边继承了内置三阶贝塞尔曲线边 cubic

      this.graph = new G6.Graph({
        container: "graph-container",
        height: height,
        width: width,
        fitView: true,
        fitViewPadding: 0,
        minZoom: 0.5,
        maxZoom: 2,
        defaultNode: {
          type: "modelRect",
          size: [200, 40],
          labelCfg: {
            style: {}
          },
          style: {
            fill: "#fff",
            radius: 4,
            cursor: "move"
          },
          preRect: {
            show: true,
            width: 4
          },
          logoIcon: {
            show: true,
            img: "/images/database.png",
            width: 20,
            height: 24,
            offset: 0
          },
          stateIcon: {
            show: true,
            img: "/images/upload.png"
          },
          linkPoints: {
            top: true,
            bottom: true,
            fill: "#fff",
            lineWidth: 2,
            cursor: "crosshair"
          },
          anchorPoints: [
            [0.5, 0],
            [0.5, 1]
          ]
        },
        defaultEdge: {
          type: "cubic-vertical",
          // sourceAnchor: 0,
          // targetAnchor: 0,
          style: {
            endArrow: {
              path: "M 0,0 L 8,4 L 8,-4 Z",
              fill: "#F6BD16"
            },
            lineWidth: 2,
            lineAppendWidth: 5,
            cursor: "pointer",
            stroke: "#F6BD16"
          },
          labelCfg: {
            autoRotate: true
          }
        },
        nodeStateStyles: {
          selected: {
            fill: "steelblue",
            fillOpacity: 0.1
          },
          hover: {}
        },
        modes: {
          default: [
            "drag-node",
            "drag-canvas",
            "zoom-canvas",
            "click-select",
            "click-add-edge"
          ]
        }
      });
      let data = this.data;
      if (data) {
        this.graph.read(data);
      }
      const menu = document.getElementById("contextMenu");
      this.graph.on("node:contextmenu", evt => {
        evt.preventDefault();
        evt.stopPropagation();
        menu.style.left = `${evt.clientX}px`;
        menu.style.top = `${evt.clientY}px`;
        menu.style.display = "block";
        this.selectedItem = evt.item;
        let model = evt.item._cfg.model;
        this.rightMenu = model.rightMenu;
        this.labelName = model.label;
      });

      this.graph.on("node:click", evt => {
        let item = evt.item;
        this.selectedItem = item;
        let model = item._cfg.model;
        this.rightMenu = model.rightMenu;
        this.labelName = model.label;
        this.selectedEdge = null;
        this.resetSelectedEdge();
      });
      this.graph.on("node:mouseleave", () => {
        menu.style.display = "none";
        // this.selectedItem = null;
        // this.rightMenu = [];
      });
      this.graph.on("keyup", evt => {
        if (evt.keyCode === 46) {
          this.graph.remove(this.selectedEdge);
          this.graph.remove(this.selectedItem);
        }
      });
    },

    handleClick(item) {
      const menu = document.getElementById("contextMenu");
      menu.style.display = "none";
      switch (item.key) {
        case 1:
          this.dialogFormVisible = true;
          break;
        case 2:
          this.graph.remove(this.selectedItem);
          break;
      }
    },
    modifyLabelName() {
      this.graph.updateItem(this.selectedItem, {
        label: this.labelName
      });
      this.dialogFormVisible = false;
      // this.labelName = "";
    },
    async runGraph() {
      this.running = true;
      const graph = this.graph;
      let adges = graph.findAll("edge", item => item);
      let nodes = graph.findAll("node", item => item);
      this.graph.updateItem(adges[0], {
        type: "line-dash"
      });
      for (let i = 0; i < nodes.length; i++) {
        await this.animateEdge(adges, nodes, i);
      }
      setTimeout(() => {
        this.graph.updateItem(nodes[nodes.length - 2], {
          stateIcon: {
            show: true,
            img: "/images/ok.png"
          }
        });
      }, 1000);
      setTimeout(() => {
        this.graph.updateItem(adges[adges.length - 1], {
          type: "cubic-vertical"
        });
        this.graph.updateItem(nodes[nodes.length - 1], {
          stateIcon: {
            show: true,
            img: "/images/ok.png"
          }
        });
      }, 2000);
    },
    animateEdge(adges, nodes, index) {
      return new Promise(resolve => {
        setTimeout(
          () => {
            if (index > 0) {
              this.graph.updateItem(adges[index - 2], {
                type: "cubic-vertical"
              });
              this.graph.updateItem(nodes[index - 1], {
                stateIcon: {
                  show: true,
                  img: "/images/ok.png"
                }
              });
            }
            this.graph.updateItem(adges[index - 1], {
              type: "line-dash"
            });
            this.graph.updateItem(nodes[index], {
              stateIcon: {
                show: true,
                img: "/images/wait.png"
              }
            });
            resolve();
          },
          index > 0 ? 1000 : 0
        );
      });
    },
    endRunning() {
      this.running = false;
      const graph = this.graph;
      graph.findAll("edge", item => {
        graph.updateItem(item, {
          type: "cubic-vertical"
        });
      });
    },
    zoomGraph(value) {
      const graph = this.graph;
      if (value < 0) {
        graph.zoom(0.9);
      } else {
        graph.zoom(1.1);
      }
    },
    setFitView() {
      this.graph.fitView(10);
    },
    setFullScreen(bool) {
      this.isFullScreen = bool;
      const leftMenu = document.getElementById("leftMenu");
      const rightPanel = document.getElementById("rightPanel");
      if (bool) {
        leftMenu.style.display = "none";
        rightPanel.style.display = "none";
        this.graph.changeSize(this.documentWidth - 4, this.documentHeight - 85);
      } else {
        leftMenu.style.display = "block";
        rightPanel.style.display = "block";
        this.graph.changeSize(
          this.documentWidth - 408,
          this.documentHeight - 44
        );
      }
    },
    resetSelectedEdge() {
      this.graph.findAll("edge", item => {
        this.graph.updateItem(item, {
          style: {
            lineWidth: 2,
            opacity: 1
          }
        });
      });
      this.graph.updateItem(this.selectedEdge, {
        style: {
          lineWidth: 4,
          opacity: 0.3
        }
      });
    },
    addNewWindow() {
      let length = this.windows.length;
      if (length === 0) {
        this.windows.push(2);
      } else {
        this.windows.push(this.windows[length - 1] + 1);
      }
    },
    closeWindow(index) {
      this.windows = this.windows.filter(item => item !== index);
    },
    checkoutWindow() {}
  }
};
</script>
<style lang="scss" scoped>
.tool-bar {
  line-height: 40px;
  background-color: rgb(248, 248, 248);
  border-bottom: 2px solid grey;
  button {
    padding: 2px 6px;
    margin-right: 10px;
  }
}
.editor-container {
  display: flex;
  .left-menu {
    background-color: rgb(248, 248, 248);
    height: 100vh;
    padding: 20px;
    box-sizing: border-box;
  }
  .left-item {
    list-style-type: none;
    width: 140px;
    line-height: 40px;
    text-align: left;
    background: transparent;
    padding: 0 10px;
    border-radius: 4px;
    border: 2px solid transparent;
    &:hover {
      border-color: #21a3dd;
      cursor: move;
    }
  }
  .pannel-type-icon {
    width: 20px;
    height: 24px;
    display: inline-block;
    vertical-align: middle;
    margin-right: 4px;
    background-size: 100%;
  }
}
.graph-container {
  position: relative;
  cursor: pointer;
}
.right-panel {
  width: 200px;
  background-color: rgb(248, 248, 248);
}
.context-menu {
  position: absolute;
  border: 1px solid #e4e7ed;
  border-radius: 4px;
  background-color: #fff;
  box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.1);
  box-sizing: border-box;
  margin: 5px 0;
  z-index: 1;
  display: none;
}

.context-menu li {
  cursor: pointer;
  font-size: 12px;
  height: 28px;
  line-height: 28px;
  list-style-type: none;
  padding: 0 10px;
}
.context-menu li:hover {
  background-color: #f5f7fa;
}
.main-section {
  border: 2px solid grey;
  border-top: 0;
  width: calc(100% - 408);
}
</style>