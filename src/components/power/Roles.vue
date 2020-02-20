<template>
  <div>
    <!-- 面包屑导航区域 -->
    <el-breadcrumb separator-class="el-icon-arrow-right">
      <el-breadcrumb-item :to="{ path: '/home' }">首页</el-breadcrumb-item>
      <el-breadcrumb-item>权限管理</el-breadcrumb-item>
      <el-breadcrumb-item>角色列表</el-breadcrumb-item>
    </el-breadcrumb>

    <!-- 卡片视图 -->
    <el-card>
      <!-- 添加角色按钮区域 -->
      <el-row>
        <el-col>
          <el-button type="primary" @click="addRoleDialogVisible = true">添加角色</el-button>
        </el-col>
      </el-row>

      <!-- 角色列表区域 -->
      <el-table :data="rolelist" border stripe>
        <el-table-column type="expand">
          <template v-slot:default="scope">
            <el-row
              :class="['bdbottom',index1===0?'bdtop':'','vcenter']"
              v-for="(item1,index1) in scope.row.children"
              :key="item1.id"
            >
              <!-- 渲染一级权限 -->
              <el-col :span="5">
                <el-tag closable @close="deleteRightById(scope.row,item1.id)">{{ item1.authName }}</el-tag>
                <i class="el-icon-caret-right"></i>
              </el-col>
              <!-- 渲染二级和三级权限 -->
              <el-col :span="19">
                <el-row
                  :class="[index2===0?'':'bdtop','vcenter']"
                  v-for="(item2,index2) in item1.children"
                  :key="item2.id"
                >
                  <!-- 通过 for 循环嵌套渲染二级权限 -->
                  <el-col :span="6">
                    <el-tag
                      type="success"
                      closable
                      @close="deleteRightById(scope.row,item2.id)"
                    >{{ item2.authName }}</el-tag>
                    <i class="el-icon-caret-right"></i>
                  </el-col>
                  <el-col :span="18">
                    <el-tag
                      v-for="item3 in item2.children"
                      :key="item3.id"
                      type="warning"
                      closable
                      @close="deleteRightById(scope.row,item3.id)"
                    >{{ item3.authName }}</el-tag>
                  </el-col>
                </el-row>
              </el-col>
            </el-row>
          </template>
        </el-table-column>
        <!-- 索引列 -->
        <el-table-column type="index"></el-table-column>
        <el-table-column label="角色名称" prop="roleName"></el-table-column>
        <el-table-column label="角色描述" prop="roleDesc"></el-table-column>
        <el-table-column label="操作" width="300px">
          <template v-slot:default="scope">
            <el-button
              type="primary"
              icon="el-icon-edit"
              size="mini"
              @click="showEditDialog(scope.row.id)"
            >编辑</el-button>
            <el-button
              type="danger"
              icon="el-icon-delete"
              size="mini"
              @click="deleteRole(scope.row.id)"
            >删除</el-button>
            <el-button
              type="warning"
              icon="el-icon-setting"
              size="mini"
              @click="showSetRightDialog(scope.row)"
            >分配权限</el-button>
          </template>
        </el-table-column>
      </el-table>
    </el-card>

    <!-- 添加角色对话框 -->
    <el-dialog title="添加角色" :visible.sync="addRoleDialogVisible" width="50%" @close="addRoleClosed">
      <el-form
        :model="addRoleForm"
        :rules="addRoleFormRules"
        ref="addRuleFormRef"
        label-width="100px"
      >
        <el-form-item label="角色名称" prop="roleName">
          <el-input v-model.trim="addRoleForm.roleName"></el-input>
        </el-form-item>
        <el-form-item label="角色描述" prop="roleDesc">
          <el-input v-model.trim="addRoleForm.roleDesc"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="addRoleDialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="addRole">确 定</el-button>
      </span>
    </el-dialog>

    <!-- 编辑角色对话框 -->
    <el-dialog
      title="添加角色"
      :visible.sync="editRoleDialogVisible"
      width="50%"
      @close="editRoleFormClosed"
    >
      <el-form
        :model="editRoleForm"
        :rules="editRoleFormRules"
        ref="editRuleFormRef"
        label-width="100px"
      >
        <el-form-item label="角色名称" prop="roleName">
          <el-input v-model.trim="editRoleForm.roleName"></el-input>
        </el-form-item>
        <el-form-item label="角色描述" prop="roleDesc">
          <el-input v-model.trim="editRoleForm.roleDesc"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="editRoleDialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="editRoleInfo">确 定</el-button>
      </span>
    </el-dialog>

    <!-- 分配角色权限的对话框 -->
    <el-dialog
      title="分配权限"
      :visible.sync="setRightDialogVisible"
      width="50%"
      @close="setRightDialogColesd"
    >
      <!-- 树形控件 -->
      <el-tree
        :data="rightList"
        :props="treetProps"
        show-checkbox
        node-key="id"
        default-expand-all
        :default-checked-keys="defKeys"
        ref="treeRef"
      ></el-tree>
      <span slot="footer" class="dialog-footer">
        <el-button @click="setRightDialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="allotRights">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
export default {
  data() {
    return {
      // 所有角色列表数据
      rolelist: [],
      // 添加角色对象框的显示和隐藏
      addRoleDialogVisible: false,
      // 添加角色对话框表单对象
      addRoleForm: {
        roleName: '',
        roleDesc: ''
      },
      // 添加角色对话框的表单输入框验证规则
      addRoleFormRules: {
        roleName: [
          { required: true, message: '请输入角色名称', trigger: 'blur' },
          {
            min: 3,
            max: 10,
            message: '角色名称在 3 到 10 个字符之间',
            trigger: 'blur'
          }
        ],
        roleDesc: [
          //   { required: true, message: '请输入角色描述', trigger: 'blur' },
          {
            min: 5,
            max: 12,
            message: '角色描述在 5 到 12 个字符之间',
            trigger: 'blur'
          }
        ]
      },
      // 编辑角色对话框的显示和隐藏
      editRoleDialogVisible: false,
      // 编辑角色表单对象
      editRoleForm: {},
      // 编辑角色表单验证规则
      editRoleFormRules: {
        roleName: [
          { required: true, message: '请输入角色名称', trigger: 'blur' },
          {
            min: 3,
            max: 10,
            message: '角色名称在 3 到 10 个字符之间',
            trigger: 'blur'
          }
        ],
        roleDesc: [
          //   { required: true, message: '请输入角色描述', trigger: 'blur' },
          {
            min: 5,
            max: 12,
            message: '角色描述在 5 到 12 个字符之间',
            trigger: 'blur'
          }
        ]
      },
      // 控制分配权限对话框的显示
      setRightDialogVisible: false,
      // 所有权限数据
      rightList: [],
      // 树形控件的属性绑定对象
      treetProps: {
        children: 'children',
        label: 'authName'
      },
      // 默认选中的节点ID值数组
      defKeys: [],
      // 当前即将分配权限的角色ID
      roleId: []
    }
  },
  created() {
    this.getRightList()
  },
  methods: {
    // 获取所有的权限列表
    async getRightList() {
      const { data: res } = await this.$http.get('roles')
      if (res.meta.status !== 200) {
        return this.$message.error('获取角色列表失败')
      }
      this.rolelist = res.data
    },
    // 添加角色对话框关闭事件
    addRoleClosed() {
      this.$refs.addRuleFormRef.resetFields()
    },
    // 添加角色事件
    addRole() {
      this.$refs.addRuleFormRef.validate(async valid => {
        // console.log(valid)
        if (!valid) {
          return false
        }
        const { data: res } = await this.$http.post('roles', this.addRoleForm)
        console.log(res)
        if (res.meta.status !== 201) {
          return this.$message.error('添加角色失败')
        }
        this.$message.success('添加角色成功')
        this.addRoleDialogVisible = false
        this.getRightList()
      })
    },
    // 编辑角色对话框
    async showEditDialog(id) {
      const { data: res } = await this.$http.get(`roles/${id}`)
      if (res.meta.status !== 200) {
        return this.$message.error('查询角色失败')
      }
      this.editRoleForm = res.data
      this.editRoleDialogVisible = true
    },
    // 编辑角色对话框关闭事件
    editRoleFormClosed() {
      this.$refs.editRuleFormRef.resetFields()
    },
    // 修改角色信息并提交
    editRoleInfo() {
      this.$refs.editRuleFormRef.validate(async valid => {
        if (!valid) {
          return false
        }
        const { data: res } = await this.$http.put(
          `roles/${this.editRoleForm.roleId}`,
          this.editRoleForm
        )
        if (res.meta.status !== 200) {
          return this.$message.error('角色信息修改失败')
        }
        this.$message.success('角色信息修改成功')
        this.editRoleDialogVisible = false
        this.getRightList()
      })
    },
    // 删除角色
    async deleteRole(id) {
      const { data: res } = await this.$http.delete(`roles/${id}`)
      //   console.log(res)
      if (res.meta.status !== 200) {
        return this.$message.error('删除角色失败')
      }
      this.$message.success('删除角色成功')
      this.getRightList()
    },
    // 根据ID删除对应权限
    async deleteRightById(role, rightId) {
      // 弹框提示是否删除
      const confirmResult = await this.$confirm(
        '此操作将永久删除该权限, 是否继续?',
        '提示',
        {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }
      ).catch(err => err)
      if (confirmResult !== 'confirm') {
        return this.$message.info('删除该权限的操作已取消')
      }
      const { data: res } = await this.$http.delete(
        `roles/${role.id}/rights/${rightId}`
      )
      if (res.meta.status !== 200) {
        return this.$message.error('删除权限失败')
      }
      //   this.getRightList()
      role.children = res.data
    },
    // 展示分配权限的对话框
    async showSetRightDialog(role) {
      this.roleId = role.id
      // 获取所有权限的数据
      const { data: res } = await this.$http.get('rights/tree')
      if (res.meta.status !== 200) {
        return this.$message.error('获取权限数据失败')
      }
      // 把获取到的权限数据保存到data中
      this.rightList = res.data
      // 递归获取三级权限ID值
      this.getLeafKeys(role, this.defKeys)
      this.setRightDialogVisible = true
    },
    // 通过递归的形式，获取到所有角色三级权限的id，并保存到defKeys数组中
    getLeafKeys(node, arr) {
      // 如果当前node节点不包含children属性，则是三级节点
      if (!node.children) {
        return arr.push(node.id)
      }
      node.children.forEach(item => {
        this.getLeafKeys(item, arr)
      })
    },
    // 监听分配权限对话框的关闭事件
    setRightDialogColesd() {
      this.defKeys = []
    },
    // 点击为角色分配权限
    async allotRights() {
      const keys = [
        ...this.$refs.treeRef.getCheckedKeys(),
        ...this.$refs.treeRef.getHalfCheckedKeys()
      ]
      // console.log(keys)
      const idStr = keys.join(',')
      const { data: res } = await this.$http.post(
        `roles/${this.roleId}/rights`,
        {
          rids: idStr
        }
      )
      if (res.meta.status !== 200) {
        return this.$message.error('分配权限失败')
      }
      this.$message.success('分配权限成功')
      this.getRightList()
      this.setRightDialogVisible = false
    }
  }
}
</script>

<style lang="less" scoped>
.el-tag {
  margin: 7px;
}
.bdtop {
  border-top: 1px solid #eee;
}
.bdbottom {
  border-bottom: 1px solid #eee;
}
.vcenter {
  display: flex;
  align-items: center;
}
</style>
