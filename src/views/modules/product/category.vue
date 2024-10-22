
<template>
  <div>
    <el-switch
      v-model="draggable"
      active-text="开启拖拽"
      inactive-text="关闭拖拽"
    >
    </el-switch>
    <el-button v-if="draggable" @click="updateSort" type="primary"
      >批量更新</el-button
    >
    <el-button type="danger" @click="batchDelete">批量删除</el-button>
    <el-tree
      :data="data"
      :props="defaultProps"
      :expand-on-click-node="false"
      show-checkbox
      node-key="catId"
      @node-drop="handleDrop"
      :default-expanded-keys="expandKeys"
      :allow-drop="allowDrop"
      :draggable="draggable"
      ref="tree"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="data.catLevel <= 2"
            type="text"
            size="mini"
            @click="() => append(data)"
          >
            添加
          </el-button>
          <el-button type="text" size="mini" @click="() => edit(data)">
            更新
          </el-button>
          <el-button
            v-if="data.children && data.children.length == 0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >
            删除
          </el-button>
        </span>
      </span>
    </el-tree>

    <!-- 添加 弹出框 -->
    <el-dialog
      :title="dialogType ? '新增' : '更新'"
      :visible.sync="dialogVisible"
      width="30%"
      :close-on-click-modal="false"
    >
      <el-form :model="categoryForm">
        <el-form-item label="类别名称">
          <el-input v-model="categoryForm.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="categoryForm.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input
            v-model="categoryForm.productUnit"
            autocomplete="off"
          ></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitForm">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
/* eslint-disable */
export default {
  data() {
    return {
      pCids: [], // 拖拽后的父节点数组
      draggable: false, // 默认拖拽功能是关闭的
      updateNodes: [], // 拖拽节点后，需要更新的节点的节点信息
      maxLevel: 0,
      dialogType: false, // true 添加  false 更新
      dialogVisible: false,
      categoryForm: {
        name: null,
        icon: null,
        productUnit: null,
        showStatus: 1,
        sort: 0,
        catId: null,
        catLevel: 1,
      },
      data: [],
      expandKeys: [],
      defaultProps: {
        children: "children",
        label: "name",
      },
    };
  },
  methods: {
    getCategory() {
      this.$http({
        url: this.$http.adornUrl("/product/category/listTree"),
        method: "get",
      }).then(({ data }) => {
        this.data = data.data;
      });
    },
    append(data) {
      this.dialogVisible = true; // 打开弹出窗口
      // console.log("添加", data);
      this.categoryForm.parentCid = data.catId; // 添加的类别对应的父菜单
      this.categoryForm.catLevel = data.catLevel + 1; // 设置添加类别的层级
      this.categoryForm.showStatus = 1; // 菜单的显示状态  1 显示  0 被删除
      this.categoryForm.sort = 0; // 排序 默认给 0

      // 重置更新的数据
      this.categoryForm.catId = null;
      this.categoryForm.name = "";
      this.categoryForm.icon = "";
      this.categoryForm.productUnit = "";
      // 更新状态
      this.dialogType = true;
    },
    batchDelete() {
      // 批量删除的方法
      let catIds = [];
      let checkedNodes = this.$refs.tree.getCheckedNodes(false, false);
      for (let i = 0; i < checkedNodes.length; i++) {
        catIds.push(checkedNodes[i].catId);
      }
      this.$confirm(`是否确认删除【${catIds}】记录?`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          // 把删除的请求提交到后台服务
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(catIds, false),
          }).then(({ data }) => {
            if (data && data.code === 0) {
              this.$message({
                message: "操作成功",
                type: "success",
              });
              // 重新加载所有的菜单数据
              this.getCategory();
            } else {
              this.$message.error(data.msg);
            }
          });
        })
        .catch(() => {
          this.$message({
            type: "info",
            message: "已取消删除",
          });
        });
    },
    // draggingNode 要拖拽的节点
    // dropNode 目标节点
    // type 参数有三种情况：'prev'、'inner' 和 'next'
    handleDrop(draggingNode, dropNode, type, event) {
      // 1. 拖拽节点的父节点需要修改
      let parentId = 0;
      // 找到拖拽节点对应的所有的兄弟节点
      let siblings = null;
      if (type == "inner") {
        parentId = dropNode.data.catId;
        siblings = dropNode.childNodes;
      } else {
        parentId =
          dropNode.parent.data.catId == undefined
            ? 0
            : dropNode.parent.data.catId;
        siblings = dropNode.parent.childNodes;
      }
      // 2. 拖拽后节点所在的新的兄弟节点间的排序问题
      for (let i = 0; i < siblings.length; i++) {
        if (siblings[i].data.catId == draggingNode.data.catId) {
          // 获取的就是拖拽的那个节点，那么我们需要更新parent_cid
          // 3. 拖拽后的节点及其节点的的 catLevel 更新问题
          let catLevel = draggingNode.level;
          if (siblings[i].level != catLevel) {
            // 拖拽后节点的层级发生了变化，
            catLevel = siblings[i].level;
            // 递归的方式来更新子节点的层级
            this.updateChildNodeLevel(siblings[i]);
          }
          this.updateNodes.push({
            catId: siblings[i].data.catId,
            sort: i,
            parentCid: parentId,
            catLevel: catLevel,
          });
        } else {
          this.updateNodes.push({ catId: siblings[i].data.catId, sort: i });
        }
      }
      // 我们需要将拖拽后的数据更新完整的更新到后端服务中
      // console.log("---->",this.updateNodes);
      // this.updateSort(parentId);
      this.pCids.push(parentId);
    },
    updateSort() {
      // 批量更新拖拽后的数据  层级+排序
      // 添加三级分类的类别信息
      this.$http({
        url: this.$http.adornUrl("/product/category/updateBatch"),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false),
      }).then(({ data }) => {
        if (data && data.code === 0) {
          this.$message({
            message: "数据拖拽成功",
            type: "success",
          });
          // 重新加载所有的菜单数据
          this.getCategory();
          // 设置默认展开的父节点信息
          this.expandKeys = this.pCids;
          // 重置存储拖拽数据的容器
          this.updateNodes = [];
          this.maxLevel = 0;
        } else {
          this.$message.error(data.msg);
        }
      });
    },
    updateChildNodeLevel(node) {
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var childNode = node.childNodes[i].data;
          this.updateNodes.push({
            catId: node.childNodes.catId,
            catLevel: node.childNodes[i].level,
          });
          // 如果还有子节点，同步的更新处理
          this.updateChildNodeLevel(node.childNodes[i]);
        }
      }
    },
    // draggingNode 要拖拽的节点
    // dropNode 目标节点
    // type 参数有三种情况：'prev'、'inner' 和 'next'
    allowDrop(draggingNode, dropNode, type) {
      // 判断拖拽的节点是否可以在该位置放置
      // 1.获取当前被拖拽的节点的最大的level
      this.countNodeLevel(draggingNode);
      let deep = Math.abs(this.maxLevel - draggingNode.level) + 1;

      if (type == "inner") {
        return deep + dropNode.level <= 3;
      }
      return deep + dropNode.parent.level <= 3;
    },
    countNodeLevel(node) {
      // 在查找拖拽节点的子节点的最大level首先做一个赋值
      this.maxLevel = node.data.catLevel;
      // 找到所有子节点，最大的level
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level;
          }
          this.countNodeLevel(node.childNodes[i]);
        }
      }
    },
    addDialog() {
      // 添加三级分类的类别信息
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.categoryForm, false),
      }).then(({ data }) => {
        if (data && data.code === 0) {
          this.$message({
            message: "数据添加成功",
            type: "success",
          });
          this.dialogVisible = false; // 关闭弹出窗口
          // 重新加载所有的菜单数据
          this.getCategory();
          // 设置默认展开的父节点信息
          this.expandKeys = [this.categoryForm.parentCid];
        } else {
          this.$message.error(data.msg);
        }
      });
    },
    editDialog() {
      // 更新三级分类的类别信息
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData(this.categoryForm, false),
      }).then(({ data }) => {
        if (data && data.code === 0) {
          this.$message({
            message: "数据更新成功",
            type: "success",
          });
          this.dialogVisible = false; // 关闭弹出窗口
          // 重新加载所有的菜单数据
          this.getCategory();
          // 设置默认展开的父节点信息
          this.expandKeys = [this.categoryForm.parentCid];
        } else {
          this.$message.error(data.msg);
        }
      });
    },
    edit(data) {
      this.dialogType = false;
      // 获取最新的数据回写
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "post",
        data: this.$http.adornData(this.categoryForm, false),
      }).then(({ data }) => {
        // 表单数据回写
        this.categoryForm.name = data.category.name;
        this.categoryForm.productUnit = data.category.productUnit;
        this.categoryForm.icon = data.category.icon;
        this.categoryForm.catLevel = data.category.catLevel;
        this.categoryForm.parentCid = data.category.parentCid;
        // 填充更新数据的id
        this.categoryForm.catId = data.category.catId;
        this.categoryForm.showStatus = 1;
        this.categoryForm.sort = 0;

        // 更新类别信息的方法
        this.dialogVisible = true; // 打开更新的窗口
      });
    },
    submitForm() {
      // 判断当前的操作是更新还是删除
      if (this.dialogType) {
        // 添加操作
        this.addDialog();
      } else {
        // 更新操作
        this.editDialog();
      }
    },
    remove(node, data) {
      this.$confirm(`是否确认删除【${data.name}】记录?`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          // 传递的数据
          let ids = [data.catId];
          // 把删除的请求提交到后台服务
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false),
          }).then(({ data }) => {
            if (data && data.code === 0) {
              this.$message({
                message: "操作成功",
                type: "success",
              });
              // 重新加载所有的菜单数据
              this.getCategory();
              // 设置默认展开的父节点信息
              this.expandKeys = [node.parent.data.catId];
            } else {
              this.$message.error(data.msg);
            }
          });
        })
        .catch(() => {
          this.$message({
            type: "info",
            message: "已取消删除",
          });
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