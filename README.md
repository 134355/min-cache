# 基于uni-app的缓存器

在前端开发应用程序中，性能一直都是被大家所重视的一点，然而判断一个应用程序的性能最直观的就是看页面打开的速度。其中提高页页面反应速度的一个方式就是使用缓存。一个优秀的缓存策略可以缩短页面请求资源的距离，减少延迟，并且由于缓存文件可以重复利用，还可以减少带宽，降低网络负荷。

前端常用缓存技术在这里我就不再描述，下面基于Storage对其进行增强，采用Map 基本相同的api。

阅读以下内容时遇到不懂的，请先科普阮一峰老师的[ECMAScript 6 入门](<http://es6.ruanyifeng.com/>)

## 下面是基本代码，会在此基础上进行增强

```
class MinCache {
  // 将数据存储在本地缓存中指定的 name 中
  set (name, data) {
    try {
      uni.setStorageSync(name, data)
    } catch (e) {
      console.log(e)
    }
  }
  // 从本地缓存中获取指定 name 对应的内容
  get (name) {
    let data
    try {
      data = uni.getStorageSync(name)
    } catch (e) {
      console.log(e)
    }
    return data
  }
  // 从本地缓存中移除指定 key
  delete (name) {
    try {
      uni.removeStorageSync(name)
    } catch (e) {
      console.log(e)
    }
  }
  // 返回一个布尔值，表示 name 是否在本地缓存之中
  has (name) {
    const value
    try {
      const res = uni.getStorageInfoSync()
      value = res.keys.includes(name)
    } catch (e) {
      console.log(e)
    }
    return value
  }
  // 清理本地数据缓存
  clear () {
    try {
      uni.clearStorageSync()
    } catch (e) {
      console.log(e)
    }
  }
}

export default MinCache
```

## 我们知道缓存往往是有危害的，那么我们最好规定个时间来去除数据。

```
class CacheCell {
  constructor (data, timeout) {
    this.data = data
    // 设置超时时间，单位秒
    this.timeout = timeout
    // 对象创建时候的时间
    this.createTime = Date.now()
  }
}
```

```
  set (name, data, timeout = 1200) {
    const cachecell = new CacheCell(data, timeout)
    try {
      uni.setStorageSync(name, cachecell)
    } catch (e) {
      console.log(e)
    }
  }
  get (name) {
    let data = null
    try {
      data = uni.getStorageSync(name)
      if (!data) return null
      const currentTime = Date.now()
      const overTime = (currentTime - data.createTime) / 1000
      if (overTime > data.timeout) {
        try {
          uni.removeStorageSync(name)
          data = null
        } catch (e) {
          console.log(e)
        }
      }
    } catch (e) {
      console.log(e)
    }
    return data
  }
```

使用了过期时间进行缓存的方式，已经可以满足绝大部分的业务场景。

uni-app的Storage在不同端的实现不同：

- H5端为localStorage，浏览器限制5M大小，是缓存概念，可能会被清理
- App端为原生的plus.storage，无大小限制，不是缓存，持久化
- 各个小程序端为其自带的storage api，数据存储生命周期跟小程序本身一致，即除用户主动删除或超过一定时间被自动清理，否则数据都一直可用。
- 微信小程序单个 key 允许存储的最大数据长度为 1MB，所有数据存储上限为 10MB。
- 支付宝小程序单条数据转换成字符串后，字符串长度最大200*1024。同一个支付宝用户，同一个小程序缓存总上限为10MB。
- 百度、头条小程序文档未说明大小限制

除此之外，H5端还支持websql、indexedDB、sessionStorage；App端还支持[SQLite](https://www.html5plus.org/doc/zh_cn/sqlite.html)、[IO文件](https://www.html5plus.org/doc/zh_cn/io.html)等本地存储方案。

我们可以看出来Storage在一些端中是有大小限制的，其实我们的数据只是想要缓存，不一定要持久化。

也就是说在应用程序生命周期内使用就行，而且直接操作Storage也不是很好。

我们知道ES6中有Map可以做缓存

## 下面代码时基于Map封装的

```
let cacheMap =  new Map()
let instance = null
let timeoutDefault = 1200

function isTimeout (name) {
  const data = cacheMap.get(name)
  if (!data) return true
  if (data.timeout === 0) return false
  const currentTime = Date.now()
  const overTime = (currentTime - data.createTime) / 1000
  if (overTime > data.timeout) {
    cacheMap.delete(name)
    return true
  }
  return false
}

class CacheCell {
  constructor (data, timeout) {
    this.data = data
    this.timeout = timeout
    this.createTime = Date.now()
  }
}

class Cache {
  set (name, data, timeout = timeoutDefault) {
    const cachecell = new CacheCell(data, timeout)
    return cacheMap.set(name, cachecell)
  }
  get (name) {
    return isTimeout(name) ? null : cacheMap.get(name).data
  }
  delete (name) {
    return cacheMap.delete(name)
  }
  has (name) {
    return !isTimeout(name)
  }
  clear () {
    return cacheMap.clear()
  }
  setTimeoutDefault (num) {
    if (timeoutDefault === 1200) {
      return timeoutDefault = num
    }
    throw Error('缓存器只能设置一次默认过期时间')
  }
}

class ProxyCache {
  constructor () {
    return instance || (instance = new Cache())
  }
}

export default ProxyCache
```

## 对Storage和Map封装的缓存进行整合

我们来分析一下

- Storage和Map共用一套api
  - 在命名上解决以下划线_开头命名的缓存到Storage，并且Map也有副本
- 尽量不操作Storage(读取速度慢)
  - 那就必须在应用程序初始化的时候把Storage加载进Map
- 像Vue插件一样使用

```
let cacheMap =  new Map()
let timeoutDefault = 1200

function isTimeout (name) {
  const data = cacheMap.get(name)
  if (!data) return true
  if (data.timeout === 0) return false 
  const currentTime = Date.now()
  const overTime = (currentTime - data.createTime) / 1000
  if (overTime > data.timeout) {
    cacheMap.delete(name)
    if (name.startsWith('_')) {
      try {
        uni.removeStorageSync(name)
      } catch (e) {
        console.log(e)
      }
    }
    return true
  }
  return false
}

class CacheCell {
  constructor (data, timeout) {
    this.data = data
    this.timeout = timeout
    this.createTime = Date.now()
  }
}

class MinCache {
  constructor (timeout) {
    try {
      const res = uni.getStorageInfoSync()
      res.keys.forEach(name => {
        try {
          const value = uni.getStorageSync(name)
          cacheMap.set(name, value)
        } catch (e) {
          console.log(e)
        }
      })
    } catch (e) {
      console.log(e)
    }
    timeoutDefault = timeout
  }
  set (name, data, timeout = timeoutDefault) {
    const cachecell = new CacheCell(data, timeout)
    let cache = null
    if (name.startsWith('_')) {
      try {
        uni.setStorageSync(name, cachecell)
        cache = cacheMap.set(name, cachecell)
      } catch (e) {
        console.log(e)
      }
    } else {
      cache = cacheMap.set(name, cachecell)
    }
    return cache
  }
  get (name) {
    return isTimeout(name) ? null : cacheMap.get(name).data
  }
  delete (name) {
    let value = false
    if (name.startsWith('_')) {
      try {
        uni.removeStorageSync(name)
        value = cacheMap.delete(name)
      } catch (e) {
        console.log(e)
      }
    } else {
      value = cacheMap.delete(name)
    }
    return value
  }
  has (name) {
    return !isTimeout(name)
  }
  clear () {
    let value = false
    try {
      uni.clearStorageSync()
      cacheMap.clear()
      value = true
    } catch (e) {
      console.log(e)
    }
    return value
  }
}

MinCache.install = function (Vue, {timeout = 1200} = {}) {
  Vue.prototype.$cache = new MinCache(timeout)
}

export default MinCache
```

## 使用方法

name以下划线_开头命名的缓存到Storage，并且Map也有副本

| 事件名 | 参数                                                         | 说明                         | 返回值             |
| ------ | ------------------------------------------------------------ | ---------------------------- | ------------------ |
| set    | name缓存的key,data缓存的数据,timeout(必须数字单位s)缓存时间，默认缓存1200s, timeout设置为0表示永久缓存 | 设置缓存数据                 | Map集合            |
| get    | name缓存的key                                                | 获取数据(缓存过期将返回null) | 返回缓存的数据data |
| has    | name缓存的key                                                | 检查值                       | true/false         |
| delete | name缓存的key                                                | 删除数据                     | true/false         |
| clear  | -                                                            | 清空Storage和Map缓存数据     | true/false         |

```
// 注册缓存器
Vue.use(MinCache)
// 设置默认缓存时间
// Vue.use(MinCache, {timeout: 600})
```

```
// 'name'不是以下划线开头的表示会缓存到Map中，在程序生命周期内有并且在有效时间内有效
this.$cache.set('name', 'MinCache')

// 过期时间设置为0表示不会过期
// 注意：'test'并不是以下划线命名表示在程序生命周期永久缓存
this.$cache.set('test', 'testdemo', 0)

// 过期时间设置为0表示不会过期
// 注意：'_imgURL'是以下划线命名表示永久缓存到Storage
this.$cache.set('_imgURL', 'data', 0)
```

```
// 获取缓存的数据
this.imgURL = this.$cache.get('_imgURL')
this.name = this.$cache.get('name')
this.test = this.$cache.get('test')
```



具体使用方法可以参考[github](https://github.com/134355/min-cache.git)

