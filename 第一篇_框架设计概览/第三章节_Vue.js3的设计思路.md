#第三章节 Vue.js3的设计思路

###3.1 声明式地描述UI

        Vue.js3是一个声明式的框架，也就是使用声明式地描述UI的，设计声明式框架前先了解下编写前端
    页面设计哪些内容：

        1. DOM元素： 例如div标签、a标签等元素标签。
        2. 属性： 例如a标签上的href、标签上的id、class等。
        3. 事件： 例如click、keydown等。
        4. 元素的层级结构： DOM树的层级结构、既有子节点又有父节点。

        Vue.js3描述上述内容的方案是：

        1. 使用与HTML标签一致的方式描述DOM元素：如描述div标签直接<div></div>。
        2. 使用与HTML标签一致的方式描述标签的属性： 如div标签中的id<div id='div'></div>。
        3. 使用v-bind或:描述动态绑定属性： 如div标签中的动态id<div :id='div'></div>或
            <div v-bind:id='div'></div>。
        4. 使用v-on或@来描述事件：例如描述点击事件<div @click='handler'></div>或
            <div v-on:click='handler'></div>。
        5. 使用与HTML标签一致的方式描述层级结构： 如div包含一个span子节点<div><span></span></div>。

        从上述中知道在Vue中DOM元素、属性、事件等都有相应的描述方式，不需要用户手写命令式代码，这
    就是声明式地描述UI

        除了使用上述地模板(组件中的<template>写法)来声明式地描述UI外，还可以使用JS对象描述，将
    <div v-on:click='handler'><span></span></div>用JS对象描述如：

        const title = {
            // 标签名称
            tag: 'h1',
            // 标签属性
            props: {
                onClick: handler
            },
            // 子节点
            children: {
                {tag: 'span'}
            }
        }

        使用JS对象描述UI的好处就是更灵活，假如描述一个h1-h6的标标签，用JS对象描述如：

        //h标签的等级
        let level = 3
        const title = {
            tag: `h${level}`
        }

        使用JS对象描述可以更改level来改变h标签。

        使用模板描述就需要依次手HTML代码如：
        <h1 v-if="level === 1"></h1>
        <h2 v-else-if="level === 2"></h2>
        <h3 v-else-if="level === 3"></h3>
        <h4 v-else-if="level === 4"></h4>
        <h5 v-else-if="level === 5"></h5>
        <h6 v-else-if="level === 6"></h6>

        可以看到模板描述远不如JS对象描述灵活，使用JS对象描述，就是所谓的虚拟DOM，也就是因为虚拟
    DOM的灵活性，Vue除了支持模板描述UI外，同时也支持虚拟DOM描述UI。

        平时组件中手写的渲染函数就是使用虚拟DOM描述UI，如下代码：

        import { h } from 'vue'
        export default {
            render() {
                return h('h1', {onClick: handler}) //虚拟DOM
            }
        }

        上述的h函数是辅助创建虚拟DOM的一个工具函数，调用后会返回一个对象，这个对象就是虚拟DOM。

###3.2 初识渲染器

        虚拟DOM的本质就是一个使用JS对象描述的真实DOM结构，虚拟DOM变成真实的DOM则需要通过渲染器。

        渲染器的作用就是将虚拟DOM渲染为真实DOM，过程如：

        h('h1', {onClick: handler})(虚拟DOM) ---> 渲染器 ---> 真实DOM

        实现一个简易的渲染器，如：
        function renderer (vnode, container) {
            // 使用vonde.tag作为标签名创建元素
            const el = document.createElement(vnode.tag)
            // 循环遍历vonde.props，将属性、事件添加到标签元素中
            for (const key in vnode.props) {
                //判断是否为事件
                if(/^on/.test(key)) {
                    el.addEventListener(
                        // 事件名称，如onClick --->click
                        key.substring(2).toLowerCase(),
                        // 处理事件
                        vnode.props[key]
                    )
                }
            }
            //处理children
            if(typeof vnode.children === 'string') {
                // 如果children是字符串，说明它是元素的文本节点
                el.appendChild(document.createTextNode(vnode.children))
            }else if(Array.isArray(vnode.children)){
                // 如果是数组，说明有子节点，递归调用函数渲染子节点,使用el作为挂载点
                vnode.children.forEach(child => renderer(child, el));        
            }
            //最后将元素添加到挂载节点中
            container.appendChild(el)
        }

        再构造一个虚拟DOM：

        const vnode = {
            tag: 'div',
            props: {
                onClick: () =>alert('hello')
            },
            children: 'click me'
        }
        // 调用renderer函数
        renderer(vnode, document.body) //将body作为挂载点

        一个简易的渲染器renderer的实现思路，总体分为三步：

        1. 创建元素：将tag作为元素名创建元素
        2. 给元素添加事件或属性：遍历props再进行判断，如果on开头的说明是事件，使用addEventListener
            绑定到元素中，注意将'on'截去再用toLowerCase转化为小写字符串。
        3. 处理处理children：如果是字符串，直接新创建文本插入标签元素，如果是数组则循环递归调用
            renderer，注意将刚刚的标签元素当成挂载点。

###3.3 组件的本质

        组件也可以使用虚拟DOM来描述，它并不是真实的DOM元素，组件的本质其实就是一组DOM元素的封装，如：

        const MyComponent = function() {
            return {
                tag: 'div',
                props: {
                  onClick: () => alert('hello')
                },
                children: 'click me'
            }
        }

        定义了一个名为MyComponent的函数，函数返回一个对象，该对象就是一个虚拟DOM，可以该函数作为
    tag，传到renderer函数里，如：

        const vnode = {
            tag: 'MyComponent'
        }
        
        为了渲染组件需要对前面的renderer函数做修改，如：

        function renderer(vnode, container) {
            if(typeof vnode.tag === 'string') {
                //如果vnode.tag为字符串，说明描述的是标签元素
                mountElement(vnode, container)
            }else if(typeof vnode.tag === 'function'){
                //如果vnode.tag为函数，说明描述的是组件
                mountComponent(vnode, container)
            }
        }

        其中mountElement函数就是上个章节的渲染器renderer函数，mountComponent函数的实现如下：

        function mountComponent(vnode, container) {
            //vnode.tag是函数，所有执行函数拿到返回的虚拟DOM
            const subtree = vnode.tag()
            //递归调用渲染器渲染subtree
            renderer(subtree, container)
        }

        因为是用将虚拟DOM对象封装到一个函数中，因此vnode.tag将是个函数，将函数执行拿到虚拟DOM，然后
    再递归渲染subtree。

        其实组件不一定得是函数，也可以是一个对象：

        const MyComponent = {
            render() {
                ...  //虚拟DOM对象
            }
        }

        这里使用了一个对象代表组件，该对象有一个函数render，它会返回需要渲染内容的虚拟DOM，为了适配对
    象类型的组件，需要对渲染器进行调整。

        //在renderer函数中将typeof vnode.tag === 'function'改为
        if(typeof vnode.tag === 'object')
        //
        //在mountComponent函数中将const subtree = vnode.tag()该为const subtree = vnode.tag.render()
        const subtree = vnode.tag.render()

        其实可以将组件的本质当作比较复杂的虚拟DOM，无论是将组件封装成什么形式的，只要能能拿到该虚拟
    DOM然后将它放入渲染器中即可。

###3.4 模板的工作原理

        手写虚拟DOM或者是模板，都属于声明式地描述UI，在Vue中虚拟DOM是需要渲染器支持才能渲染成真实DOM,
    而模板需要编译器才能正常的工作。

        编译器与渲染器一样,只是一段程序,不过它们的工作内容不一样,编译器作用是将模板编译成渲染函数,如:

        <div @click="handler">
           click me
        </div>

        对于编译器来说,模板就是普通的字符串,它会分析该字符串生成一个渲染函数,如:

        render() {
            return return h('div', {onClick: handler}, 'click me')
        }

        在组件.vue文件中,如下所示:

        <template>
            <div @click="handler">
                click me
            </div>
        </template>
        <script>
            export default {
                data() {/*...*/},
                methods: {
                     handler(){/*...*/}
               },
            }
        </script>

        其中<template>内容就是模板内容,编译器会把模板内容编译成渲染函数并添加到<script>标签块的组件
    对象上,最终在浏览器内运行的是:

        export default {
            data() {/*...*/},
            methods: {
                handler(){/*...*/},
            },
            render() {
                return return h('div', {onClick: handler}, 'click me')
            }
        }

        无论是使用模板还是手写渲染函数,对于组件来说,它要渲染的内容最终都是通过渲染函数产生的,然后渲染
    器再将返回的虚拟DOM渲染成真实的DOM,这就是模板的工作原理,也就是Vue渲染页面的流程。

###3.5 Vue.js是个个模块组成的有机整体

        组件的实现依赖渲染器;模板实现依赖编译器，并且编译后生成代码是根据渲染器和虚拟DOM的设计决定的，
    因此Vue的个个模块间是相互关联、相互制约的，共同构成一个有机整体。

        假设有如下模板：

        <div id="foo" :class="cls"></div>

        编译器会将这段代码编译成渲染函数：

        // 下面代码等价于：return h('div', {id: 'foo', class: 'cls'}),因为它们返回同样的虚拟DOM
        render() {
            return {
                tag: 'div',
                props: {
                    id: 'foo',
                    :class: cls
                }
            }
        }

        这段代码中cls并不是字符串，而是一个变量，它可能会发生变化。渲染器的一个重要作用是寻找并且只更
    新变化的内容(寻找使用了diff算法)，而寻找肯定是需要寻找差异性能消耗的。其实从编译器角度看，它是能在编译的过程中分析出哪些是存在可能变化的动态内容的。

        假设编译器在编译过程中分析到class是一个动态内容，那么在生成虚拟DOM的时候附带一个标识信息，如
    patchFlags：1代表class是动态的，如下代码：
        
        render() {
            return {
                tag: 'div',
                props: {
                    id: 'foo',
                    :class: cls
                },
                patchFlags：1
            }
        }

        当渲染器发现标识就可以知道这是一个动态内容，就相当于省去了寻找变更点的工作量，同时性能也会得到
    提升。

        从这可以看出编译器和渲染器是存在信息交流的，它们之间的交流媒介就是虚拟DOM。