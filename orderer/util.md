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

对于conf.General.LedgerType，如果是"file"，获取ld，即账本的位置，调用fileledger.New(ld)创建账本lf，调用createSubDir(ld, fsblkstorage.ChainsDir)创建本地子目录存放每个chain的账本。

如果conf.General.LedgerType是json，从conf.FileLedgerLocation上获取ld，即账本位置，如果ld为空，调用createTempDir(conf.FileLedger.Prefix)创建临时目录，然后调用jsonledger.New(ld)创建账本lf

如果conf.General.LedgerType是ram或其他，调用ramledger.New(int(conf.RAMLedger.HistorySize))创建账本lf

返回lf和ld

# createTempDir方法

调用ioutil.TempDir创建目录

# createSubDir方法

创建子目录

调用os.Mkdir(subDirPath,0755)创建目录