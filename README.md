# 基于uni-app的缓存器

在前端开发应用程序中，性能一直都是被大家所重视的一点，然而判断一个应用程序的性能最直观的就是看页面打开的速度。其中提高页页面反应速度的一个方式就是使用缓存。一个优秀的缓存策略可以缩短页面请求资源的距离，减少延迟，并且由于缓存文件可以重复利用，还可以减少带宽，降低网络负荷。

前端常用缓存技术在这里我就不再描述，下面基于Storage对其进行增强，采用Map 基本相同的api。

下面是基本代码，会在此基础上进行增强

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

我们知道缓存往往是有危害的，那么我们最好规定个时间来去除数据。

```
class CacheCell {
  constructor (data, timeout) {
    this.data = data
    // 设置超时时间
    this.timeout = timeout
    // 对象创建时候的时间
    this.createTime = Date.now()
  }
}
```

```

```

