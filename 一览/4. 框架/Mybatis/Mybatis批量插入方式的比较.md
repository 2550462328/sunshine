对比3种可用的方式

1. 反复执行单条插入语句
2. xml拼接sql
3. 批处理执行

先说结论：少量插入请使用反复插入单条数据，方便。数量较多请使用批处理方式。（可以考虑以有需求的插入数据量20条左右为界吧，在我的测试和数据库环境下耗时都是百毫秒级的，方便最重要）。**无论何时都不用xml拼接sql的方式**。



**测试方法**（省略其他代码）：

```
@Service
public class ItemService {
    @Autowired
    private ItemMapper itemMapper;
    @Autowired
    private SqlSessionFactory sqlSessionFactory;
    //批处理
    @Transactional
    public void add(List<Item> itemList) {
        SqlSession session = sqlSessionFactory.openSession(ExecutorType.BATCH,false);
        ItemMapper mapper = session.getMapper(ItemMapper.class);
        for (int i = 0; i < itemList.size(); i++) {
            mapper.insertSelective(itemList.get(i));
            if(i%1000==999){//每1000条提交一次防止内存溢出
                session.commit();
                session.clearCache();
            }
        }
        session.commit();
        session.clearCache();
    }
    //拼接sql
    @Transactional
    public void add1(List<Item> itemList) {
        itemList.insertByBatch(itemMapper::insertSelective);
    }
    //循环插入
    @Transactional
    public void add2(List<Item> itemList) {
        itemList.forEach(itemMapper::insertSelective);
    }
}
```



**测试结果：**

10条 25条数据插入经多次测试，波动性较大，但基本都在百毫秒级别

| 方式         | 50条   | 100条  | 500条  | 1000条  |
| ------------ | ------ | ------ | ------ | ------- |
| 批处理       | 159ms  | 208ms  | 305ms  | 432ms   |
| xml拼接sql   | 208ms  | 232ms  | 报错   | 报错    |
| 反复单条插入 | 1013ms | 2266ms | 8141ms | 18861ms |

其中 拼接sql方式在插入500条和1000条时报错（似乎是因为sql语句过长，此条跟数据库类型有关，未做其他数据库的测试）



可以发现

- 循环插入的时间复杂度是 O(n),并且常数C很大
- 拼接SQL插入的时间复杂度（应该）是 O(logn),但是成功完成次数不多，不确定
- 批处理的效率的时间复杂度是 O(logn),并且常数C也比较小

