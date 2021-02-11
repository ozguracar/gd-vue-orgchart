<template>
  <div ref="container" class="org-container" :class="opts.containerClass">
    <OcLoader v-if="spinner" :type="opts.spinnerType" />
    <div class="search-box">
      <input
        v-model="searchInput"
        :class="!opts.search.button ? 'center' : ''"
        type="text"
        :placeholder="(opts.search.button ? '  ' : '') + opts.search.text"
        @focusin="focusInput = true"
        @focusout="focusInput = false"
      />
      <i v-if="opts.search.button" class="ri-search-eye-line search-button"></i>
    </div>
    <div v-if="showSearchList" class="search-list">
      <ul @mouseover="hoverInput = true" @mouseleave="hoverInput = false">
        <li v-for="el in searchList" :key="el[opts.node.id]">
          <div
            class="search-list-left"
            :class="opts.search.listButton ? 'has-button' : ''"
          >
            <h6>{{ el[opts.node.title] }}</h6>
            <span>{{ el[opts.node.content] }}</span>
          </div>
          <div v-if="opts.search.listButton" class="search-list-right">
            <button @click="changeRoot(el)">
              {{ opts.search.listButtonText }}
            </button>
          </div>
        </li>
      </ul>
    </div>
    <div>
      <div ref="chart" class="orgchart" :class="opts.chartClass">
        <TableComponent ref="table" :data="ds" :fire="fire" />
      </div>
    </div>
  </div>
</template>
<script>
import panzoom from "panzoom";
import Vue from "vue";
import TableComponent from "./table";
import NodeComponent from "./node";
import "../assets/fonts/remixicon.css";
import "../assets/style.css";
import LoaderComponent from "./Loader.vue";
export default {
  components: { TableComponent },
  props: {
    data: {
      type: [Object, Array],
      default() {
        return [];
      },
    },
    nodeComponent: {
      type: Object,
      default() {
        return NodeComponent;
      },
    },
    spinnerComponent: {
      type: Object,
      default() {
        return LoaderComponent;
      },
    },
    options: {
      type: Object,
      default() {
        return {};
      },
    },
  },
  data() {
    return {
      spinner: false,
      ds: {},
      virtualDS: {},
      container: null,
      defaultOptions: {
        mode: "static", //both,ajax,static
        node: {
          id: "id",
          title: "title",
          titleIcon: true,
          content: "content",
          contentIcon: false,
          nodeIcon: false,
          spinnerType: "default",
        },
        toggleSiblingsResp: false,
        chartClass: "",
        nodeClass: "",
        containerClass: "",
        containerHeight: "",
        autoMoveCenter: true,
        panzoom: true,
        draggable: true,
        dropCriteria: null,
        onDrop: null,
        autoScroll: true,
        autoScrollSpeed: 5,
        autoScrollMargin: 250,
        spinnerType: "default",
        nodeSpinnerType: "facebook",
        search: {
          text: "Search",
          button: false,
          listButtonText: "View Tree",
          listButton: true,
        },
      },
      search: null,
      searchList: [],
      focusInput: false,
      hoverInput: false,
    };
  },
  computed: {
    opts() {
      return {
        ...this.defaultOptions,
        ...this.options,
        node: { ...this.defaultOptions.node, ...this.options.node },
        search: { ...this.defaultOptions.search, ...this.options.search },
      };
    },
    searchInput: {
      get() {
        return this.search;
      },
      set(val) {
        this.search = val;
        this.searchList = [];
        if (val) {
          this.searchInChild(this.virtualDS);
        }
      },
    },
    showSearchList() {
      if (this.hoverInput || this.focusInput) {
        return true;
      }
      return false;
    },
  },
  watch: {
    data: {
      deep: true,
      immediate: true,
      handler(val) {
        this.ds = this.virtualDS = this.setDS(val);
      },
    },
    "opts.containerHeight": {
      deep: true,
      immediate: true,
      handler(val) {
        if (this.container && this.container.style) {
          this.container.style.height = val;
        }
      },
    },
  },
  created() {
    // console.log(this.$slots.default);
    Vue.component("OcNode", this.nodeComponent);
    Vue.component("OcLoader", this.spinnerComponent);
    const rootUser = {
      id: "root",
      hasParent: false,
      hasSiblings: false,
      hasLeftSiblings: false,
      hasRightSiblings: false,
      hasChildren: true,
    };
    if (Array.isArray(this.data)) {
      // this.data.forEach((element) => {
      //   element.hasParent = true;
      // });
      rootUser[this.opts.node.id] = "$oc-root-user";
      rootUser[this.opts.node.title] = "ROOT";
      rootUser[this.opts.node.content] = "";

      rootUser.children = this.data;
      this.ds = this.virtualDS = this.setDS(rootUser);
    } else {
      this.ds = this.virtualDS = this.setDS(this.data);
    }
  },
  mounted() {
    this.container = this.$refs.container;
    const chart = this.$refs.chart;
    this.chart = chart;
    this.container.style.height = this.opts.containerHeight;
    if (this.opts.panzoom) {
      this.panzoom = panzoom(chart, {
        maxZoom: 3,
        minZoom: 0.01,
        bounds: true,
        boundsPadding: 0.1,
      });
      this.transform = this.panzoom.getTransform();
      this.moveTo(true);
    }
  },
  methods: {
    searchInChild(data) {
      if (
        data[this.opts.node.id]
          .toLowerCase()
          .includes(this.search.toLowerCase()) ||
        data[this.opts.node.title]
          .toLowerCase()
          .includes(this.search.toLowerCase()) ||
        data[this.opts.node.content]
          .toLowerCase()
          .includes(this.search.toLowerCase())
      ) {
        this.searchList.push(data);
      }
      data.children.forEach((element) => {
        this.searchInChild(element);
      });
    },
    clicked(type, payload) {
      this.$emit(type + "Click", payload);
    },
    startSpinner() {
      this.spinner = true;
    },
    finishSpinner() {
      this.spinner = false;
    },
    getHierarchy() {
      const hierarvhy = JSON.stringify({
        ...this.ds,
        $oc_parent: undefined,
        $oc_isRoot: undefined,
        $oc_visible: undefined,
      });
      this.cleanData(hierarvhy);
      return hierarvhy;
    },
    cleanDataForHierarvhy(data) {
      delete data.$oc_parent;
      delete data.$oc_isRoot;
      delete data.$oc_visible;
      data.children.forEach((child) => {
        this.cleanData(child);
      });
    },
    _closest(el, fn) {
      return (
        el &&
        (fn(el) && el !== this.chart ? el : this._closest(el.parentNode, fn))
      );
    },
    onDragStart(data, event) {
      this.panzoom.pause();
      const nodeDiv = event.target;
      const opts = this.opts;
      const isFirefox = /firefox/.test(
        window.navigator.userAgent.toLowerCase()
      );

      if (isFirefox) {
        event.dataTransfer.setData("text/html", "hack for firefox");
      }
      if (this.chart.style.transform) {
        let ghostNode, nodeCover;

        if (!document.querySelector(".ghost-node")) {
          ghostNode = document.createElementNS(
            "http://www.w3.org/2000/svg",
            "svg"
          );
          ghostNode.classList.add("ghost-node");
          nodeCover = document.createElementNS(
            "http://www.w3.org/2000/svg",
            "rect"
          );
          ghostNode.appendChild(nodeCover);
          this.chart.appendChild(ghostNode);
        } else {
          ghostNode = this.chart.querySelector(":scope > .ghost-node");
          nodeCover = ghostNode.children[0];
        }
        const transValues = this.chart.style.transform.split(",");
        const scale = Math.abs(
          window.parseFloat(
            transValues[0].slice(transValues[0].indexOf("(") + 1)
          )
        );

        ghostNode.setAttribute("width", nodeDiv.offsetWidth);
        ghostNode.setAttribute("height", nodeDiv.offsetHeight);
        nodeCover.setAttribute("x", 5 * scale);
        nodeCover.setAttribute("y", 5 * scale);
        nodeCover.setAttribute("width", 120 * scale);
        nodeCover.setAttribute("height", 40 * scale);
        nodeCover.setAttribute("rx", 4 * scale);
        nodeCover.setAttribute("ry", 4 * scale);
        nodeCover.setAttribute("stroke-width", 1 * scale);
        let xOffset = event.offsetX * scale;
        let yOffset = event.offsetY * scale;

        if (isFirefox) {
          // hack for old version of Firefox(< 48.0)
          const ghostNodeWrapper = document.createElement("img");

          ghostNodeWrapper.src =
            "data:image/svg+xml;utf8," +
            new XMLSerializer().serializeToString(ghostNode);
          event.dataTransfer.setDragImage(ghostNodeWrapper, xOffset, yOffset);
          nodeCover.setAttribute("fill", "rgb(255, 255, 255)");
          nodeCover.setAttribute("stroke", "rgb(191, 0, 0)");
        } else {
          event.dataTransfer.setDragImage(ghostNode, xOffset, yOffset);
        }
      }
      const dragged = event.target;
      const dragZone = this._closest(dragged, (el) => {
        return el.classList && el.classList.contains("nodes");
      }).parentNode.children[0].querySelector(".node");
      const dragHier = Array.from(
        this._closest(dragged, (el) => {
          return el.nodeName === "TABLE";
        }).querySelectorAll(".node")
      );
      this.dragged = { node: dragged, data };
      Array.from(this.chart.querySelectorAll(".node")).forEach((node) => {
        if (!dragHier.includes(node)) {
          if (opts.dropCriteria) {
            if (
              opts.dropCriteria(
                { node: dragged, data },
                { node: dragZone, data: data.$oc_parent },
                { node, data: this.findById(node.dataset.id, this.ds) }
              )
            ) {
              node.classList.add("allowedDrop");
            }
          } else {
            if (node.dataset.id !== data.$oc_parent[opts.node.id]) {
              node.classList.add("allowedDrop");
            }
          }
        }
      });
    },
    containerCalc() {
      if (this.$refs.container) {
        return {
          offsetLeft:
            this.$refs.container.offsetLeft + this.opts.autoScrollMargin,
          offsetTop:
            this.$refs.container.offsetTop + this.opts.autoScrollMargin,
          offsetHeight:
            this.$refs.container.offsetTop +
            this.$refs.container.offsetHeight -
            this.opts.autoScrollMargin,
          offsetWidth:
            this.$refs.container.offsetLeft +
            this.$refs.container.offsetWidth -
            this.opts.autoScrollMargin,
        };
      }
      return {};
    },
    onDrag(data, event) {
      this.checkMove = false;
      this.move = {
        x: this.transform.x,
        y: this.transform.y,
      };
      if (event.clientX) {
        if (this.containerCalc().offsetWidth < event.clientX) {
          this.move.x =
            this.transform.x -
            (event.clientX - this.containerCalc().offsetWidth) *
              this.opts.autoScrollSpeed;
          this.checkMove = true;
        } else if (event.clientX < this.containerCalc().offsetLeft) {
          this.move.x =
            this.transform.x +
            (this.containerCalc().offsetLeft - event.clientX) *
              this.opts.autoScrollSpeed;
          this.checkMove = true;
        }
      }
      if (event.clientY) {
        if (this.containerCalc().offsetHeight < event.clientY) {
          this.move.y =
            this.transform.y -
            (event.clientY - this.containerCalc().offsetHeight) *
              this.opts.autoScrollSpeed;
          this.checkMove = true;
        } else if (event.clientY < this.containerCalc().offsetTop) {
          this.move.y =
            this.transform.y +
            (this.containerCalc().offsetTop - event.clientY) *
              this.opts.autoScrollSpeed;
          this.checkMove = true;
        }
      }
      if (this.checkMove) {
        this.panzoom.smoothMoveTo(this.move.x, this.move.y);
      }
    },
    findById(id, ds, obj) {
      if (ds[this.opts.node.id] === id) {
        return ds;
      }
      let object = {
        data: null,
      };
      if (obj) {
        object = obj;
      }
      ds.children.forEach((child) => {
        if (object.data) {
          return;
        }
        if (ds[this.opts.node.id] === id) {
          object.data = ds;
        } else {
          object.data = this.findById(id, child, object);
        }
      });
      return object.data;
    },
    onDragOver(data, event) {
      event.preventDefault();
      const dropZone = event.currentTarget;

      if (!dropZone.classList.contains("allowedDrop")) {
        event.dataTransfer.dropEffect = "none";
      }
    },
    onDragEnd() {
      Array.from(this.chart.querySelectorAll(".allowedDrop")).forEach((el) => {
        el.classList.remove("allowedDrop");
      });
      if (!this.spinner) {
        this.panzoom.resume();
      }
    },
    onDrop(data, event) {
      const dropZone = event.currentTarget;
      const copyData = JSON.parse(
        JSON.stringify({
          ...data,
          $oc_parent: undefined,
          $oc_isRoot: undefined,
          $oc_visible: undefined,
          children: undefined,
        })
      );
      const copyDragged = JSON.parse(
        JSON.stringify({
          ...this.dragged.data,
          $oc_parent: undefined,
          $oc_isRoot: undefined,
          $oc_visible: undefined,
          children: undefined,
        })
      );
      const node = document.querySelector(".node.active");
      if (node) {
        node.classList.remove("active");
      }
      if (this.opts.onDrop) {
        this.startSpinner();
        this.panzoom.pause();
        this.opts
          .onDrop(copyData, copyDragged)
          .then(() => {
            const index = this.dragged.data.$oc_parent.children.findIndex(
              (x) =>
                x[this.opts.node.id] === this.dragged.data[this.opts.node.id]
            );
            if (index >= 0) {
              this.dragged.data.$oc_parent.children.splice(index, 1);
              if (!this.dragged.data.$oc_parent.children.length) {
                this.dragged.data.$oc_parent.hasChildren = false;
              }
              this.dragged.data.$oc_parent = data;
              if (data.children.length) {
                this.dragged.data.hasSiblings = true;
                this.dragged.data.hasLeftSiblings = true;
              }
              data.children.push(this.dragged.data);
            }
            dropZone.classList.add("active");
          })
          .catch(() => {})
          .finally(() => {
            this.finishSpinner();
            this.panzoom.resume();
            this.resetPan(dropZone);
          });
      } else {
        const index = this.dragged.data.$oc_parent.children.findIndex(
          (x) => x[this.opts.node.id] === this.dragged.data[this.opts.node.id]
        );
        if (index >= 0) {
          this.dragged.data.$oc_parent.children.splice(index, 1);
          if (!this.dragged.data.$oc_parent.children.length) {
            this.dragged.data.$oc_parent.hasChildren = false;
          }
          this.dragged.data.$oc_parent = data;
          if (data.children.length) {
            this.dragged.data.hasSiblings = true;
            this.dragged.data.hasLeftSiblings = true;
          }
          data.children.push(this.dragged.data);
        }
        dropZone.classList.add("active");
        this.resetPan(dropZone);
      }
    },
    moveTo(smooth) {
      if (!this.opts.autoMoveCenter || !this.opts.panzoom) {
        return;
      }
      const transform = this.panzoom.getTransform();
      const interval = setInterval(() => {
        const node = this.$refs.table.$refs.node;
        if (this.$refs.container.clientWidth && node) {
          clearInterval(interval);
          if (smooth) {
            this.panzoom.smoothMoveTo(
              this.$refs.container.clientWidth / 2 -
                node.offsetLeft * transform.scale -
                (node.offsetWidth / 2) * transform.scale,
              0
            );
          } else {
            this.panzoom.moveTo(
              this.$refs.container.clientWidth / 2 -
                node.offsetLeft * transform.scale -
                (node.offsetWidth / 2) * transform.scale,
              0
            );
          }
        }
      }, 250);
    },
    openParent(data) {
      if (!data.$oc_parent) {
        return;
      }
      this.ds = data.$oc_parent;

      this.changeRoot(this.ds);
      this.ds.children.forEach((element) => {
        if (element[this.opts.node.id] !== data[this.opts.node.id]) {
          this.hideChildren(element);
        }
      });
      this.$nextTick(() => {
        this.moveTo();
      });
    },
    changeRoot(data) {
      this.resetRoot();
      this.ds = data;
      this.ds.$oc_isRoot = true;
      this.$nextTick(() => {
        this.moveTo();
      });
    },
    changeRootProp(ds) {
      ds.$oc_isRoot = false;
      ds.children.forEach((child) => {
        this.changeRootProp(child);
      });
    },
    resetRoot() {
      const ds = this.virtualDS;
      ds.$oc_isRoot = false;
      ds.children.forEach((child) => {
        this.changeRootProp(child);
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
    resetPan(node) {
      if (!this.opts.autoMoveCenter || !this.opts.panzoom) {
        return;
      }
      const transform = this.panzoom.getTransform();
      const clientX =
        this.$refs.container.clientWidth / 2 -
        node.offsetLeft * transform.scale -
        (node.offsetWidth / 2) * transform.scale;
      const clientY = (node.clientHeight - node.offsetTop) * transform.scale;
      this.panzoom.moveTo(clientX, clientY);
    },
    fire(fn) {
      return this[fn];
    },
    addProp(ds) {
      ds.$oc_visible = true;
      ds.$oc_isRoot = false;
      if (ds.children && ds.children.length) {
        ds.children.forEach((child) => {
          child.$oc_parent = ds;
          this.addProp(child);
        });
      } else {
        ds.children = [];
      }
    },
    setDS(data) {
      const ds = JSON.parse(JSON.stringify(data));
      ds.$oc_visible = true;
      ds.$oc_isRoot = true;
      if (ds.children && ds.children.length) {
        ds.children.forEach((child) => {
          child.$oc_parent = ds;
          this.addProp(child);
        });
      } else {
        ds.children = [];
      }
      return ds;
    },
  },
};
</script>
<style>
.search-box {
  width: 100%;
  max-width: 500px;
  margin: 50px auto;
  position: relative;
  box-sizing: border-box;
  font-family: "Poppins", sans-serif;
  z-index: 6;
}
.search-box i.search-button {
  position: absolute;
  right: 10px;
  top: calc(50% - 10px);
  font-size: 20px;
  color: #707070;
  cursor: pointer;
}
.search-box input {
  width: 100%;
  height: 60px;
  border: none;
  border-bottom: 2px solid #e8ebf0;
  line-height: 60px;
  font-weight: 500;
  color: #333;
  outline: none;
  background: transparent;
  font-size: 10px;
  font-size: 20px;
  font-family: "Poppins", sans-serif;
  padding: unset;
}
.search-box input.center {
  text-align: center;
}
.search-box input:focus {
  border-color: #eee;
  background: #fff;
}
.search-list {
  width: 100%;
  position: absolute;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 6;
  left: 0;
  margin-top: -65px;
  font-family: "Poppins", sans-serif;
}

.search-list ul {
  width: 100%;
  max-width: 500px;
  padding: 0 15px;
  max-height: 250px;
  overflow: auto;
}
.search-list li {
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 100%;
  padding: 5px 10px;
  background: #fff;
  border-bottom: 1px solid #e4f1fe;
  font-family: "Poppins", sans-serif;
  box-sizing: border-box;
}
.search-list li:last-child {
  border-bottom: none;
}
.search-list li .search-list-left {
  width: 100%;
  line-height: 18px;
}
.search-list li .search-list-left.has-button {
  width: 80%;
}
.search-list li .search-list-left h6 {
  margin: 0;
  font-size: 14px;
  color: #00c9a7;
  font-weight: 500;
  font-family: "Poppins", sans-serif;
}
.search-list li .search-list-left span {
  font-size: 13px;
  color: #707070;
  font-weight: 400;
}
.search-list li .search-list-right {
  width: 20%;
}
.search-list li .search-list-right button {
  outline: none;
  background: #00c9a7;
  color: #fff;
  border: none;
  font-family: "Poppins", sans-serif;
  font-size: 12px;
  font-weight: 500;
  padding: 7px 15px;
  cursor: pointer;
}
</style>