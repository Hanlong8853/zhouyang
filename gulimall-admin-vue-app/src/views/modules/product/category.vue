<template>
  <div>
    <el-tree
      :data="menus"
      :props="defaultProps"
      :expand-on-click-node="false"
      show-checkbox
      node-key="catId"
      :default-expanded-keys="expandedkey"
      draggable
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="node.level <= 2"
            type="text"
            size="mini"
            @click="() => append(data)"
          >Append</el-button>
          <el-button type="text" size="mini" @click="edit(data)">Edit</el-button>
          <el-button
            v-if="node.childNodes == 0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >Delete</el-button>
        </span>
      </span>
    </el-tree>
 
    <el-dialog
      :title="title"
      :visible.sync="dialogVisible"
      width="30%"
      :close-on-click-modal="false"
    >
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input v-model="category.productUnit" autocomplete="off"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitData">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>
 
<script>
//这里可以导入其他文件（比如：组件，工具js，第三方插件js，json文件，图片文件等等）
//例如：import 《组件名称》 from '《组件路径》';
 
export default {
  //import引入的组件需要注入到对象中才能使用
  components: {},
  data() {
    return {
      updateNodes: [],
      maxLevel: 0,
      title: "",
      dialogType: "", // edit  add
      category: {
        name: "",
        parentCid: 0,
        catLevel: 1,
        showStatus: 1,
        sort: 0,
        catId: null,
        icon: "",
        productUnit: "",
      },
      dialogVisible: false,
      menus: [],
      expandedkey: [],
      defaultProps: {
        children: "children",
        label: "name",
      },
    };
  },
 
  //监听属性 类似于data概念
  computed: {},
  //监控data中的数据变化
  watch: {},
  //方法集合
  methods: {
    //获取菜单
    getMenus() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        console.log("成功获取到菜单数据...", data.data);
        this.menus = data.data;
      });
    },
    handleDrop(draggingNode, dropNode, dropType, ev) {
      // draggingNode-当前正在拖拽的节点  dropNode-进入到哪个节点   dropType-进入到节点的什么位置
      console.log("handleDrop: ", draggingNode, dropNode, dropType);
      // 1、当前节点最新父节点ID
      let pCid = 0;
      let siblings = null;
      if (dropType == "before" || dropType == "after") {
        pCid =
          dropNode.parent.data.catId == undefined
            ? 0
            : dropNode.parent.data.catId;
        siblings = dropNode.parent.childNodes;
      } else {
        pCid = dropNode.data.catId;
        siblings = dropNode.childNodes;
      }
      //2、当前拖拽节点的最新顺序
      for (let i = 0; i < siblings.length; i++) {
        if (siblings[i].data.catId == draggingNode.data.catId) {
          //如果遍历的是当前正在拖拽的节点
          let catLevel = draggingNode.level;
          if (siblings[i].level != draggingNode.level) {
            //当前节点的层级发生变化
            catLevel = siblings[i].level;
            //修改他子节点层级
            this.updateChildNodeLevel(siblings[i]);
          }
 
          this.updateNodes.push({
            catId: siblings[i].data.catId,
            sort: i,
            parentCid: pCid,
            catLevel: catLevel,
          });
        } else {
          this.updateNodes.push({ catId: siblings[i].data.catId, sort: i });
        }
      }
 
      //3、当前拖拽节点的最新层级
      console.log("updateNodes：", this.updateNodes);
 
      this.$http({
        url: this.$http.adornUrl("/product/category/update/sort"),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单顺序等修改成功",
          type: "success",
        });
        //刷新删除成功后的菜单
        this.getMenus();
        //设置需要默认展开的菜单
        this.expandedkey = [pCid];
        this.updateNodes = [];
        this.maxLevel = 0;
      });
    },
    updateChildNodeLevel(node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var cNode = node.childNodes[i].data;
          this.updateNodes.push({
            catId: cNode.catId,
            catLevel: node.childNodes[i].level,
          });
          this.updateChildNodeLevel(node.childNodes[i]);
        }
      }
    },
    allowDrop(draggingNode, dropNode, type) {
      // 1、被拖动的当前节点，以及所在的父节点，总层数不能大于三
 
      // 1）、被拖动当前节点总层数 draggingNode-当前节点  dropNode-哪个节点的这些位置  type-放到了哪个位置
      console.log("allowDrop：", draggingNode, dropNode, type);
 
      //找到所有子节点，求出最大深度
      this.countNodeLevel(draggingNode.data);
 
      //当前节点的深度 当前正在拖动的节点+父节点所在的深度，不大于3即可
      let deep = this.maxLevel - draggingNode.data.catLevel + 1;
 
      console.log(
        "深度：",
        deep,
        "当前正在拖动的节点最深节点_" + this.maxLevel,
        "父节点所在的深度_" + draggingNode.data.catLevel
      );
 
      if (type == "inner") {
        console.log("innder", deep, dropNode.levle);
        return deep + dropNode.level <= 3;
      } else {
        console.log("other", deep, dropNode.parent.level);
        return deep + dropNode.parent.level <= 3;
      }
    },
    countNodeLevel(node) {
      // 找到所有子节点，求出最大深度
      if (node.children != null && node.children.length > 0) {
        for (let i = 0; i < node.children.length; i++) {
          if (node.children[i].catLevel > this.maxLevel) {
            this.maxLevel = node.children[i].catLevel;
          }
          this.countNodeLevel(node.children[i]);
        }
      }
    },
    edit(data) {
      console.log("要修改的数据", data);
      this.dialogType = "edit";
      this.title = "修改分类";
      this.dialogVisible = true;
      //发送请求获取当前节点的最新数据
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
      }).then(({ data }) => {
        console.log("要返回的数据", data);
        //请求成功
        this.category.name = data.data.name;
        this.category.catId = data.data.catId;
        this.category.icon = data.data.icon;
        this.category.productUnit = data.data.productUnit;
        this.category.parentCid = data.data.parentCid;
        this.category.catLevel = data.data.catLevel;
        this.category.sort = data.data.sort;
        this.category.showStatus = data.data.showStatus;
      });
    },
    append(data) {
      console.log("append", data);
      this.dialogType = "add";
      this.title = "添加分类";
      this.dialogVisible = true;
      this.category.parentCid = data.catId;
      this.category.catLevel = data.catLevel * 1 + 1;
 
      this.category.catId = null;
      this.category.name = "";
      this.category.icon = "";
      this.category.productUnit = "";
      this.category.sort = 0;
      this.category.showStatus = 1;
    },
 
    submitData() {
      if (this.dialogType == "add") {
        this.addCategory();
      }
      if (this.dialogType == "edit") {
        this.editCategory();
      }
    },
 
    //修改三级分类的方法
    editCategory() {
      var { catId, name, icon, productUnit } = this.category;
 
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData({ catId, name, icon, productUnit }, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单修改成功",
          type: "success",
        });
        //关闭窗口
        this.dialogVisible = false;
        //刷新删除成功后的菜单
        this.getMenus();
        //设置需要默认展开的菜单
        this.expandedkey = [this.category.parentCid];
      });
    },
 
    //添加三级分类的方法
    addCategory() {
      console.log("提交的三级分类数据", this.category);
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单保存成功",
          type: "success",
        });
 
        //关闭窗口
        this.dialogVisible = false;
        //刷新删除成功后的菜单
        this.getMenus();
        //设置需要默认展开的菜单
        this.expandedkey = [this.category.parentCid];
      });
    },
 
    remove(node, data) {
      var ids = [data.catId];
 
      this.$confirm(`是否删除【${data.name}】菜单`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false),
          }).then(({ data }) => {
            //console.log("删除成功");
            this.$message({
              message: "删除成功",
              type: "success",
            });
            //刷新删除成功后的菜单
            this.getMenus();
            //设置需要默认展开的菜单
            this.expandedkey = [node.parent.data.catId];
          });
        })
        .catch(() => {});
 
      console.log("remove", node, data);
    },
  },
  //生命周期 - 创建完成（可以访问当前this实例）
  created() {
    this.getMenus();
  },
  //生命周期 - 挂载完成（可以访问DOM元素）
  mounted() {},
  beforeCreate() {}, //生命周期 - 创建之前
  beforeMount() {}, //生命周期 - 挂载之前
  beforeUpdate() {}, //生命周期 - 更新之前
  updated() {}, //生命周期 - 更新之后
  beforeDestroy() {}, //生命周期 - 销毁之前
  destroyed() {}, //生命周期 - 销毁完成
  activated() {}, //如果页面有keep-alive缓存功能，这个函数会触发
};
</script>
<style scoped>
</style>