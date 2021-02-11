<template>
  <table>
    <tr>
      <td :colspan="allVisibleChildren ? allVisibleChildren * 2 : ''">
        <div
          @click="clicked('node', $event)"
          ref="node"
          class="node"
          :data-id="data[opts.node.id]"
          :class="opts.nodeClass"
          :draggable="!isRoot"
          @dragstart="fire('onDragStart')(data, $event)"
          @dragend="fire('onDragEnd')(data, $event)"
          @dragover="fire('onDragOver')(data, $event)"
          @drop="fire('onDrop')(data, $event)"
          @drag="fire('onDrag')(data, $event)"
        >
          <OcLoader v-if="spinner" :type="opts.node.spinnerType" />
          <i
            v-if="opts.node.nodeIcon"
            @click="clicked('nodeIcon', $event)"
            class="ri-share-box-line open-profile-icon"
          ></i>
          <OcNode
            :data="data"
            :options="opts"
            @clicked="clicked($event.type, $event.event)"
          />
          <i
            v-if="!spinner && data.hasParent"
            class="edge verticalEdge topEdge"
            :class="!isRoot ? 'ri-arrow-down-s-fill' : 'ri-arrow-up-s-fill'"
            @click="changeRoot"
          ></i
          ><i
            v-if="!spinner && data.hasChildren"
            class="edge verticalEdge bottomEdge"
            :class="
              getVisibleStatus(data.children)
                ? 'ri-arrow-up-s-fill'
                : 'ri-arrow-down-s-fill'
            "
            @click="toggleChildren"
          ></i>
          <i
            v-if="
              !spinner &&
              (opts.toggleSiblingsResp
                ? allSiblings.length
                : leftSiblings.length)
            "
            class="edge horizontalEdge leftEdge"
            :class="
              getVisibleStatus(
                opts.toggleSiblingsResp ? allSiblings : leftSiblings
              )
                ? 'ri-arrow-right-s-fill'
                : 'ri-arrow-left-s-fill'
            "
            @click="toggleSiblings('left')"
          ></i
          ><i
            v-if="
              !spinner &&
              (opts.toggleSiblingsResp
                ? allSiblings.length
                : rightSiblings.length)
            "
            class="edge horizontalEdge rightEdge"
            :class="
              getVisibleStatus(
                opts.toggleSiblingsResp ? allSiblings : rightSiblings
              )
                ? 'ri-arrow-left-s-fill'
                : 'ri-arrow-right-s-fill'
            "
            @click="toggleSiblings('right')"
          ></i>
        </div>
      </td>
    </tr>
    <tr v-if="allVisibleChildren" class="lines">
      <td :colspan="allVisibleChildren ? allVisibleChildren * 2 : ''">
        <div class="downLine"></div>
      </td>
    </tr>
    <tr v-if="allVisibleChildren" class="lines">
      <td
        v-for="n in allVisibleChildren * 2"
        :key="n"
        :class="getLineClass(n)"
      ></td>
    </tr>
    <tr v-if="allVisibleChildren" class="nodes">
      <td
        v-for="child in data.children.filter((x) => x.$oc_visible)"
        :key="child[opts.node.id]"
        colspan="2"
      >
        <TableComponent :data="child" :fire="fire" />
      </td>
    </tr>
  </table>
</template>
<script>
export default {
  name: "Table",
  components: { TableComponent: () => import("./table.vue") },
  props: {
    data: {
      type: Object,
      required: true,
    },
    fire: {
      type: Function,
      required: true,
    },
  },
  data() {
    return {
      spinner: false,
    };
  },
  computed: {
    opts() {
      return this.fire("opts");
    },
    isRoot() {
      return this.data.$oc_isRoot;
    },
    parent() {
      return this.data.$oc_parent;
    },
    index() {
      if (!this.parent) {
        return -1;
      }
      return this.parent.children.findIndex(
        (x) => x[this.opts.node.id] === this.data[this.opts.node.id]
      );
    },
    leftSiblings() {
      if (!this.parent) {
        return [];
      }
      return this.parent.children.filter((x, i) => i < this.index);
    },
    rightSiblings() {
      if (!this.parent) {
        return [];
      }
      return this.parent.children.filter((x, i) => i > this.index);
    },
    allSiblings() {
      if (!this.parent) {
        return [];
      }
      return this.parent.children.filter((x, i) => i !== this.index);
    },
    allVisibleChildren() {
      return this.data.children.filter((x) => x.$oc_visible).length;
    },
  },
  methods: {
    clicked(type, event) {
      if (type === "node") {
        const node = document.querySelector(".node.active");
        if (node) {
          node.classList.remove("active");
        }
        if (this.$refs.node) {
          this.$refs.node.classList.add("active");
        }
      }
      this.fire("clicked")(type, { event: event, data: this.data });
    },
    getLineClass(n) {
      let line;
      if (n % 2) {
        line = "rightLine";
      } else {
        line = "leftLine ";
      }
      if (n !== 1 && n !== this.allVisibleChildren * 2) {
        line += " topLine";
      }
      return line;
    },
    getVisibleStatus(list) {
      return list.filter((x) => x.$oc_visible).length;
    },
    changeRoot() {
      if (this.isRoot) {
        this.fire("openParent")(this.data);
        return;
      }
      this.fire("changeRoot")(this.data);
    },
    toggleSiblings(direction) {
      if (this.isRoot) {
        this.fire("openParent")(this.data);
      }
      if (this.opts.toggleSiblingsResp) {
        direction = "all";
      }
      if (this.index >= 0) {
        let list;
        switch (direction) {
          case "left":
            list = this.leftSiblings;
            break;
          case "right":
            list = this.rightSiblings;
            break;
          default:
            list = this.allSiblings;
            break;
        }
        if (list.filter((x) => x.$oc_visible).length) {
          list.forEach((element) => {
            this.hideChildren(element);
          });
        } else {
          switch (direction) {
            case "left":
              list[list.length - 1].$oc_visible = true;
              break;
            case "right":
              list[0].$oc_visible = true;
              break;
            default:
              list.forEach((element) => {
                element.$oc_visible = true;
              });
              break;
          }
        }
      }
      this.$nextTick(() => {
        this.fire("resetPan")(this.$refs.node);
      });
    },
    hideChildren(data) {
      data.$oc_visible = false;
      if (data.children.length) {
        data.children.forEach((child) => {
          this.hideChildren(child);
        });
      }
    },
    toggleChildren() {
      if (!this.allVisibleChildren) {
        if (this.data.children.length) {
          this.data.children.forEach((child) => {
            child.$oc_visible = true;
          });
        }
      } else {
        this.data.children.forEach((child) => {
          this.hideChildren(child);
        });
      }
      this.$nextTick(() => {
        this.fire("resetPan")(this.$refs.node);
      });
    },
  },
};
</script>
