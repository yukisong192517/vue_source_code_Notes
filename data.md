# Vue 的 数据驱动相关

---

## new Vue

位置在`src/core/instance/index.js`中：
Vue构造函数中： 判断this是不是Vue的实例，不是的话证明构造函数的调用方式有问题。然后调用`this._init(options)`初始化。

`this._init`在`src/core/instance/init.js`中定义:

 - 定义私有uid： vm._uid = uid++
 - 标识为vue：_isVue为true，
 - 合并配置
 - 初始化Proxy
 - 私有_self设置为当前vm
 - 初始化生命周期
 - 初始化事件
 - 初始化渲染render
 - 调用beforeCreate
 - 初始化注入
 - 初始化props，data，compued，watcher
 - 初始化provide
 - 调用生命周期函数created
 - 如果定义了el属性调用$mount进行挂载，渲染成dom

## Vue 实例的挂载

 1. 挂载方式：`$mount`挂载vm。
 2. 对于compiler版本的`$mount`定义：
    - 缓存原型上的`$mount`,
    - 检测是否挂载在html或者body标签上面
    - 如果没有定义render方法，会将el或者template转换成为render方法。这就是在线编译的过程通过`compileToFunctions`方法实现。
    - 调用`$mount`方法进行挂载
 3. $mount方法支持传入2个参数，
el标识挂载的元素，字符串或者dom对象。字符串会通过query转换成dom对象。
第二个参数和服务端渲染相关。

### mountComponent

$mount最终调用mountComponent方法，定义在`src/core/instance/lifecycle.js`文件中。

- 主要流程：
    - 实例化一个渲染watcher 
    - 回调函数中调用updateComponent方法
    - 调用vm._render方法生成虚拟dom   
    - 调用_update更新dom
    - 判断根节点（$vnode为null）的时候设置vm._isMounted = true,表示已经被挂载了，同时执行mount钩子函数。

- 实例化渲染watcher的作用：
    - 初始化时执行回调函数执行updateComponent
    - 当vm实例中的检测数据发生变化的时候再次执行回调函数。

## render

- 用来将一个实例渲染成一个虚拟node
- 位置： `src/core/instance/render.js`
- 入参：createElement
- 主要流程：
    -  赋值`$scopeslots`
    -  赋值`$vode`，标识父节点
    -  **`render.call(vm._renderProxy,vm.$createElement)`**
    -  如果是空节点： createEmptyVnode
    -  vnode.parent赋值为父节点

`vm._render`通过执行createElement方法返回vnode。vnode就是一个virtual Dom。

### Virtual Dom

 - 用一个js对象描述dom节点，性能比直接创建dom要小的多。
 - 文件位置： `src/core/vdom/vnode.js`
 - 核心属性：
标签名，数据，子节点，键值。vnode本身只是用来映射到真实dom的渲染，不需要包含操作dom的方法。从vnode到真实的dom需要经历vnode的create，diff，patch。createElement就实现了create vnode。

### createElement

- Vue中使用createElement方法创建vnode
- 文件位置： createElement定义在`src/core/vdom/createelement.js`中。
- 实际上是调用的_createElement，但是为了使参数更加灵活，因此封装了一层。

### _createElement

- 入参： context表示component上下文环境，tag为标签，data是vnode的数据，children为子节点数组，normalizationType是规范类型。因为render分为编译生成还是手动编写的，因此要规范这个。
- children的处理
    - vnode理论上应该是一个树形结构，每一个vnode都会有若干的子节点。规范化之后都应该是vnode类型的，但是在调用的时候，children为任意类型的。
    - 对于编译生成的children数组理论上都是vnode数组，但是对于functionComponent返回的是一个数组而不是根节点，因此要将children打平，让他深度只有一层，调用的是simpleNormalizeChildren
    - normalizeChildren 调用的场景：render函数为用户手写的，或者当编译slot，v-for的时候会产生嵌套数组的情况。
    该函数接受两个参数：children为要规范子节点，nestedIndex表示嵌套的索引。如果对于元素是数组类型，则递归调用该函数，如果是基础类型通过createTextVnode转换为vnode，如果children的children也是数组，就根据nestedIndex更新key。
    normalizeChildren最后返回的是一个vnode的array。

- 创建vnode实例
  判断tag， 如果是普通的内置节点创建普通的vnode（new Vnode）。如果是注册过的组件名，通过createComponent创建组件类型vnode，如果位置标签就创建未知标签的vnode（通过new Vnode），如果tag是component类型，通过createComponent创建。
        
可以看出，vnode有children，children的每个元素都是vnode。因此最后就形成了dom tree。

通过_render就创建好了vnode，接下来需要的是渲染这个vnode。


## update

_update是实例的私有方法，在首次渲染的时候会被调用，以及在数据更新的时候会被调用。这个函数的作用就是把vnode渲染成为真实的dom。

本质上调用的是`vm.__patch__`方法(在`src/platforms/web/runtime/index.js`中)，而这个方法最终指向的是`src/platforms/web/runtime/patch.js`文件。

### patch 方法

`createPatchFunction` 通过传入一个对象，包含nodeOps参数（封装了dom操作）和modules的参数（定义了模块的钩子函数），最终返回一个patch方法赋值给了`vm.__patch__`。

patch方法接受4个参数，oldVnode，vnode，hydrating,removeOnly。oldvnode表示旧的节点，可以不存在或者是一个dom对象。vnode是_render之后返回的vnode节点。hydrating是否需要服务端渲染，removeonly是用于transition组件相关。

在update中调用时可能的样子是：
`vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)`当首次执行时，$el是id为app的dom对象。

判断oldvnode是否为真实的dom。对于服务端渲染会有不同的处理方式。通过emptyNodeat方法将oldVnode转换为vnode对象，然后通过调用那个createElm方法创建一个真实的dom节点并插入到他的父节点中。

### createElm

createElm会先判断是否定义了`vnode.elm`和`ownerarray`.这两个属性用来标识是已经渲染过的vnode还是全新的vnode。对于渲染过的vnode拷贝放入ownerarr中当前vnode的位置。

接下来会尝试创建组件，如果成功就直接返回了。

如果不成功的话，先会对vnode的tag进行合法性判断，然后调用平台dom的操作去创建一个占位符。如果没有tag有可能是注释或者纯文本节点，可以直接的插入

**调用createChildren创建子元素，遍历子虚拟节点，递归的调用createElm** 为子虚拟节点创建子元素.如果vnode.text是普通类型，遍历过程中会把 vnode.elm 作为父容器的 DOM 节点占位符传入。

接下来调用**invokeCreateHooks执行create钩子函数**，并把vnode推入insertedVnodeQueue。

最后通过**insert方法将dom插入父节点**，子元素先insert。插入通过的是appendChild调用原生dom api进行操作。


总结：
整个过程：
从new Vue开始，调用`_init`方法进行合并初始化配置，调用$mount 进行实例挂载到真实的dom上面。`$mount`的过程是通过compile之后，调用`_render`生成vnode，在`update`方法中调用`_patch` 将虚拟的vnode映射成为真实的dom。
