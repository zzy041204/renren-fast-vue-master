<template>
  <div>
    <el-tree
      :data="data"
      :props="defaultProps" 
      node-key="catId"
      @node-click="nodeclick"
    >
    </el-tree>
  </div>
</template>

<script>
/* eslint-disable */
export default {
  data() {
    return {
      data:[],
      defaultProps: {
        children: "children",
        label: "name",
      },
    };
  },
  methods: {
    nodeclick(data,node,component){
        //console.log("分类节点被点击了",data,node,component)
        this.$emit("show",data,node,component);
    },
    getCategory() {
      this.$http({
        url: this.$http.adornUrl("/product/category/listTree"),
        method: "get",
      }).then(({ data }) => {
        this.data = data.data;
      });
    },
  },
  created() {
    this.getCategory();
  },
};
</script>

<style>
</style>