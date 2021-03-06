# Polymer 2.0 (5) Observers && Computed Properites
有两种监听器：
1. 简单监听器，只能监听单一的property
2. 复杂监听器：可以监听一到多个property

每个监听器都有一个或多个 _依赖_ ，当依赖发生可监听的变化是，监听方法就会被调用。


>Computed properties are virtual properties based on one or more pieces of the element's data. A
computed property is generated by a computing function—essentially, a complex observer that returns
a value.
计算属性顾名思义，是由一个返回某个值的计算函数算出来的属性。


### Observers and element initialization
只有当元素初始值加载完毕并且至少有一个依赖的属性被定义(`undefined => somevalue`)，监听器才会被调用。


### Observers are synchronous

1. 监听器的执行也是同步的
2. 如果监听器调用比较频繁，影响效率，可以使用[`Polymer.Async`](https://www.polymer-project.org/2.0/docs/api/namespaces/Polymer.Async)库来将它放到异步任务队列里面去。
3. Polymer不保证异步执行的监听器所传的参数是否是最新的


## Simple observers

简单监听器需要在`properties`中需要监听的property里面注册。监听函数里面不能依赖其他任何property

> Simple observers are fired the first time the property becomes defined (!= `undefined`), and on
every change thereafter, *_even if the property becomes undefined._*

简单监听器第一次触发在property被定义后(`undefined` => `somevalue`)，在此之后，任何property变化(包括重新变回`undefined`)都会被调用。

>Simple observers only fire when the property *itself* changes. They don't fire on subproperty
changes, or array mutation.

简单监听器只会响应当前property的可观察变化(observable changes)

>You specify an observer method by name. The host element must have a method with that name.

简单监听器的定义和实现(函数本身)必须一一对应,简单监听器的实现可以在当前class里面也可以继承自superclass也可以添加自mixin。


>The observer method receives the new and old values of the property as arguments.

简单监听器接受两个参数： `oldValue`,`newValue`

### Observe a property

```javascript
class XCustom extends Polymer.Element {

  static get is() {return 'x-custom'; }

  static get properties() {
    return {
      active: {
        type: Boolean,
        // Observer method identified by name
        observer: '_activeChanged'
      }
    }
  }

  // Observer method defined as a class method
  _activeChanged(newValue, oldValue) {
    this.toggleClass('highlight', newValue);
  }
}
```

## Complex observers

复杂监听器需要在`this.observers`数组里面注册，复杂监听器可以监听多条路径（依赖）

```javascript
static get observers() {
  return [
    // Observer method name, followed by a list of dependencies, in parenthesis
    'userListChanged(users.*, filter)'
  ]
}

```
注册函数里面的参数有如下几种：

*   一个简单的属性路径 (`firstName`).

*   一个简单的子属性路径 (`address.street`).

*   一个数组的变化结果路径 (`users.splices`).

*   一个包含通配符的路径 (`users.*`).


函数被调用时所得到的参数依据监听的路径种类不同而不同：


*   简单的属性或子属性，传参为新值
*   数组变化路径，传参为描述变化详情的 _变化记录_ 对象
*   通配符路径，传参为 _变化记录_ 对象以及变化的详细路径


>Note that any of the arguments can be `undefined` when the observer is called.

监听函数中每次调用，任何一个参数都有可能是`undefined`

复杂监听器的参数只有新值没有旧值


### Observe changes to multiple properties 

```js
class XCustom extends Polymer.Element {

  static get is() {return 'x-custom'; }

  static get properties() {
    return {
        preload: Boolean,
        src: String,
        size: String
    }
  }

  // Each item of observers array is a method name followed by
  // a comma-separated list of one or more dependencies.
  static get observers() {
    return [
        'updateImage(preload, src, size)'
    ]
  }

  // Each method referenced in observers must be defined in
  // element prototype. The arguments to the method are new value
  // of each dependency, and may be undefined.
  updateImage(preload, src, size) {
    // ... do work using dependent values
  }
}
```


### Observe array mutations

数组变化路径的参数change record是一个对象，包含一个`indexSplices`数组，数组中的每一项表示一处变更记录，包含下面信息：
   -   `index`. 变更其实的地方
   -   `removed`. 被删除的数据
   -   `addedCount`. 插入的数据长度
   -   `object`: 一个指向新数据的数组（不是拷贝）  //A reference to the array in question. 这句话不懂什么意思
   -   `type`: 一个值为'splice'的字符串


注意： change record也许会为`undefined`


```js
class XCustom extends Polymer.Element {

  static get is() {return 'x-custom'; }

  static get properties() {
    return {
      users: {
        type: Array,
        value: function() {
          return [];
        }
      }
    }
  }

  // Observe changes to the users array
  static get observers() {
    return [
      'usersAddedOrRemoved(users.splices)'
    ]
  }


  // For an array mutation dependency, the corresponding argument is a change record
  usersAddedOrRemoved(changeRecord) {
    if (changeRecord) {
      changeRecord.indexSplices.forEach(function(s) {
        s.removed.forEach(function(user) {
          console.log(user.name + ' was removed');
        });
        for (var i=0; i<s.addedCount; i++) {
          var index = s.index + i;
          var newUser = s.object[index];
          console.log('User ' + newUser.name + ' added at index ' + index);
        }
      }, this);
    }
  }

  ready() {
    super.ready();
    this.push('users', {name: "Jack Aubrey"});
  }
}

customElements.define(XCustom.is, XCustom);
```

### Observe all changes related to a path
通配符路径的参数是一个change record对象，包含：

*   `path`  监听器监听的路径
*   `value` 新值
*   `base`  通配符之前的路径所代表的对象

如果通配符路径监听到的是数组的变化，那么
1. `path`则是该数组的路径（不包括`.splices`）
2. `value`里面则会包含一个`indexSplices`项

```html
<dom-module id="x-custom">
  <template>
    <input value="{{user.name.first::input}}"
           placeholder="First Name">
    <input value="{{user.name.last::input}}"
           placeholder="Last Name">
  </template>
  <script>
    class XCustom extends Polymer.Element {

      static get is() { return 'x-custom'; }

      static get properties() {
        return {
          user: {
            type: Object,
            value: function() {
              return {'name':{}};
            }
          }
        }
      }

      static get observers() {
        return [
            'userNameChanged(user.name.*)'
        ]
      }

      userNameChanged(changeRecord) {
        console.log('path: ' + changeRecord.path);
        console.log('value: ' + changeRecord.value);
      }
    }

    customElements.define(XCustom.is, XCustom);
  </script>
</dom-module>
```


### Identify all dependencies

>Observers shouldn't rely on any properties, sub-properties, or paths other
than those listed as dependencies of the observer. This creates "hidden" dependencies

监听器不能依赖任何其他未被注册过的路径，否则：
1. 不能保证该路径是否已经初始化完成
2. 当该路径的属性发生变化时，无法触发当前监听器


For example:

```js
static get properties() {
  return {
    firstName: {
      type: String,
      observer: 'nameChanged'
    },
    lastName: {
      type: String
    }
  }
}

// WARNING: ANTI-PATTERN! DO NOT USE
nameChanged(newFirstName, oldFirstName) {
  // Not invoked when this.lastName changes
  var fullName = newFirstName + ' ' + this.lastName;
  // ...
}
```

>Note that Polymer doesn't guarantee that properties are
initialized in any particular order.

Polymer不能保证属性之间的初始化顺序。


## Computed properties

>Computed properties are virtual properties computed on the basis of one or more paths. The computing
function for a computed property follows the same rules as a complex observer, except that it
returns a value, which is used as the value of the computed property.

计算属性定义跟复杂监听器类似，但是计算属性的计算函数需要返回一个值。


### Define a computed property

计算属性需要在properties里面的property配置对象中使用computed键注册，注册语法跟复杂监听器一致。
```javascript
<dom-module id="x-custom">

  <template>
    My name is <span>{{fullName}}</span>
  </template>

  <script>
    class XCustom extends Polymer.Element {

      static get is() { return 'x-custom'; }

      static get properties() {
        return {
          first: String,

          last: String,

          fullName: {
            type: String,
            // when `first` or `last` changes `computeFullName` is called once
            // and the value it returns is stored as `fullName`
            computed: 'computeFullName(first, last)'
          }
        }
      }

      computeFullName(first, last) {
        return first + ' ' + last;
      }
    }

    customElements.define(XCustom.is, XCustom);
  </script>

</dom-module>
```

另外见 ： Computed Bindings


## Dynamic observer methods 

>If the observer method is declared in the `properties` object, the method is considered _dynamic_:
the method itself may change during runtime. A dynamic method is considered an extra dependency of
the observer, so the observer re-runs if the method itself changes. For example:

如果一个监听器或者计算属性的方法被定义在了`properties`里面，那么我们可以动态的对这个方法进行覆盖、重写。
当方法发生变化的时候，新的简单监听器或计算属性会被立即触发或者被更新。

```js
class NameCard extends Polymer.Element {

  static get is() {
    return 'name-card'
  }

  static get properties() {
    return {
      // Override default format by assigning a new formatter
      // function
      formatter: {
        type: Function
      },
      formattedName: {
        computed: 'formatter(name.title, name.first, name.last)'
      },
      name: {
        type: Object,
        value() {
         return { title: "", first: "", last: "" };
        }
      }
    }
  }

  constructor() {
    super();
    this.formatter = this.defaultFormatter;
  }

  defaultFormatter(title, first, last) {
    return `${title} ${first} ${last}`
  }
}
customElements.define('name-card', NameCard);
```

```js
nameCard.name = { title: 'Admiral', first: 'Grace', last: 'Hopper'}
console.log(nameCard.formattedName); // Admiral Grace Hopper
nameCard.formatter = function(title, first, last) {
  return `${last}, ${first}`
}
console.log(nameCard.formattedName); // Hopper, Grace
```
计算属性`formattedName`的方法`formatter`发生变化的时候,尽管依赖`name`没有变化，但是该属性还是触发更新了。

因为动态监听器方法出于properties里面，因此会被看作一个依赖，一旦这个方法被定义，监听器就在初始化的时候触发，尽管其他依赖都没有被定义。

## Add observers and computed properties dynamically

>In some cases, you may want to add an observer or computed property dynamically. A set of instance
methods allow you to add a simple observer, complex observer, or computed property to the current
element _instance_.

使用js API来动态添加计算属性或监听器
- `_createPropertyObserver`
- `_createMethodObserver` 
- `_createComputedProperty`


### Add a simple observer dynamically
```js
this._observedPropertyChanged = (newVal) => { console.log('observedProperty changed to ' + newVal); };
this._createPropertyObserver('observedProperty', '_observedPropertyChanged', true);
```
第三个参数代表这个方法(`_observedPropertyChanged`)是否应该被看作一个依赖


### Add a complex observer dynamically

```js
this._createMethodObserver('_observeSeveralProperties(prop1,prop2,prop3)', true);
```

第三个参数代表这个方法(`_observeSeveralProperties`)是否应该被看作一个依赖


### Add a computed property dynamically

```js
this._createComputedProperty('newProperty', '_computeNewProperty(prop1,prop2)', true);
```
第三个参数代表这个方法(`_computeNewProperty`)是否应该被看作一个依赖