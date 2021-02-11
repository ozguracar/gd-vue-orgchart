<template>
  <div @mouseover="hover = true" @mouseleave="hover = false">
    <div
      class="title"
      :style="hover && options.node.titleIcon ? 'text-align: left' : ''"
    >
      <span>
        {{ data[options.node.title] }}
        <i
          v-if="hover && options.node.titleIcon"
          @click="clicked('titleIcon', $event)"
          class="ri-share-box-line chart-profil-link"
        ></i>
      </span>
    </div>
    <div
      class="content"
      :style="hover && options.node.contentIcon ? 'text-align: left' : ''"
    >
      <span>
        {{ data[options.node.content] }}
        <i
          v-if="hover && options.node.contentIcon"
          @click="clicked('contentIcon', $event)"
          class="ri-share-box-line chart-profil-link"
        ></i>
      </span>
    </div>
  </div>
</template>
<script>
export default {
  props: {
    data: {
      type: Object,
      required: true,
    },
    options: {
      type: Object,
      required: true,
    },
  },
  data() {
    return {
      hover: false,
    };
  },
  methods: {
    clicked(type, event) {
      this.$emit("clicked", { type, event });
    },
  },
};
</script>
