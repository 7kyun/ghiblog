

<p align='center'>
    <img src="https://badgen.net/badge/labels/6"/>
    <img src="https://badgen.net/github/issues/7kyun/ghiblog"/>
    <img src="https://badgen.net/badge/last-commit/2022-07-14 07:34:33"/>
    <img src="https://badgen.net/github/forks/7kyun/ghiblog"/>
    <img src="https://badgen.net/github/stars/7kyun/ghiblog"/>
    <img src="https://badgen.net/github/watchers/7kyun/ghiblog"/>
    <img src="https://badgen.net/github/release/7kyun/ghiblog"/>
</p>

<p align='center'>
    <img src="https://visitor-badge.glitch.me/badge?page_id=7kyun.ghiblog"/>
</p>



<p align='center'>
<a href='https://github.com/7kyun/ghiblog/issues/1#issuecomment-1120738390'>
<img src='https://user-images.githubusercontent.com/56475308/167338714-306950ac-bc9e-4968-a5d6-22fc157362db.jpg' width='50%' alt='
看看月亮吧 '>
</a>
</p>
<p align='center'>
<span>
看看月亮吧 </span>
</p>

    
## 置顶 :pushpin: 
- [为什么会出现这个博客](https://github.com/7kyun/ghiblog/issues/6)  <sup>0 :speech_balloon:</sup>  	 
## 最新 :new: 

#### [Vue源码-双向数据绑定原理（四）](https://github.com/7kyun/ghiblog/issues/7) <sup>0 :speech_balloon:</sup> 	 2022-07-14 07:34:05

:label: : [:v:vue](https://github.com/7kyun/ghiblog/labels/%3Av%3Avue), [:scroll:源码解读](https://github.com/7kyun/ghiblog/labels/%3Ascroll%3A%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB)

### Dep
首先是 `Dep` 类
其内部会维护一个 `id`，用作 `Dep` 实例的唯一标识
还有一个 `subs` 数组，用于存贮订阅的 `Watcher` 实例

`addSub`、`removeSub` 用于 `Watcher` 实例新增和移除
`depend` 实际上就是调用了 `Watcher` 实例中的 `addDep` 方法
`notify` 订阅的数据改变时，用于通知视图更新

[相关源码](https://github.com/vuejs/vue/blob/2.6/src/core/observer/dep.js#L13)
```ts
let uid = 0

/**
 * A dep is an observable that can have multiple
 * directives subscribing to it.
 */
export default class Dep {
  static target: ?Watcher;
  id: number;
  subs: Array<Watcher>;

  constructor () {
    this.id = uid++
    this.subs = []
  }

  // 添加观察者
  addSub (sub: Watcher) {
    this.subs.push(sub)
  }

  // 移除观察者
  removeSub (sub: Watcher) {
    remove(this.subs, sub)
  }

  // 依赖收集，当存在 target 的时候添加观察者对象
  depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }

  notify () {
    // stabilize the subscriber list first
    // 拷贝一份订阅列表
    const subs = this.subs.slice()

    // 关闭 async 的情况会对订阅列表进行排序 让数据按顺序更新
    if (process.env.NODE_ENV !== 'production' && !config.async) {
      // subs aren't sorted in scheduler if not running async
      // we need to sort them now to make sure they fire in correct
      // order
      subs.sort((a, b) => a.id - b.id)
    }
    // 通知所有订阅者
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
}

// The current target watcher being evaluated.
// This is globally unique because only one watcher
// can be evaluated at a time.
Dep.target = null
// 依赖收集完后将 target 设为 null
// 防止后面重复添加依赖

```

### Watcher
这里对 `Watcher` 类相当一部分的边界条件的处理及特殊情况(deep、lazy等)的处理代码进行删减，以便能清晰的解读原理部分，删减部分的内容，有时间有兴趣的自己去看一看就行了
[相关源码](https://github.com/vuejs/vue/blob/2.6/src/core/observer/watcher.js#L26)
```ts
/**
 * A watcher parses an expression, collects dependencies,
 * and fires callback when the expression value changes.
 * This is used for both the $watch() api and directives.
 */
export default class Watcher {
  vm: Component;
  expression: string;
  cb: Function;
  // ...

  constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    // ...
  ) {
    this.vm = vm
    if (isRenderWatcher) {
      vm._watcher = this
    }
    vm._watchers.push(this)
    
    // ...

    this.cb = cb
    this.id = ++uid // uid for batching
    
    // ...

    this.expression = process.env.NODE_ENV !== 'production'
      ? expOrFn.toString()
      : ''
    // parse expression for getter
    if (typeof expOrFn === 'function') {
      this.getter = expOrFn
    } else {
      this.getter = parsePath(expOrFn)
      if (!this.getter) {
        this.getter = noop
        process.env.NODE_ENV !== 'production' && warn(
          `Failed watching path: "${expOrFn}" ` +
          'Watcher only accepts simple dot-delimited paths. ' +
          'For full control, use a function instead.',
          vm
        )
      }
    }
    this.value = this.lazy
      ? undefined
      : this.get()
  }

  /**
   * Evaluate the getter, and re-collect dependencies.
   */
  get () {
    pushTarget(this)
    let value
    const vm = this.vm
    
    try {
      value = this.getter.call(vm, vm)
    } catch (e) {
      if (this.user) {
        handleError(e, vm, `getter for watcher "${this.expression}"`)
      } else {
        throw e
      }
    } finally {
      // "touch" every property so they are all tracked as
      // dependencies for deep watching
      if (this.deep) {
        traverse(value)
      }

      // 将当前实例从 targetStack 中取出 并将 target 设为堆栈中的最后一项
      popTarget()
      this.cleanupDeps()
    }
    return value
  }

  /**
   * Add a dependency to this directive.
   */
  addDep (dep: Dep) {
    const id = dep.id
    if (!this.newDepIds.has(id)) {
      this.newDepIds.add(id)
      this.newDeps.push(dep)
      if (!this.depIds.has(id)) {
        dep.addSub(this)
      }
    }
  }

  /**
   * Clean up for dependency collection.
   */
  cleanupDeps () {
    let i = this.deps.length
    while (i--) {
      const dep = this.deps[i]
      if (!this.newDepIds.has(dep.id)) {
        dep.removeSub(this)
      }
    }
    let tmp = this.depIds
    this.depIds = this.newDepIds
    this.newDepIds = tmp
    this.newDepIds.clear()
    tmp = this.deps
    this.deps = this.newDeps
    this.newDeps = tmp
    this.newDeps.length = 0
  }

  /**
   * Subscriber interface.
   * Will be called when a dependency changes.
   */
  update () {
    /* istanbul ignore else */
    if (this.lazy) {
      this.dirty = true
    } else if (this.sync) {
      this.run()
    } else {
      queueWatcher(this)
    }
  }

  /**
   * Scheduler job interface.
   * Will be called by the scheduler.
   */
  run () {
    if (this.active) {
      const value = this.get()
      if (
        value !== this.value ||
        // Deep watchers and watchers on Object/Arrays should fire even
        // when the value is the same, because the value may
        // have mutated.
        isObject(value) ||
        this.deep
      ) {
        // set new value
        const oldValue = this.value
        this.value = value
        if (this.user) {
          try {
            this.cb.call(this.vm, value, oldValue)
          } catch (e) {
            handleError(e, this.vm, `callback for watcher "${this.expression}"`)
          }
        } else {
          this.cb.call(this.vm, value, oldValue)
        }
      }
    }
  }

  /**
   * Evaluate the value of the watcher.
   * This only gets called for lazy watchers.
   */
  evaluate () {
    this.value = this.get()
    this.dirty = false
  }

  /**
   * Depend on all deps collected by this watcher.
   */
  depend () {
    let i = this.deps.length
    while (i--) {
      this.deps[i].depend()
    }
  }

  /**
   * Remove self from all dependencies' subscriber list.
   */
  teardown () {
    if (this.active) {
      // remove self from vm's watcher list
      // this is a somewhat expensive operation so we skip it
      // if the vm is being destroyed.
      if (!this.vm._isBeingDestroyed) {
        remove(this.vm._watchers, this)
      }
      let i = this.deps.length
      while (i--) {
        this.deps[i].removeSub(this)
      }
      this.active = false
    }
  }
}
```

`Watcher` 实例化会执行 `this.get` 方法，其内部会通过 `expOrFn` 方法执行相应的 `getter`，在这个过程中，或渲染页面，或获取某个数据的值。总之，会通过 `getter`，来建立数据的双向绑定。当绑定的数据改变时，会由 `Dep` 触发 `notify`，通知所有绑定的 `Watcher` 的 `update` 方法，进而调用 `run` 方法，`run` 方法中调用 `this.get` 获取修改后的数据，最后调用 `cb` 更新视图。



[更多>>>](https://github.com/7kyun/ghiblog/issues/7)

---


#### [为什么会出现这个博客](https://github.com/7kyun/ghiblog/issues/6) <sup>0 :speech_balloon:</sup> 	 2022-05-12 06:03:26

:label: : [:pushpin:置顶](https://github.com/7kyun/ghiblog/labels/%3Apushpin%3A%E7%BD%AE%E9%A1%B6), [:pencil2:随笔](https://github.com/7kyun/ghiblog/labels/%3Apencil2%3A%E9%9A%8F%E7%AC%94)

### 碎碎念
> 关于为什么现在才开始写博客，其实我更愿意把这称为笔记

之前看过一篇文章，他提到，快速改变人生的五件事情：早起，阅读，写作，运动，冥想。
想了想，我好像除了早起，阅读，运动都做到了，那我改变自己的人生岂不是就剩写作和冥想？
因此为了能快速的改变人生，我毅然决然的开启了写

[更多>>>](https://github.com/7kyun/ghiblog/issues/6)

---


#### [Vue源码-双向数据绑定原理（二）](https://github.com/7kyun/ghiblog/issues/5) <sup>0 :speech_balloon:</sup> 	 2022-05-11 07:34:00

:label: : [:v:vue](https://github.com/7kyun/ghiblog/labels/%3Av%3Avue), [:scroll:源码解读](https://github.com/7kyun/ghiblog/labels/%3Ascroll%3A%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB)

### defineReactive
`defineReactive` 就是定义对象上的响应性属性的方法
它是通过 `Object.defineProperty` 为数据定义上 `getter\setter` 方法
该方法会形成一个闭包，实例化一个私有的 `Dep` 实例进行该对象的依赖收集，

[更多>>>](https://github.com/7kyun/ghiblog/issues/5)

---


#### [Vue源码-双向数据绑定原理（三）](https://github.com/7kyun/ghiblog/issues/4) <sup>0 :speech_balloon:</sup> 	 2022-05-09 09:29:27

:label: : [:v:vue](https://github.com/7kyun/ghiblog/labels/%3Av%3Avue), [:scroll:源码解读](https://github.com/7kyun/ghiblog/labels/%3Ascroll%3A%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB)

### Observer类
Observer的作用就是遍历对象的所有属性将其进行双向绑定
如果是对象，则进行深度递归遍历，将每一个子对象都转化成响应式对象
如果是数组，则会对每一个成员都执行一遍 `observe` 方法，并对其原生的数组方法进行改写。
[相关源码](https://gith

[更多>>>](https://github.com/7kyun/ghiblog/issues/4)

---


#### [Vue源码-双向数据绑定原理（一）](https://github.com/7kyun/ghiblog/issues/3) <sup>0 :speech_balloon:</sup> 	 2022-05-09 08:10:59

:label: : [:v:vue](https://github.com/7kyun/ghiblog/labels/%3Av%3Avue), [:scroll:源码解读](https://github.com/7kyun/ghiblog/labels/%3Ascroll%3A%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB)

### initData
这段代码主要是初始化data中的数据，将数据进行Observer，监听数据的变化，其他的监视原理一致，这里以data为例
[相关源码](https://github.com/vuejs/vue/blob/2.6/src/core/instance/state.js#L1

[更多>>>](https://github.com/7kyun/ghiblog/issues/3)

---


## 分类  :card_file_box: 

<details open="open">
    <summary>
        <img src="assets/wordcloud.png" title="词云, 点击展开详细分类" alt="词云， 点击展开详细分类">
        <p align="center">:cloud: 词云 :cloud: <sub>点击词云展开详细分类:point_down: </sub></p>
    </summary>


<details>
<summary>:framed_picture:封面	<sup>1:newspaper:</sup></summary>

- [Cover](https://github.com/7kyun/ghiblog/issues/1)  <sup>1 :speech_balloon:</sup>  	 


</details>

<details>
<summary>:pencil2:随笔	<sup>1:newspaper:</sup></summary>

- [为什么会出现这个博客](https://github.com/7kyun/ghiblog/issues/6)  <sup>0 :speech_balloon:</sup>  	 


</details>

<details>
<summary>:pushpin:置顶	<sup>1:newspaper:</sup></summary>

- [为什么会出现这个博客](https://github.com/7kyun/ghiblog/issues/6)  <sup>0 :speech_balloon:</sup>  	 


</details>

<details>
<summary>:scroll:源码解读	<sup>4:newspaper:</sup></summary>

- [Vue源码-双向数据绑定原理（四）](https://github.com/7kyun/ghiblog/issues/7)  <sup>0 :speech_balloon:</sup>  	 
- [Vue源码-双向数据绑定原理（二）](https://github.com/7kyun/ghiblog/issues/5)  <sup>0 :speech_balloon:</sup>  	 
- [Vue源码-双向数据绑定原理（三）](https://github.com/7kyun/ghiblog/issues/4)  <sup>0 :speech_balloon:</sup>  	 
- [Vue源码-双向数据绑定原理（一）](https://github.com/7kyun/ghiblog/issues/3)  <sup>0 :speech_balloon:</sup>  	 


</details>

<details>
<summary>:thinking:思考	<sup>0:newspaper:</sup></summary>



</details>

<details>
<summary>:v:vue	<sup>4:newspaper:</sup></summary>

- [Vue源码-双向数据绑定原理（四）](https://github.com/7kyun/ghiblog/issues/7)  <sup>0 :speech_balloon:</sup>  	 
- [Vue源码-双向数据绑定原理（二）](https://github.com/7kyun/ghiblog/issues/5)  <sup>0 :speech_balloon:</sup>  	 
- [Vue源码-双向数据绑定原理（三）](https://github.com/7kyun/ghiblog/issues/4)  <sup>0 :speech_balloon:</sup>  	 
- [Vue源码-双向数据绑定原理（一）](https://github.com/7kyun/ghiblog/issues/3)  <sup>0 :speech_balloon:</sup>  	 


</details>


</details>    
