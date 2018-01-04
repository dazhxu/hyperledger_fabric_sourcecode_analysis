util
===

util文件定义了一些辅助方法

# createLedgerFactory方法

创建账本工厂方法

参数：

- conf *config.TopLevel

返回值：

- ledger.Factory
- string : 账本位置

对于conf.General.LedgerType，如果是"file"，获取ld，即账本的位置，调用fileledger.New(ld)创建账本lf，创建本地。

