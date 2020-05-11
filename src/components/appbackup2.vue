<template>
  <div id="app">
    <header id="example"></header>
    <div id="container"></div>
  </div>
</template>

<script>
import G6 from "@antv/g6";

export default {
  name: "App",
  components: {},
  data() {
    return {
      container: null,
      modelList: []
    };
  },
  mounted() {
    this.$nextTick(() => {
      this.drawExample();
      this.drawContainer();
    });
  },
  methods: {
    drawExample() {
      const data = {
        nodes: [
          {
            x: 100,
            y: 50,
            type: "rect",
            label: "A"
          },
          {
            x: 300,
            y: 50,
            type: "rect",
            label: "B"
          },
          {
            x: 500,
            y: 50,
            type: "rect",
            label: "C"
          }
        ]
      };
      const graph = new G6.Graph({
        container: "example",
        width: 1500,
        height: 100
      });
      graph.data(data);
      graph.render();
      graph.on("node:dblclick", evt => {
        const node = evt.item;
        const model = node.getModel();
        this.addNewNode(model);
      });
    },
    addNewNode(model) {
      model.id = `node-${this.modelList.length + 1}`;
      this.modelList.push(model);
      const data = {
        nodes: this.modelList
      };
      this.container.data(data);
      this.container.render();
    },
    drawContainer() {
      const data = {
        nodes: [
          {
            id: "node1",
            x: 100,
            y: 200
          },
          {
            id: "node2",
            x: 300,
            y: 200
          },
          {
            id: "node3",
            x: 300,
            y: 300
          }
        ]
      };
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
          console.log(ev);
          const node = ev.item;
          // 鼠标当前点击的节点的位置
          const point = { x: ev.x, y: ev.y };
          const model = node.getModel();
          if (this.addingEdge && this.edge) {
            graph.updateItem(this.edge, {
              target: model.id
            });

            this.edge = null;
            this.addingEdge = false;
          } else {
            // 在图上新增一条边，结束点是鼠标当前点击的节点的位置
            this.edge = graph.addItem("edge", {
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
            graph.updateItem(this.edge, {
              target: point
            });
          }
        },
        // getEvents 中定义的 'edge:click' 的响应函数
        onEdgeClick(ev) {
          const currentEdge = ev.item;
          // 拖拽过程中，点击会点击到新增的边上
          if (this.addingEdge && this.edge == currentEdge) {
            graph.removeItem(this.edge);
            this.edge = null;
            this.addingEdge = false;
          }
        }
      });
      const graph = (this.container = new G6.Graph({
        container: "container",
        width: document.getElementById("container").scrollWidth,
        height: document.getElementById("container").scrollHeight || 500,
        modes: {
          default: ["click-add-edge"]
        },
        nodeStateStyles: {
          // 节点在 selected 状态下的样式，对应内置的 click-select 行为
          selected: {
            stroke: "#666",
            lineWidth: 2,
            fill: "steelblue"
          }
        }
      }));
      this.container.data(data);
      this.container.render();
    }
  }
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
#container {
  background: #efefef;
}
header {
  line-height: 80px;
  background: lightgray;
}
</style>
