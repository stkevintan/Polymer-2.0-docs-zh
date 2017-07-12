# Polymer 2.0 (8) Thinking

1. react是在js中写html，不符合日常直觉。而Polymer直接以html为单位，通过浏览器自带的shadowDOM做组件间的隔离。在html里面写js、css，比较自然。
2. Polymer的数据绑定系统受Angular影响太大。既包含双向绑定又包含单向绑定，但是限于浏览器兼容条件很多情况下只能双向绑定、只能单向绑定。坑点较多。不如React的单向绑定简洁明了。
3. 部分的cssNext语法支持，不能扩展`Native Element`,不开心。
4. 对于Polymer 1.x 来说，语言升级变化太大，导致兼容太繁琐。一个框架里面组件有三套写法，还有两套不同的生命周期回调函数。混乱。
5. 基于路径的事件监听方式，只能说很新颖，`Array.splice`可以直接把数组变动传给监听函数，感觉还不错，但是`linkPath`就有点徒增坑点了。