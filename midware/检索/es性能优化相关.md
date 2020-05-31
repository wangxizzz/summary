# ES批量并发写入压测结果：
## 优化读
多线程批量写入，对于只有两列的es表，多线程批量写入性能实测如下：

20个线程，每次插入2000条，现在跑插入1000w数据，7.25分钟  
20个线程，每次插入1000条，现在跑插入1000w数据，6.8分钟  
30个线程，每次插入1000条，现在跑插入1000w数据，7.58分钟  
10个线程，每次插入1000条，现在跑插入1000w数据，8.2分钟  
10个线程，每次插入2000条，10w数据  13s

代码如下：
```java
public boolean data2() throws IOException {
   long start = System.currentTimeMillis();
   ExecutorService executorService = Executors.newFixedThreadPool(10);
 
   int count = 50;
   final CountDownLatch latch = new CountDownLatch(count);
   for (int n = 0; n < count; n++) {
      executorService.submit(new Runnable() {
         @Override
         public void run() {
            BulkRequest request = new BulkRequest();
            try {
               for (int i = 0; i < 2000; i++) {
                  Map<String, Object> docMap = new HashedMap();
                  docMap.put("crow_id", 100);
                  docMap.put("user_id", 101);
                  request.add(new IndexRequest(INDEX1).id(UUID.randomUUID().toString())
                        .source(JSON.toJSONString(docMap), XContentType.JSON));
               }
               BulkResponse responses = crowdEsTemplate.bulk(request, RequestOptions.DEFAULT);
               if (responses.hasFailures()) {
                  return;
               }
            } catch (Exception e) {
               log.warn("---", e);
            } finally {
               latch.countDown();
            }
         }
      });
   }
   try {
      latch.await();
   } catch (InterruptedException e) {
      e.printStackTrace();
   }
   log.info("========end:{}", (System.currentTimeMillis() - start) / 1000);
   return true;
}
```

# ES批量并发scroll读压测结果：
## 优化写：
目前大数据量的读取一般采用scroll方式读取，但每次生成下一次的scroll的过程很耗时，大约每次需要2~3s, scroll的步长最多设置1w,这样如果读取一千万数据，光生成scroll就需要 1000 *(2~3s)

根据userId(ES主键)来做数据分片，而且可以尽量使得数据均匀。先查min(UserId)和max(UserId),然后将max(UserId)与min(UserId)分为很多小区间，比如分为1000个小区间。然后，把这些小区间乱序打散，再次将乱序打散的小区间分成10组。。。1组就是一个数据分片。然后采用多线程并发读取这10组数据



