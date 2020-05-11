<template>
  <div>
    <header class="tool-bar">
      <button @click="runGraph()" v-show="!running">运行</button>
      <button @click="endRunning()" v-show="running">结束</button>
      <button @click="zoomGraph(-1)">缩小</button>
      <button @click="zoomGraph(1)">放大</button>
      <button @click="setFitView()">自动适应</button>
      <button>全屏</button>
    </header>
    <div class="editor-container">
      <ul class="left-menu">
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
      <div id="graph-container" class="graph-container"></div>
      <div class="right-panel"></div>
    </div>
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
          label: "mysql数据同步",
          icon: "/images/database.png"
        },
        {
          label: "hive数据同步",
          icon: "/images/database.png"
        },
        {
          label: "特征选择",
          icon: "/images/database.png"
        }
      ],
      data: {
        type: Object,
        default: () => {}
      },
      running: false
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
      const height = document.documentElement.clientHeight - 44;
      const width = document.documentElement.clientWidth - 404;
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
          const currentEdge = ev.item;
          // 拖拽过程中，点击会点击到新增的边上
          if (this.addingEdge && this.edge == currentEdge) {
            this.graph.removeItem(this.edge);
            this.edge = null;
            this.addingEdge = false;
          }
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
            show: true
          },
          linkPoints: {
            top: true,
            bottom: true,
            fill: "#fff",
            lineWidth: 2
          },
          anchorPoints: [
            [0.5, 0],
            [0.5, 1]
          ]
        },
        defaultEdge: {
          type: "cubic-vertical",
          style: {
            endArrow: {
              path: "M 0,0 L 8,4 L 8,-4 Z",
              fill: "#F6BD16"
            },
            lineWidth: 2,
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
          }
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
    },
    async runGraph() {
      this.running = true;
      const graph = this.graph;
      let adges = [];
      graph.findAll("edge", item => {
        adges.push(item);
      });
      for (let i = 0; i < adges.length; i++) {
        await this.animateEdge(adges[i], adges[i - 1]);
      }
    },
    animateEdge(item, lastItem) {
      return new Promise(resolve => {
        setTimeout(
          () => {
            if (lastItem) {
              this.graph.updateItem(lastItem, {
                type: "cubic-vertical"
              });
            }
            this.graph.updateItem(item, {
              type: "line-dash"
            });
            resolve();
          },
          lastItem ? 2000 : 1000
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
      this.graph.set("fitView", true);
    }
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
  border: 2px solid grey;
  border-top: none;
}
.right-panel {
  width: 200px;
  background-color: rgb(248, 248, 248);
}
</style>