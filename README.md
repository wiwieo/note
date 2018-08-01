# 问题记录
**一、GO**

1、日期格式必须为：`2006-01-02 15:04:05`

2、在将json转对象时，如果为数值型，则只能转为float型

3、[]interface{}不能直接断言为其他对象，需要循环转换

4、避免对map进行并发处理。可以使用Go1.9的sync.Map

5、将空指针传给interface{}后，当前interface{}并不是空，而是指向空的对象

**二、框架（gorilla、sqlx）**

1、在使用sqlx映射获取json数据时，获取的是[]unit8类型，需要再转换一次

2、如果数据库中字段值是null，直接通过sqlx框架映射到struct时会报错。解决办法为，数据库中设置默认值，不能为空。


**三、数据库**

1、数据库连接后，连接没有释放，导致服务不可用。
```
handle, err := dbIns.Prepare("CALL sp_withdraw_operate (?,?,?,?,?,?,?,?)")
if err != nil {
	return 0, err
}
defer handle.Close() // 必须释放
//call procedure
rst, err := handle.Query(0, applyAccountID, WithdrawModeBankTransfer, amount, WithdrawStatusApplying, desc, "", "")
defer rst.Close() // 必须释放
```

2、mysql中，如果想将时间戳格式默认值设为`0000-00-00 00:00:00`，则必须将mysql的sql_mode中的`NO_ZERO_IN_DATE、NO_ZERO_DATE`值取消

