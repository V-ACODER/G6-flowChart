<template>
  <div>
    <div id="mountNode"></div>
  </div>
</template>
<script>
import G6 from "@antv/g6";
export default {
  mounted() {
    this.$nextTick(() => {
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
      G6.registerBehavior("click-add-edge", {
        getEvents() {
          return {
            "node:click": "onClick",
            mousemove: "onMousemove",
            "edge:click": "onEdgeClick" // 点击空白处，取消边
          };
        },
        onClick(ev) {
          const node = ev.item;
          const graph = this.graph;
          const point = {
            x: ev.x,
            y: ev.y
          };
          const model = node.getModel();
          if (this.addingEdge && this.edge) {
            graph.updateItem(this.edge, {
              target: model.id
            });
            // graph.setItemState(this.edge, 'selected', true);
            this.edge = null;
            this.addingEdge = false;
          } else {
            this.edge = graph.addItem("edge", {
              source: model.id,
              target: point
            });
            this.addingEdge = true;
          }
        },
        onMousemove(ev) {
          const point = {
            x: ev.x,
            y: ev.y
          };
          if (this.addingEdge && this.edge) {
            this.graph.updateItem(this.edge, {
              target: point
            });
          }
        },
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

      const graph = new G6.Graph({
        container: "mountNode",
        width: 500,
        height: 500,
        modes: {
          default: ["click-add-edge", "click-select"]
        },
        // The node styles in different states
        nodeStateStyles: {
          // The node styles in selected state, corresponds to the built-in click-select behavior
          selected: {
            stroke: "#666",
            lineWidth: 2,
            fill: "steelblue"
          }
        }
      });

      graph.data(data);
      graph.render();
    });
  }
};
</script>