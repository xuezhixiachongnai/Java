# 权限系统

现代权限系统主要有两种主流模型：**RBAC**、**ABAC**

### RBAC

RBAC 是最常见、最广泛使用的权限模型。核心思想是：

用户不直接与权限绑定，而是通过 **角色（Role）** 间接获取权限。

**关系结构：**

```
用户（User） → 角色（Role） → 权限（Permission）
```

关键实体类

| 实体       | 描述                                         |
| ---------- | -------------------------------------------- |
| User       | 系统中的用户                                 |
| Role       | 用户的身份或职位（如管理员、客服、普通用户） |
| Permission | 具体的操作权限（如查看、编辑、删除）         |

它的优点是简单清晰，易于实现；管理方便，适合组织层级清晰的系统；可扩展，例如支持角色继承。

但是它对复杂的动态场景不灵活；当权限规则随业务动态变化（如时间、环境、数据属性）时，维护困难。

接下来我们来看一个**具体设计**

表设计

```sql
--- 用户表
CREATE TABLE user (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  username VARCHAR(50) NOT NULL,
  password VARCHAR(100) NOT NULL,
  status TINYINT DEFAULT 1 COMMENT '1启用 0禁用',
  create_time DATETIME,
  update_time DATETIME
);

--- 角色表
CREATE TABLE role (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  role_name VARCHAR(50) NOT NULL,
  code VARCHAR(50) UNIQUE NOT NULL COMMENT '角色标识，比如 ADMIN / USER',
  description VARCHAR(255)
);

--- 菜单资源表
CREATE TABLE menu (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  parent_id BIGINT DEFAULT 0 COMMENT '父菜单ID',
  name VARCHAR(50) NOT NULL COMMENT '菜单名称',
  type TINYINT NOT NULL COMMENT '1目录 2菜单 3按钮(API)',
  path VARCHAR(100) COMMENT '路由path或API路径',
  component VARCHAR(100) COMMENT '前端组件路径',
  perms VARCHAR(100) COMMENT '权限标识（如 sys:user:list ）',
  icon VARCHAR(50),
  sort INT DEFAULT 0,
  visible TINYINT DEFAULT 1 COMMENT '是否显示 1是 0否'
);

--- 权限表 一般被菜单表的 perms 字段包含
CREATE TABLE permission (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(50),
  code VARCHAR(100) UNIQUE NOT NULL COMMENT '权限编码，如 sys:user:add',
  description VARCHAR(255)
);

--- 用户角色关联表
CREATE TABLE user_role (
  user_id BIGINT NOT NULL,
  role_id BIGINT NOT NULL,
  PRIMARY KEY (user_id, role_id)
);

---角色菜单关联表
CREATE TABLE role_menu (
  role_id BIGINT NOT NULL,
  menu_id BIGINT NOT NULL,
  PRIMARY KEY (role_id, menu_id)
);
```

权限模型图

```markdown
User
 └── user_role
      └── Role
           └── role_menu
                └── Menu (type=1目录 / type=2菜单 / type=3按钮)
                     └── perms (API 权限)
```

用户登录成功后：

通过 `user_role` 拿到所有 `role_id`

再通过 `role_menu` / `role_permission` 获取所有可访问资源，构建用户的权限集合（`Set<String> perms`）

前端根据接口返回的菜单树来生成左侧导航。

后端返回的菜单树一般是

```json
[
  {
    "id": 1,
    "name": "系统管理",
    "path": "/system",
    "icon": "Setting",
    "children": [
      {
        "id": 2,
        "name": "用户管理",
        "path": "/system/user",
        "component": "system/user/index",
        "children": []
      },
      {
        "id": 3,
        "name": "角色管理",
        "path": "/system/role",
        "component": "system/role/index",
        "children": []
      }
    ]
  },
  {
    "id": 4,
    "name": "商品管理",
    "path": "/product",
    "icon": "Box",
    "children": [
      {
        "id": 5,
        "name": "商品列表",
        "path": "/product/list",
        "component": "product/list/index",
        "children": []
      }
    ]
  }
]
```

前端接收菜单树

```js
import { defineStore } from 'pinia'
import axios from 'axios'

export const useMenuStore = defineStore('menu', {
  state: () => ({
    menuList: []
  }),
  actions: {
    async fetchMenu() {
      // 向后端请求菜单
      const res = await axios.get('/api/system/menu/tree')
      this.menuList = res.data || []
    }
  }
})

```

渲染左侧导航栏

```vue
<template>
  <el-menu
    router
    background-color="#001529"
    text-color="#fff"
    active-text-color="#409EFF"
  >
    <MenuItem v-for="item in menuList" :key="item.id" :menu="item" />
  </el-menu>
</template>

<script setup>
import { useMenuStore } from '@/store/menu'
import MenuItem from './MenuItem.vue'

const menuStore = useMenuStore()
const menuList = computed(() => menuStore.menuList)
</script>
```

递归渲染组件 `MenuItem.vue`

```vue
<template>
  <template v-if="menu.children && menu.children.length">
    <el-sub-menu :index="menu.path">
      <template #title>
        <el-icon><component :is="menu.icon" /></el-icon>
        <span>{{ menu.name }}</span>
      </template>
      <MenuItem v-for="child in menu.children" :key="child.id" :menu="child" />
    </el-sub-menu>
  </template>
  <template v-else>
    <el-menu-item :index="menu.path">
      <el-icon><component :is="menu.icon" /></el-icon>
      <span>{{ menu.name }}</span>
    </el-menu-item>
  </template>
</template>

<script setup>
defineProps({
  menu: {
    type: Object,
    required: true
  }
})
</script>
```

动态生成并注册路由

```js
import router from './index'
import { useMenuStore } from '@/store/menu'

// 自动导入 views 下的所有页面组件
const modules = import.meta.glob('@/views/**/*.vue')

function loadComponent(componentPath) {
  // Layout 特殊处理
  if (componentPath === 'Layout') {
    return () => import('@/layout/index.vue')
  }
  return modules[`/src/${componentPath}`]
}

function generateRoutesFromMenu(menuTree) {
  return menuTree.map(item => {
    const route = {
      path: item.path,
      name: item.name,
      component: loadComponent(item.component),
      meta: item.meta || {}
    }

    if (item.children && item.children.length > 0) {
      route.children = generateRoutesFromMenu(item.children)
    }

    return route
  })
}

export async function setupDynamicRoutes() {
  const menuStore = useMenuStore()
  await menuStore.fetchMenu()
  const routes = generateRoutesFromMenu(menuStore.menuList)
  routes.forEach(r => router.addRoute(r))
}

```

在 `main.js` 中调用

```js
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import router from '@/router'
import { setupDynamicRoutes } from '@/router/dynamic'
import App from './App.vue'

const app = createApp(App)

app.use(createPinia())
app.use(router)

// 在路由准备前加载动态菜单
setupDynamicRoutes().then(() => {
  app.mount('#app')
})

```

前端大致是这个流程

### ABAC

ABAC 是更灵活、更智能的权限模型。它通过 **属性（Attributes）** 来动态计算权限。

权限决策 = 基于用户属性、资源属性、操作属性、环境属性的策略判断。

属性类别

| 类型                    | 示例                         |
| ----------------------- | ---------------------------- |
| 用户属性（User）        | 部门、职位、年龄、级别、地区 |
| 资源属性（Resource）    | 所属部门、资源类型、创建人   |
| 操作属性（Action）      | 读取、修改、删除             |
| 环境属性（Environment） | 时间、地点、设备、安全级别   |

它的 **核心思想**

权限判断不再是简单的角色匹配，而是 **执行策略表达式**（如基于 JSON 或 SpEL 规则）。

例如

```json
{
  "policy_id": "policy-001",
  "effect": "allow",
  "condition": {
    "user.department": "equals resource.department",
    "and": {
      "environment.time": "in business_hours"
    }
  },
  "action": ["read", "update"]
}
```

当用户访问资源时：

1. 收集用户、资源、环境的属性；
2. 加载匹配的策略；
3. 执行策略表达式；
4. 根据结果 `allow` / `deny` 做出决策。

这种模式更加灵活，能处理复杂动态场景；规则可配置化；易于与策略引擎集成。

但是实现复杂；性能开销大；策略管理需要统一规则体系。
