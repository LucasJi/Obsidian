## 业务背景

现有一音频标注系统，该系统可以通过**任务**的方式安排标注人员对音频进行标注。任务的创建需要选定某个**数据集**（一个数据集通常包含几十万条**音频数据**），从该数据集中选择指定数量的音频数据进行后续标注。

数据集和任务的关系是一对多，即一个数据集可以被多个任务使用。由于某些任务已经安排标注人员进行标注，这些任务对应的数据集中的部分数据的状态就变成了“已标注”。当使用这些数据集新建任务时，需要过滤这些已标注的数据（不需要安排标注人员再次进行标注了），只统计未标注的数据。

## 现有问题

新建任务获取数据集列表的接口耗时太久——7s 左右。

## 问题分析

获取数据集列表的时候需要统计每个数据集中未分配数据的数量，主要的时间应该就花费在“统计”操作上。

### 旧业务代码

```java
// 1. 数据集列表  
List<DataSetDTO> dataSets = getDataSets();  
  
// 2. 数据集中所有音频数据  
AggregateIterable<Document> documents = mongoTemplate.getCollection("data_log").aggregate(aggregateCond).allowDiskUse(true);  
  
// 3. 统计每个数据集可分配音频数据数  
try (MongoCursor<Document> iterator = documents.iterator()) {  
  ArrayList<DataLog> dataLogs = new ArrayList<>();  
  
  while (iterator.hasNext()) {  
    Document document = iterator.next();  
    DataLog dataLog = documentToDataLogDTO(document);  
    dataLogs.add(dataLog);  
  }  
  
  // 使用过的音频数据  
  List<DataLog> usedDataLogs =  
      dataLogs.stream().filter(e -> e.getIsUse().equals(1)).collect(Collectors.toList());  
  
  dataSets.forEach(  
      dataSet -> {  
        // 音频总数  
        long count =  
            dataLogs.stream().filter(e -> e.getDataSetId().equals(dataSet.getId())).count();  
        dataSet.setAllCount((int) count);  
        // 已分配数  
        long useCount =  
            usedDataLogs.stream().filter(e -> e.getDataSetId().equals(dataSet.getId())).count();  
        dataSet.setMarkedDataCount((int) useCount);  
        // 可分配数  
        dataSet.setAvailableCount((int) count - (int) useCount);  
      });  
}
```

上述代码的逻辑是先将所有的音频数据存入一个集合中，再针对每个数据集过滤出该数据集对应的音频数据，最后再统计可分配的音频数据数。不难看出该逻辑存在很多的冗余操作，比如使用游标遍历文档的时候其实可以同时进行数据统计操作了，没有必要等遍历完成之后，再针对每个数据集重复过滤遍历。

## 去除冗余遍历操作进行改进

```Java
// 1. 数据集列表  
List<DataSetDTO> dataSets = getDataSets();  
  
// 2. 数据集中所有音频数据  
AggregateIterable<Document> documents =  
    mongoTemplate.getCollection("data_log").aggregate(aggregateCond).allowDiskUse(true);
    
// 3. 数据集id和对象的映射，方便后续快速获取数据集进行数据统计  
Map<Integer, DataSetDTO> dataSetIdObjMap = new LinkedHashMap<>();  
for (DataSetDTO dataSet : dataSets) {  
  dataSetIdObjMap.putIfAbsent(dataSet.getId(), dataSet);  
}  
  
// 4. 统计每个数据集可分配音频数据数  
try (MongoCursor<Document> iterator = documents.iterator()) {  
  while (iterator.hasNext()) {  
    Document document = iterator.next();  
    DataLog dataLog = documentToDataLogDTO(document);  
  
    Integer dataSetId = dataLog.getDataSetId();  
    if (dataSetIdObjMap.containsKey(dataSetId)) {  
      DataSetDTO dataSet = dataSetIdObjMap.get(dataSetId);  
      // 4.1 初始化数据  
      if (Objects.isNull(dataSet.getAllCount())) {  
        dataSet.setAllCount(0);  
      }  
  
      if (Objects.isNull(dataSet.getMarkedDataCount())) {  
        dataSet.setMarkedDataCount(0);  
      }  
  
      // 4.2 统计样本数  
      dataSet.setAllCount(dataSet.getAllCount() + 1);  
  
      // 4.3 统计已分配数  
      if (Objects.equals(1, dataLog.getIsUse())) {  
        dataSet.setMarkedDataCount(dataSet.getMarkedDataCount() + 1);  
      }  
  
      // 4.4 统计可分配数  
      dataSet.setAvailableCount(dataSet.getAllCount() - dataSet.getMarkedDataCount());  
    }  
  }  
}
```

优化后的代码去除了冗余的遍历操作，在通过游标遍历音频数据的时候同时进行可分配数的统计操作。优化后的代码耗时约为 3s 左右。

## 使用 MongoDB 聚合管道进行改进

在旧业务代码和改进后的业务代码中，统计操作都是通过 Java 代码另外执行的。既然查询音频文档的时候已经使用了聚合管道，统计时又没有额外业务逻辑，何不讲统计操作也加入管道中一并执行呢？

### 旧管道聚合表达式

```
db.getCollection("data_log").aggregate([{
    $match: {
        dataSetId: {
            $in: [
                473
            ]
        }
    }
}, {
    $project: {
        dataSetId: 1,
        isUse: 1
    }
}])
```

上述表达式包含两个管道操作，`$match` 操作只查询给定数据集的音频数据，`$project` 操作过滤无用字段，减少数据传输，加快查询速度。

### 统计表达式

```
$group: {
        _id: "$dataSetId",
        allCount: {
            $sum: 1
        },
        usedCount: {
            $sum: {
                $cond: [{
                    $eq: ["$isUse", 1]
                }, 1, 0]
            }
        }
    }
```

该表达式通过 `$group` 操作对音频数据按照数据集 id 进行分组，分组之后使用 `$sum` 累加器可以轻松获取所有音频数据数和已分配音频数据数。

### 改进后的管道聚合表达式

```
db.getCollection("data_log").aggregate([{
    $match: {
        dataSetId: {
            $in: [
                473
            ]
        }
    }
}, {
    $project: {
        dataSetId: 1,
        isUse: 1
    }
}, {
    $group: {
        _id: "$dataSetId",
        allCount: {
            $sum: 1
        },
        markedDataCount: {
            $sum: {
                $cond: [{
                    $eq: ["$isUse", 1]
                }, 1, 0]
            }
        }
    }
}])
```

### 完整的业务代码

```Java
// 1. 数据集列表  
List<DataSetDTO> dataSets = getDataSets();  
  
// 2. 数据集中所有音频数据  
AggregateIterable<Document> documents =  
    mongoTemplate.getCollection("data_log").aggregate(aggregateCond).allowDiskUse(true);  
  
// 3. 数据集id和对象的映射，方便后续快速获取数据集进行数据统计  
Map<Integer, DataSetDTO> dataSetIdObjMap = new LinkedHashMap<>();  
for (DataSetDTO dataSet : dataSets) {  
  dataSetIdObjMap.putIfAbsent(dataSet.getId(), dataSet);  
}  
  
// 4. 统计每个数据集可分配音频数据数  
try (MongoCursor<Document> iterator = documents.iterator()) {  
  while (iterator.hasNext()) {  
    Document document = iterator.next();  
    Integer dataSetId = document.getInteger("_id");  
  
    if (dataSetIdObjMap.containsKey(dataSetId)) {  
      DataSetDTO dataSet = dataSetIdObjMap.get(dataSetId);  
  
      // 样本数  
      dataSet.setAllCount(document.getInteger("allCount"));  
  
      // 已分配数  
      dataSet.setMarkedDataCount(document.getInteger("usedCount"));  
  
      // 可分配数  
      dataSet.setAvailableCount(dataSet.getAllCount() - dataSet.getMarkedDataCount());  
    }  
  }  
}
```

### 改进之后耗时统计

改进之后的耗时相比于代码层面的改进有了进一步的降低，达 2s 左右。
