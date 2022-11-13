# setup注意点
- `this`为`undefined`
- 无法获取vue2写法的option配置(vue3可以用vue2写法混写)
- `setup`函数有两个参数
  - `props` 父组件传入的props属性
  - `content` 上下文，提供了一些属性和方法
    - `content.attrs` 存放未被`props`配置项声明的props属性
    - `content.emit` 用于触发父组件事件
    - `content.slots` 父组件传入子组件的插槽，父组件如果传入命名插槽需要使用`v-slot=xxx`
- setup函数在`beforeCreate`之前执行
- 返回值支持promise（需要<strong color="red">Suspense+异步引入</strong>配合使用）

# 响应式原理
## ref
- 可以传入简单数据类型、复杂数据类型（复杂数据类型不推荐使用）
- 在`setup`中使用，需要通过`.value`的方式
- 在template模板中使用，需要省略`.value`
- 简单数据类型通过`Object.defineProperty`，复杂数据类型通过`Proxy`
  
## reactive
- 只能复杂数据类型使用
- 不要`.value`就能使用
- 通过`Proxy`实现

## 通过Proxy实现原理
```js
  const obj = { a: 1 }
  new Proxy(obj, {
    // 获取
    get: (target, propName) => {
      return Reflect.get(target, propName)
    },
    // 添加、修改
    set: (target, propName, value) => {
      Reflect.set(target, propName, value)
    },
    // 删除
    deleteProperty: (target, propName) => {
      return Reflect.deleteProperty(target, propName)
    }
  })
```

# computed计算属性
```js
  import { reactive, computed } from 'vue'


  setup() {
    let obj = reactive({
      name: 'xxx',
      age: '11'
    })

    // 简单使用
    let a = computed(() => {
      return obj.name + '-' + obj.age
    })

    // 完整使用，支持计算属性读+写
    let a = computed({
      get() {
        return obj.name + '-' + obj.age
      },
      set(value) {
        const [name, age] = value.split('-')
        obj.name = name
        obj.age = age
      }
    })
  }
```

# watch 监听
## 注意
- 只能传入ref、reactive、数组格式数据
- watch函数有三个参数
  - 监听对象
  - 处理函数
  - 配置项
    - `immediate` 默认`false`
    - `deep` 默认`false`，reactive定义的值，`deep`强制为`true`
- ref传入简单、复杂数据类型时，监听传参不一样
  - 简单 `watch(xxx, () => {})`
  - 复杂 `watch(xxx.value, () => {})` 或者 `watch(xxx, () => {}, { deep: true })`
## 监听 ref 定义的数据
```js
let num = ref(1)
// newValue, oldValue不用.value获取
watch(num, (newValue, oldValue) => {})
```
## 监听 多个 响应式数据
```js
let num = ref(1)
let num2 = ref(1)
// newValue, oldValue为数组，对应[newNum, newNum2]，[oldNum, oldNum2]
watch([num, num2], (newValue, oldValue) => {})
```
## 监听 reactive 定义的全部属性
```js
let obj = ref({
  a: {
    b: { c: 1 }
  }
})
// deep强制开启
watch(obj, (newValue, oldValue) => {})
```
## 监听 reactive 定义的某个属性
```js
let obj = ref({
  a: 1
})
watch(() => obj.a, (newValue, oldValue) => {})
```
## 监听 reactive 定义的某些属性
```js
let obj = ref({
  a: 1,
  b: 2
})
watch([() => obj.a, () => obj.b], (newValue, oldValue) => {})
```
## 特殊情况 reactive 定义对象中的某个复杂类型属性的属性
```js
let obj = ref({
  a: {
    b: { c: 1 }
  }
})
// 需要开启deep
watch(() => obj.a, (newValue, oldValue) => {}, { deep: true })
```

# watchEffect
- 会立即执行(immediate: true)
- 只有回调内部用到的属性发生变化才会执行
```js
const x1 = ref(1)
const x2 = reactive({
  a: 1,
  b: { c: 2 }
})

// 只有x1.value和x2.b.c发生变化，才会执行
// x2.a发生变化，不会执行
watchEffect(() => {
  const xx1 = x1.value
  const xx2 = x2.b.c
})
```
### 与computed的区别
- computed更注重值
- watchEffect更注重过程

# 生命周期
- option方式
  - mount
    - vue3 mount执行之后才会进行后续的生命周期
    - vue2 会先执行`beforeCreate`，`created`，然后再判断mount
  - 卸载
    - vue3 `beforeUnmount` 和 `unmounted`
    - vue2 `beforeDestroy` 和 `destroyed`
- 组合式api
  - beforeCreate ==> setup()
  - created ==> setup()
  - beforeMount ==> onBeforeMount()
  - mounted ==> onMounted()
  - beforeUpdate ==> onBeforeUpdate()
  - updated ==> onUpdated()
  - beforeUnmount ==> onBeforeUnmount()
  - unmounted ==> onUnmounted()
- 执行顺序
  - 组合在option之前执行
  - setup最早执行

# 自定义hook
把setup函数中，使用的composition api进行了封装。类似vue2中的mixin<br>
组件可复用逻辑等，封装成一个函数，在return功能需要的数据，需要用的组件引入该方法即可。

# API —— toRef，toRefs
作用：创建一个ref对象，其value值指向另一个对象中的某个属性
```js
  let obj = ref({ a: 1, b: { c: 1 } })
  const a1 = toRef(obj, 'a')
  const a2 = toRef(obj.b, 'c')

  const { a, b } = toRefs(obj)
```

# API —— shallowRef，shallowReactive
### shallowRef
- `shallowRef`不会处理对象类型的数据（和`ref`的区别）
### shallowReactive
- 只会代理最外层的属性，深层嵌套不会代理
### 使用时机
- 对象数据只用表层属性性能优化时用 —— `shallowReactive`
- 对象数据不会修改属性值，只会替换对象 —— `shallowRef`

# API —— readonly，shallowReadonly
### readonly
- 将响应式数据变成只读
### shallowReadonly
- 将响应式数据的 表层属性变成只读

# API —— toRaw，markRaw
### toRaw
- 将响应式数据变为原始数据
### markRaw
- 标记一个对象，使其永远不会变为响应式对象
- 应用：第三方库、不需要响应式的大型数据时的性能优化

# API —— customRef
- 作用：实现一个自定义ref，并对其依赖项和更新触发进行显示控制
```js
// 自定义ref防抖功能
setup() {
  function myRef(value, delay = 300) {
    let timer
    return customRef((track, trigger) => {
      return {
        get() {
          // 通知vue追踪数据的变化
          track()
          return value
        },
        set(v) {
          value = v
          // 通知vue重新解析模板
          clearTimeout(timer)
          timer = setTimeout(() => {
            trigger()
          }, delay)
        }
      }
    })
  }

  const a = myRef('hello ')
}
```

# API —— provide, inject
- 作用：祖孙组件通信
```js
  // 父组件
  let car = ref('xxx')
  provide('car', car)

  // 子组件
  let car = inject('car')
```

# 判断响应式数据
- `isRef` 判断ref
- `isReactive` 判断reactive
- `isReadonly` 判断readonly
- `isProxy` 判断是否reactive或者readonly创建的

# 组件 —— Fragment
- vue3 可以没有根标签
- 内部会将多标签包含在Fragment虚拟元素中

# 组件 —— Teleport
将组件移动到指定位置
```js
  // 将内容传送到body标签下
  <teleport to="body">
    <div>xxxxx</div>
  </teleport>
```

# 组件 —— Suspense
等待异步组件时，渲染一些额外内容
```js
// 动态引入
import { defineAsyncComponent } from 'vue'
const c = defineAsyncComponent(() => import('./xxx'))

export default {
  components: {
    c
  }
}

// template模板
<Suspense>
  <template v-solt:deault>
    <c />
  </template>
  <template v-solt:fallback>
    <div>加载中</div>
  </template>
</Suspense>
```

# vue3其他改变
- 转移
  - `Vue.config.xxx` => `app.config.xxx`
  - `Vue.config.productionTip` => 移除
  - `Vue.component` => `app.component`
  - `Vue.directive` => `app.directive`
  - `Vue.mixin` => `app.mixin`
  - `Vue.use` => `app.use`
  - `Vue.prototype` => `app.config.grobalPorperties`
- `data`配置项必须是函数
- 过渡类名更改
  - a
    - .v-enter => .v-enter-from
    - .v-leave-to => .v-leave-to
  - b
    - .v-leave => .v-leave-from
    - .v-enter-to => .v-enter-to
- 移除`keyCode`作为`v-on`修饰符，同时不支持`config.keyCodes`
- 移除`v-on.native`（原生事件）
  - vue3通过emits: ['close', 'click']指定自定义事件
- 移除`filter`过滤器
  - 学习、使用成本高（建议通过方法、计算属性实现）


