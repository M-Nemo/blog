---
title: Vue+vuex+vue-router实现多级权限路由管理
date: 2018-03-09 22:14:26
tags: 
    - FE
    - Vue
categoties: Vue
---
后台管理项目中通常会遇到多级权限管理的问题，时常需要根据不同的权限来渲染不同的导航栏，也就是路由，在参考了QQ群中一些开源项目后，针对自己的项目也开发出了一套适用的路由管理，在这里也感谢dalao提供的帮助

------
### 后台权限标识
根据项目的需求，我们把后台管理员身份分成三种

| role      |    type | description  |
| :-------- | :--------:| :--|
| 超级管理员  |   0     |  拥有所有权限，拥有控制其他管理员的权限  |
| 运营管理员  |   1     |  拥有所有权限，不能控制其他管理员 |
| 分支管理员  |   2     |  拥有受限制的操作权限  |

### 具体思路
1. 建立两套路由表，一套是所有管理员都能查看的部分baseRouter，一套是根据权限动态分配的路由表asyncRouter
2. 登录鉴权：登录时校验用户名密码，校验成功后获取用户类型
3. 每次路由前校验权限，根据权限递归筛选出符合当前权限的路由表并添加到路由中，然后根据路由表渲染导航栏

### 部分代码实现
#### 登录
```javascript
handleLogin() {
    this.$refs.loginForm.validate ( valid => {
        if (valid) {
            this.loading = true
            this.$store.dispatch ( 'CheckUserLogin', this.loginForm ).then ( (response) => {
                console.log(response);
                if(response.status!=200){
                    this.$message.error ( '请求失败，网络原因请重试！' );
                }else {
                    if(response.data.data&&response.data.code==0){
                        this.$router.push ( { path : '/' } )
                    }else{
                        this.$message.error ( '用户名或密码错误！' );
                    }

                }
                this.loading = false
            } ).catch ( () => {
                this.loading = false
            } )
        } else {
            console.log ( 'error submit!!' )
            return false
        }
    } )
},
```
#### Vuex递归获取符合权限的路由表

```javascript
function hasPermission(roles, route) {
  if (route.meta && route.meta.role) {
    // return roles.some(role => route.meta.role.indexOf(role) >= 0)
    return route.meta.role.indexOf(roles) >= 0
  } else {
    return true
  }
}

/**
 * 递归过滤异步路由表，返回符合用户角色权限的路由表
 * @param asyncRouterMap
 * @param roles
 */
function filterAsyncRouter(asyncRouterMap, roles) {
  const accessedRouters = asyncRouterMap.filter(route => {
    if (hasPermission(roles, route)) {
      if (route.children && route.children.length) {
        route.children = filterAsyncRouter(route.children, roles)
      }
      return true
    }
    return false
  })
  return accessedRouters
}

```

#### 路由处理

```javascript
router.beforeEach ( ( to, from, next ) => {
    NProgress.start () // 开启Progress
    let user = JSON.parse(sessionStorage.getItem('user'));
    if (user) {
        if (to.path === '/login') {
            next ( { path : '/' } )
            NProgress.done () 
        } else {
            if (store.getters.roles.length === 0) { // 判断当前用户是否已拉取完user_info信息
                store.dispatch ( 'GetUserInfo',user ).then ( res => { // 拉取user_info
                    const roles = res.data.data.type
                    store.dispatch ( 'GenerateRoutes', { roles } ).then ( () => { // 生成可访问的路由表
                        router.addRoutes ( store.getters.addRouters ) // 动态添加可访问路由表
                        next ( { ... to } )
                    } )
                } ).catch ( () => {
                    store.dispatch ( 'FedLogOut' ).then ( () => {
                        Message.error ( '验证失败,请重新登录' )
                        next ( { path : '/login' } )
                    } )
                } )
            } else {
                next()
            }
        }
    } else {
        if (whiteList.indexOf ( to.path ) !== - 1) { // 在免登录白名单，直接进入
            next ()
        } else {
            next ( '/login' ) // 否则全部重定向到登录页
            NProgress.done ()
        }
    }
} )
```