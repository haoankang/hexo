---
title: 踩坑系列——JSONArray引发的并发问题
date: 2019-12-04 11:40:33
categories: 踩坑系列
tags: JSONArray，并发
---

##1.问题描述
最近开发一个造数功能模块，因为有递增和递减，所以需要生成的数据有序；这里用到多线程，多线程相关的核心逻辑如下：
```java
//造数逻辑
public JSONArray buildDataCore(JSONArray datalogic, Integer num) throws Exception {
    JSONObject result = new JSONObject();
    JSONArray columns = new JSONArray();

    //每次造数num条记录
    JSONArray data = new JSONArray(num);
    long beginTime = System.currentTimeMillis();
    final CountDownLatch countDownLatch = new CountDownLatch(num);
    AtomicInteger count = new AtomicInteger(0);

    CopyOnWriteArrayList list = new CopyOnWriteArrayList(new Object[num]);

    while(count.intValue()<num){
        BaseService.executor.execute(new BuildDataThread(count.intValue(), datalogic, list, columns, countDownLatch));
        count.incrementAndGet();
    }

    countDownLatch.await();
    long endTime = System.currentTimeMillis();
    data.addAll(list);
    log.info("造数时间："+(endTime-beginTime));
    return data;
}

class BuildDataThread implements Runnable{

        private int count;
        private JSONArray datalogic;
        private CopyOnWriteArrayList data;
        private JSONArray columns;
        private CountDownLatch countDownLatch;

        public BuildDataThread(int count, JSONArray datalogic, CopyOnWriteArrayList data, JSONArray columns, CountDownLatch countDownLatch){
            this.count = count;
            this.datalogic = datalogic;
            this.data = data;
            this.columns = columns;
            this.countDownLatch = countDownLatch;
        }

        @Override
        public void run() {
            JSONObject data_child = new JSONObject();
            for(int n=0;n<datalogic.size();n++){
                JSONObject jsonObject = datalogic.getJSONObject(n);
                DataType dateType = DataType.getDateType(jsonObject.getString("type"));
                Object object = null;
                try {
                    object = BuildDataUtil.buildData(jsonObject.getJSONObject("logic"), dateType, count);
                } catch (Exception e) {
                    log.error("造数出错：",e);
                }
//                if(count==0){
//                    columns.add(jsonObject.getString("name"));
//                }

                data_child.put(jsonObject.getString("name"),object);
            }
            data.set(count,data_child);

            countDownLatch.countDown();
        }
    }
```
一开始是把JSONArray直接作为入参，希望通过jsonArray.set(index, object)合并结果，当数据量
比较大时，必定会出现一些结果是null的情况，且不会报错，通过打印日志也可以确定造数逻辑没问题，每次必定返回结果，
不会生成null；

##2. 问题分析.
程序没有报错且造数方法每次返回正常结果，所以可以确认程序没问题，JSONArray的set方法单线程下也可以确认没问题，
可以得到我们期望的执行结果；所以这里问题就可以定位到是多线程环境下JSONArray的set方法出问题了；源码如下：
```java
public JSONArray() {
    this.list = new ArrayList();
}
    
    
public Object set(int index, Object element) {
    if (index == -1) {
        this.list.add(element);
        return null;
    } else if (this.list.size() > index) {
        return this.list.set(index, element);
    } else {
        for(int i = this.list.size(); i < index; ++i) {
            this.list.add((Object)null);
        }

        this.list.add(element);
        return null;
    }
}
```
JSONArray底层默认用ArrayList，list的size只能通过增删个体改变，无法指定，虽然ArrayList有默认构造方法
public ArrayList(int initialCapacity)，但是这个是容器长度，用于扩容等，并不是size；因此，当调用
ArrayList.set(index, object)这个方法时，假如有线程1填充JSONArray[1]和线程2填充JSONArray[2]，线程1
执行到this.list.add(element)这部分了，线程2正好执行到else{...}子句，就会把线程1的结果覆盖为null；

##3. 解决方案及问题衍生.
解决思路就如前面代码所示，用一个线程安全的CopyOnWriteArrayList代替JSONArray；

一般并发编程时，一定要注意使用的使用的集合和变量是否有线程安全问题；