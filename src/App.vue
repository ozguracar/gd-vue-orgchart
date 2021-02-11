<template>
  <div>
    <OrgChart :data="data" :options="options"> </OrgChart>
  </div>
</template>
<script>
import OrgChart from "./components/orgchart";
import json from "./assets/data.json";
export default {
  components: {
    OrgChart,
  },
  data() {
    return {
      data: json,
      options: {
        node: {
          title: "fullName",
          content: "email",
        },
      },
    };
  },
};
</script>
