<template>
  <div>
    <el-switch
        v-model="draggable"
        active-text="开启拖拽"
        inactive-text="关闭拖拽">
    </el-switch>
    <el-button v-show="draggable" @click="batchSave">批量保存</el-button>
    <el-button type="danger" @click="batchDelete">批量删除</el-button>
    <el-tree :data="menus" show-checkbox node-key="catId" :props="defaultProps" :expand-on-click-node="false"
             :default-expanded-keys="expandedKey" :draggable="draggable" :allow-drop="allowDrop" @node-drop="handleDrop" ref="menuTree">
    <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
              v-if="node.level <= 2"
              type="text"
              size="mini"
              @click="() => append(data)">
            Append
          </el-button>
          <el-button
              v-if="node.level <= 2"
              type="text"
              size="mini"
              @click="() => edit(data)">
            Edit
          </el-button>
          <el-button
              v-if="node.childNodes.length === 0"
              type="text"
              size="mini"
              @click="() => remove(node, data)">
            Delete
          </el-button>
        </span>
      </span>
    </el-tree>

    <el-dialog
        :title="title"
        :visible.sync="dialogVisible"
        :close-on-click-modal="false"
        width="30%">
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
export default {
  data () {
    return {
      draggable: false,
      pCid: [],
      updateNodes: [],
      title: null,
      category: {
        catId: null,
        name: null,
        parentCid: null,
        catLevel: null,
        showStatus: null,
        sort: null,
        icon: null,
        productUnit: null,
        productCount: null
      },
      dialogVisible: false,
      menus: [],
      expandedKey: [],
      defaultProps: {
        children: 'children',
        label: 'name'
      }
    }
  },
  methods: {
    getMenus () {
      this.$http({
        url: this.$http.adornUrl('/product/category/list/tree'),
        method: 'get',
        params: this.$http.adornParams({})
      }).then(({data}) => {
        console.log('成功获取到菜单数据', data.data)
        this.menus = data.data
      })
    },
    append (data) {
      this.title = '添加分类'
      console.log('append', data)

      this.category = {
        catId: null,
        name: null,
        parentCid: data.catId,
        catLevel: data.catLevel * 1 + 1,
        showStatus: 1,
        sort: 0,
        icon: null,
        productUnit: null,
        productCount: 0
      }

      this.dialogVisible = true
    },
    edit (data) {
      this.title = '修改分类'
      console.log('edit', data)

      this.$http({
        url: this.$http.adornUrl('/product/category/info/' + data.catId),
        method: 'get',
        params: this.$http.adornParams({})
      }).then(({data}) => {
        this.category = data.category
      })

      this.dialogVisible = true
    },
    submitData () {
      console.log(this.category)

      if (this.category.catId == null) {
        this.$http({
          url: this.$http.adornUrl('/product/category/save'),
          method: 'post',
          data: this.$http.adornData(this.category, false)
        }).then(() => {
          this.$message.success('添加成功')
          // 刷新菜单
          this.getMenus()
          // 设置需要默认展开的菜单
          this.expandedKey = [this.category.parentCid]
          this.dialogVisible = false
        })
      } else {
        this.$http({
          url: this.$http.adornUrl('/product/category/update'),
          method: 'post',
          data: this.$http.adornData(this.category, false)
        }).then(() => {
          this.$message.success('修改成功')
          // 刷新菜单
          this.getMenus()
          // 设置需要默认展开的菜单
          this.expandedKey = [this.category.parentCid]
          this.dialogVisible = false
        })
      }
    },
    remove (node, data) {
      console.log('remove', node, data)

      this.$confirm(`是否删除【${data.name}】菜单?`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        const catIds = [data.catId]
        this.$http({
          url: this.$http.adornUrl('/product/category/delete'),
          method: 'post',
          data: this.$http.adornData(catIds, false)
        }).then(() => {
          this.$message({
            type: 'success',
            message: '菜单删除成功'
          })
          // 刷新菜单
          this.getMenus()
          // 设置需要默认展开的菜单
          this.expandedKey = [node.parent.data.catId]
        })
      }).catch(() => {
        this.$message({
          type: 'info',
          message: '已取消删除'
        })
      })
    },
    allowDrop (draggingNode, dropNode, type) {
      // 1、被拖动的当前节点及所在的父节点总层数不能大于3

      // 1）、被拖动的当前节点总层数
      console.log('allowDrop', draggingNode, dropNode, type)

      // 当前正在拖动的节点+父节点所在的深度不大于3即可
      const level = Math.abs(this.countNodeLevel(draggingNode) - draggingNode.level) + 1

      if (type === 'inner') {
        return level + dropNode.level <= 3
      }
      return level + dropNode.parent.level <= 3
    },
    countNodeLevel (node) {
      let maxLevel = node.level
      // 找到所有子节点，求出最大深度
      for (const childNode of node.childNodes) {
        maxLevel = Math.max(childNode.level, this.countNodeLevel(childNode))
      }
      return maxLevel
    },
    handleDrop (draggingNode, dropNode, dropType, ev) {
      console.log('handleDrop', draggingNode, dropNode, dropType)
      // 1、当前节点最新的父节点id
      let pCid = 0
      let siblings = []
      if (dropType === 'inner') {
        pCid = dropNode.data.catId
        siblings = dropNode.childNodes
      } else {
        pCid = dropNode.parent.data.catId === undefined ? 0 : dropNode.parent.data.catId
        siblings = dropNode.parent.childNodes
      }
      this.pCid.push(pCid)

      // 2、当前拖拽节点的最新顺序
      for (let i = 0; i < siblings.length; i++) {
        const sibling = siblings[i]
        // 如果遍历的是当前正在拖拽的节点
        if (sibling.data.catId === draggingNode.data.catId) {
          // 当前节点的层级发生变化
          let catLevel = draggingNode.data.level
          if (sibling.level !== catLevel) {
            catLevel = dropNode.level
            // 修改子节点的层级
            this.updateChildNodeLevel(sibling)
          }
          this.updateNodes.push({catId: sibling.data.catId, sort: i, parentCid: pCid, catLevel})
        } else {
          this.updateNodes.push({catId: sibling.data.catId, sort: i})
        }
      }

      // 3、当前拖拽节点的最新层级
      console.log(this.updateNodes)
    },
    updateChildNodeLevel (node) {
      for (let i = 0; i < node.childNodes.length; i++) {
        const cNode = node.childNodes[i].data
        this.updateNodes.push({
          catId: cNode.catId,
          catLevel: node.childNodes[i].level
        })
        this.updateChildNodeLevel(node.childNodes[i])
      }
    },
    batchSave () {
      this.$http({
        url: this.$http.adornUrl('/product/category/update/sort'),
        method: 'post',
        data: this.$http.adornData(this.updateNodes, false)
      }).then(({ data }) => {
        this.$message({
          type: 'success',
          message: '菜单顺序等修改成功'
        })
        // 刷新菜单
        this.getMenus()
        // 设置需要默认展开的菜单
        this.expandedKey = this.pCid
        this.updateNodes = []
      })
    },
    batchDelete () {
      const catIds = []
      const checkedNodes = this.$refs.menuTree.getCheckedNodes()
      console.log('被选中的元素', checkedNodes)
      for (const checkedNode of checkedNodes) {
        catIds.push(checkedNode.catId)
      }

      this.$http({
        url: this.$http.adornUrl('/product/category/delete'),
        method: 'post',
        data: this.$http.adornData(catIds, false)
      }).then(() => {
        this.$confirm(`是否批量删除【${catIds}】菜单?`, '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {
          this.$message({
            type: 'success',
            message: '菜单批量删除成功!'
          })
          this.getMenus()
        }).catch(() => {
          this.$message({
            type: 'info',
            message: '已取消删除'
          })
        })
      })
    }
  },
  created () {
    this.getMenus()
  }
}
</script>

<style scoped>

</style>
