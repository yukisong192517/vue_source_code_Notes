# Vue的组件化
---
## createComponent

在数据驱动中，`_createElement`会进行tag的判断，如果是普通的html标签，实例化vnode节点，否则就会通过createComponent方法创建一个组件的vnode

```
  vnode = createComponent(Ctor, data, context, children, tag)
```

createComponent函数中可以抽象出三个步骤：
* 构造子类构造函数
* 安装组件钩子函数
* 实例化vnode

### 构造子类构造函数

通常编写vue组件的时候，都会通过`export`一个对象，然后将这个对象传入`baseCtor.extend(Ctor)`.这个函数本质上就是执行的`Vue.extend(Ctor)`.
（其中baseCtor就是Vue，在initGlobalApi中定义的`Vue.options._base = Vue`）。

Vue.extend的作用是用来构造一个Vue的子类，使用继承的方法将纯对象转换成为Vue的构造器Sub并返回。Sub是做了扩展的，其中包括options，全局api，初始化了props和computed，以及对于Sub构造函数进行缓存，避免多次执行Vue.extend的时候多次重复构造。
在extend内部，实例化Sub的时候，会执行`this._init`从而再次进入Vue实例的初始化逻辑。


### 安装组件的钩子函数

`installComponentHooks（data）`函数就是将componentVnodeHooks的钩子函数合并在data.hook中，在Vnode执行patch的过程中执行相关的钩子函数，对于已经存在在`data.hook`中的钩子，通过执行mergeHook做合并，最终执行的时候一次执行两个钩子函数即可。

### 实例化VNODE

通过new Vnode 实例化一个vnode并返回，组件vnode是没有children的。

```
const vnode = new VNode(
  `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
  data, undefined, undefined, undefined, context,
  { Ctor, propsData, listeners, tag, children },
  asyncFactory
)
```

createComponent中的三个关键逻辑：构造子类构造函数`Vue.extend(Ctor)`，安装组件钩子函数`installComponentHooks`，实例化vnode。最后返回的是vnode，也一样走到了vm._update方法，然后执行patch函数转换为真正的dom节点。

## 组件的patch

patch的过程会 调用createElm创建元素节点，对于能够createComponent则直接返回

## createComponent

atch的过程会 调用createElm创建元素节点，对于能够createComponent则直接返回。`createComponent`首先会对vnode.data进行判断，如果vnode是组件Vnode，会执行init(vnode,false)函数。

### init

这个函数通过`createComponentInstanceForVnode`
创建一个Vue实例，然后调用`$mount`方法进行挂载子组件。

### createComponentInstanceForVnode

构造一个内部组件的参数，然后通过执行vnode.componentOptions.Ctor（options）子组件的构造函数，（也就是上面那个继承自super的构造函数）。
函数内部会将`_isComponent`设置为true。这个属性会对`_init`的执行有影响。这个构造函数执行的时候就会执行子组件内部的`_init`.如果`_isComponent`为true的话，则会执行`initInternalComponent`.initInternal函数会将传入的参数合并到`$option`中。

最后会通过执行`_init`，执行子组件的`$mount`方法，从而调用`mountComponent`方法，进而执行`_render`.组件的初始化没有el传入，因此`$mount`方法自己接管。

### `_render`

对于组件的render函数与与全局的render函数的区别在于，组件的会进行父子关系的确认。
`_parentVnode`为当前组件的父组件。通过render函数生成的当前组件的渲染vnode，通过将`vm.$vnode=_parentVnode`,指定了父子关系。

### `_update`

update函数将render生成的vnode渲染出来，update函数的关键逻辑：
* 保留当前的`activeInstance`。vue的初始化是一个深度遍历的过程，为了正确的实例化，需要明确当前的上下文vue实例是什么
* `vm._vnode = vnode` 将组件的渲染vnode赋值给_vnode,因此可以看出`$vnode`和`_vnode`是父子关系。
* `vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)`
在patch函数内部，负责渲染的函数是createElm。对于组件vnode，parentElm传入的是undefined。先创建一个父组件的占位符，遍历子组件递归调用createElm。在完成组件的整个 patch 过程后，最后执行 insert(parentElm, vnode.elm, refElm) 完成组件的 DOM 插入，如果组件 patch 过程中又创建了子组件，那么DOM 的插入顺序是先子后父。



## 合并配置

new Vue的过程有两种，主动调用new Vue(options)实例化，另一种组件过程中通过new Vue（options）实例化组件。

无论哪一种都会执行_init方法，就会执行内部的mergeOptions。

mergeOptions对于不同的场景，合并逻辑是不一样的，传入的options也不同。

### 外部调用场景

new Vue执行下面的逻辑进行options的合并。
```
vm.$options = mergeOptions(
  resolveConstructorOptions(vm.constructor),
  options || {},
  vm
)
```

在这个场景还是简单返回vm.constructor.options(Vue.options).这个值的定义是在initGlobalApi中，通过创建空对象，遍历assets_types进行赋值。

```
export const ASSET_TYPES = [
  'component',
  'directive',
  'filter'
]
```

接下来执行_base的赋值，方便在之后的子组件创建中使用构造函数。接着将`extend(Vue.options.components, builtInComponents)`将内置组件扩展到component上。

mergeOptions的主要功能就是将parent和child这两个对象根据一些策略合并为一个新对象并且返回。合并规则：
* 递归的将extends和mixin合并到parent
* 遍历parent调用mergeField。
* 遍历child，key如果不在parent的自身属性上，则调用mergeField。
* mergeField对于不同的key有不同的合并策略。比如生命周期钩子函数都调用mergeHook函数。对于不存在childVal就返回parentVal。存在就判断是否存在parentVal,如果存在就合并返回新数组，否则就返回childVal。
* 最后mergeField将合并的结果保存在options中。
在合并后的options：
```
vm.$options = {
  components: { },
  created: [
    function created() {
      console.log('parent created')
    }
  ],
  directives: { },
  filters: { },
  _base: function Vue(options) {
    // ...
  },
  el: "#app",
  render: function (h) {
    //...
  }
}
```
### 组件场景

组件是通过Vue.extend继承自vue得到构造函数的。组件通过调用`vnode.componentCtor(options)`接着执行`_init`，然后走到了initInternalComponent逻辑。

在initInternalComponent函数中首先根据构造函数中的options复制一份options出来。接着把父Vnode实例，父Vue实例保存在`$options
`中。

执行initInternalComponent之后，$options实例：
```
vm.$options = {
  parent: Vue /*父Vue实例*/,
  propsData: undefined,
  _componentTag: undefined,
  _parentVnode: VNode /*父VNode实例*/,
  _renderChildren:undefined,
  __proto__: {
    components: { },
    directives: { },
    filters: { },
    _base: function Vue(options) {
        //...
    },
    _Ctor: {},
    created: [
      function created() {
        console.log('parent created')
      }, function created() {
        console.log('child created')
      }
    ],
    mounted: [
      function mounted() {
        console.log('child mounted')
      }
    ],
    data() {
       return {
         msg: 'Hello Vue'
       }
    },
    template: '<div>{{msg}}</div>'
  }
}
```

## 组件的生命周期

生命周期的执行一定是调用callHook方法。这个方法根据传入的hook，拿到vm.$options[hook]对应的回调函数数组，再依次执行。

### beforeCreate & created

* 执行时机： _init中initState的前后。 
* 这两个钩子函数执行的时候没有渲染dom。如果与后端的交互可以放在这两个钩子，但是如果需要访问props，data等需要created钩子。vue-router和vuex都进行了beforeCreated钩子函数的mixin。

### beforeMounted & mounted

beforeMount在mount之前被调用，调用时机是mountComponent函数中。在执行vm.render函数渲染vnode之前，执行beforemount。在执行_update函数之后执行mounted钩子。

mount钩子会对$vnode进行判断，如果为null证明这不是组件初始化而是new Vue的初始化。组件的mount则是在patch到dom之后，调用invokeInsertHook函数，将内部的钩子函数执行一遍。函数内部最终调用的是insert函数，insert函数调用mounted钩子函数执行。insert的添加顺序是先子后父，因此mounted的顺序也是先子后父。

### beforeUpdate和updated

beforeUpdate函数的执行时机是渲染watcher的before函数中。在mountComponent函数中会初始化一个渲染watcher，beforeUpdate就是在这里执行的。

update是在flushScheduleQueue函数调用的时候。如果当前的watcher为`vm._watcher`并且mounted已经完成的时候，才会对updateQueue执行update。渲染watcher是用来监听vm上的数据变化重新渲染。只有在`vm._watcher`回调执行完成之后，才会执行update函数。

### beforeDestroy & destroyed

* 执行时机：组件销毁阶段，
* before执行在销毁函数的最开始。
* 接着会从parent的$children数组中删除自身，删除watcher，当前渲染vnode执行销毁钩子函数等，最后调用destroyed函数。
* 在销毁过程中会再次执行`__patch__`函数触发子组件的销毁钩子函数。因此destroyed 的顺序也是先子后父。也就是说父子的递归都是在patch函数中进行的。

### activated & deactivated

keep-alive组件专用。


## 组件的注册

### 全局注册

调用全局api：`Vue.component`。这个函数定义的位置是在初始化全局函数的时候。

### 局部注册

* 组件vue实例化的阶段有合并options的逻辑，通过将components合并到vm.$options.components,就可以在createComponent的时候作为参数传入。

一般基础组件是全局注册的，业务组件是局部注册的。

## 异步注册

一般为了减小首屏的体积，将一些非首屏的组件设计为异步组件。

```
Vue.component('xx',function(resolve,reject){
require(['./xxx'],resolve)
})
Vue.component('xx',()=>import('xx'))
const asynccomp = ()=>{
{
compoent:import('xx'),
loading：loadingcomp，
error:errorComp
}
}
Vue.component('xx',asynccomp)
```

因为异步组件不会执行Vue.extend也就不会有cid，先执行了resolveAsyncComponent处理了三种不同异步组件的创建方式。

### 普通异步函数

通过执行`factory(resolve,reject)`进行加载，拿到对象之后执行resolve逻辑，在此之前先执行factory.resolved = ensureCtor(res,baseCtor)确保能找到js定义的组件对象。
在resolve之后会执行forceUpdate，调用渲染watcher的update方法让渲染watcher对应的回调函数执行，触发组件的重新渲染。

### promise异步组件

因为返回的是一个promise对象，命中`typeof res.then === function`逻辑。当组件加载成功之后，执行resolve。

### 高级异步组件

高级异步和普通的异步都是执行resolveAsyncComponent，接着执行res.component.then进行加载，接着会判断error组件和loading组件，以及延时的定义。
如果组价加载失败，会执行reject函数，同时error为true，执行forceRender再次执行resolveAsyncComponent,直接渲染errorComp。
如果组件加载成功，会执行resolve函数，将结果缓存到factory.resolved中，再次调用resolveAsyncComponent，渲染成功的组件

### 异步组件的patch

如果是第一次执行resovleAsyncComponent，通常会返回undefined，通过createAsyncPlaceHolder创建一个注释节点占位符。之后再forceRender的时候触发重新渲染，再次resovleAsyncComponent，就走正常的render，patch，update的过程，
