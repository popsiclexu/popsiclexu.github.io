---
layout: post
title: FastJson 读取超大json文件引起OOM问题排查与解决
date: 2021-05-23T00:00:00.000Z
author: Xu Zhenxue
header-img: img/post-bg-universe.jpg
catalog: true
tags:
  - Java
---
> 滚动阅读全文

## 背景
最近工作有一个需求，需要读取一个约2GB的json文件（存储了约3千万个json对象的集合），解析其中的每个json对象，并进行一些数据转换，最后把转换后的json对象存储到es中。json文件格式大概是这样的：

```javascript
[
    {
        lng: 116.22
        lat: 22.00,
        count: xxxx
    },
    {
        lng: 116.22
        lat: 22.00,
        count: xxxx
    },
    ...
]
```
一次性将这个json文件加载到内存中进行json解析肯定是不行的，因此，我想到的是利用fastjson中的JSONReader流式读取文件逐步解析，快速看了一下JSONReader的API及demo，便开始coding，很快便写出了如下简化版的代码，fastjson 版本 1.2.70

```java
    private final static int BATCH_SIZE = 20000;
    public int importDataToEs(String jsonPath) {
        int count = 0;
        List<JSONObject> items = new ArrayList<>(BATCH_SIZE);
        try (Reader fileReader = new FileReader(jsonPath)) {
            JSONReader jsonReader = new JSONReader(fileReader);
            jsonReader.startArray();
            while (jsonReader.hasNext()) {
                JSONObject object = (JSONObject) jsonReader.readObject();
                JSONObject item = processObject(object);
                items.add(item);
                if (items.size() >= BATCH_SIZE) {
                    batchInsert(items);
                    count = count + items.size();
                    items.clear();
                }
            }
            if (items.size() > 0) {
                batchInsert(items);
                count = count + items.size();
            }
            jsonReader.endArray();
            jsonReader.close();
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```

刚开始时程序一切正常，大约读取了600万个数据后，程序抛出`java.lang.OutOfMemoryError:Java heap space`，通过`jvisualvm`可以看到内存随着程序的运行在逐渐增加，jstat命令也可以看到jvm中老年代内存在逐渐加载，最后抛出OOM，那肯定是存在某些对象无法回收造成的，于是便开始了问题的排查。

![](http://zhenxuexu.github.io/in-post/fastjson/内存分析.jpg)

## 问题排查过程

###  1. `jmap -dump:format=b,file=heapdump pid`：将内存使用的详细情况输出到文件

### 2. 使用MAT工具分析dump文件，查看内存中最多的对象实例

从dump文件（dump是在还没有达到OOM时导出的）中，可以看出存在大量HashMap对象，同时存在大量JSONObject对象。我们的程序虽然是批量插入数据，但是每次写完一批数据后都会将list清空，这部分JSONObject对象应该会被GC回收，按道理不应该存在那么多JSONObject实例。另外，我们知道JSONObject的底层实现是HashMap，所以HashMap的实例对象应该也和JSONObject未被回收有关，接下来我们来分析这些实例的GC Roots，看下是什么地方引用的JSONObject，导致JSONObject不能被回收。

![dump文件分析](http://zhenxuexu.github.io/in-post/fastjson/dump文件分析.jpg)

### 3. 分析HashMap实例的GC ROOT

在上图展示的视图中，选中hashmap实例，右键选择 `Path to GC Roots`，我们可以看到下图所示的GC ROOTS。

![](http://zhenxuexu.github.io/in-post/fastjson/引用分析.jpg)

由上图可以看出，并不是我们写的代码造成的JSONObject实例未被回收。主要持有`JSONObject`引用的是`DefaultJSONParser`中的`ParseContext`数组，现在问题算是找到了。

为了搞清楚是自己的使用不当，还是FastJson的Bug，于是看了一下FastJson这部分的代码，发现当前版本的FastJson中，readObject调用DefaultJsonParser中的parseObject方法，每解析一个JSONObject会创建一个context对象放到contextArray数组中，但是没有及时删除context数组中无用的对象，导致OOM。

## 解决方法

既然已经发现是FastJson造成的问题，那么我们很容易想到几种解决方案：

- 使用其他的JSON解析工具，比如Gson，但是我没有测过Gson是否存在这样的问题，感兴趣的朋友可以试一下；

- 升级FastJson，看该问题是否优化或修复

- 使用其他方法代替readObject，这也是本文使用的方法。将代码改为使用 startObject 将每行中的 key、value 单独解析，内存和CPU占用稳定无增长，问题解决。

  ```java
  private final static int BATCH_SIZE = 20000;
      public int importDataToEs(String jsonPath) {
          int count = 0;
          List<JSONObject> items = new ArrayList<>(BATCH_SIZE);
          try (Reader fileReader = new FileReader(jsonPath)) {
              JSONReader jsonReader = new JSONReader(fileReader);
              jsonReader.startArray();
              while (jsonReader.hasNext()) {
                  jsonReader.startObject();
                  JSONObject object = new JSONObject();
                  while (jsonReader.hasNext()) {
                      String key = jsonReader.readString();
                      if ("lng".equals(key)) {
                          object.put(key, jsonReader.readObject());
                      } else if ("lat".equals(key)) {
                          object.put(key, jsonReader.readObject());
                      } else if ("count".equals(key)) {
                          object.put(key, jsonReader.readObject());
                      }
                  }
                  jsonReader.endObject();
                  JSONObject item = processObject(object);
                  items.add(item);
                  if (items.size() >= BATCH_SIZE) {
                      batchInsert(items);
                      count = count + items.size();
                      items.clear();
                  }
              }
              if (items.size() > 0) {
                  batchInsert(items);
                  count = count + items.size();
              }
              jsonReader.endArray();
              jsonReader.close();
          } catch (FileNotFoundException e) {
              e.printStackTrace();
          } catch (IOException e) {
              e.printStackTrace();
          }
      }
  ```

  修改完代码后，可以看到程序运行过程中的内存在一定范围内波动，不会持续增加

  

![](http://zhenxuexu.github.io/in-post/fastjson/内存分析2.jpg)