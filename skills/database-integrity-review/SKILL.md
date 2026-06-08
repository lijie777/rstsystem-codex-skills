---
name: database-integrity-review
description: Use when reviewing database, SQL, Qt QSqlQuery/QSqlDatabase, transactions, prepared statements, login/password handling, schema constraints, or data integrity code.
---

# 数据库 / 数据完整性审查 (database-integrity-review)

数据库层的 bug 不像崩溃那样显眼——一条被注入的登录查询、一次没回滚的半截写入、一个跨线程共享的连接，平时都"能用"，直到某天数据被悄悄篡改、患者案例写了一半、或并发下数据错乱。在医疗系统里，**案例/患者/植入物数据的完整性和登录鉴权的安全性是底线**。审查时戴上"如果这条 SQL 拿到恶意输入 / 这次写入中途失败 / 这个连接被两个线程同时用"的眼镜逐项核对。

## 本项目的关键结构（先记住）

项目里**并存两套数据库子系统，写法截然相反**——这本身就是最好的"反模式 vs 正例"对照：

| 子系统 | 位置 | 特征 |
|---|---|---|
| **MySQL 层** | `src/Common/DataBase/`（`DBMgt`/`DataBaseSingleton` 单例 + `MysqlInterfaceImpl` + `DB*Impl` DAO） | ⚠️ **几乎全是字符串拼接 SQL**，反模式集中地 |
| **SQLite 层** | `src/Common/SQLiteAliasManager/`（`SQLiteUserManager`/`SQLiteCaseManager`，登录鉴权+案例） | ✅ **几乎全用 `prepare()`+`bindValue()`**，正例集中地 |

审查 MySQL 层时基本默认有问题；SQLite 层可作为"应该怎么写"的模板。但**SQLite 层在鉴权上也有重大 bug**（见第 7 节），别因为它参数化做得好就放过密码问题。

## 审查前先确认三件事
1. **这条 SQL 的输入来自哪里**？只要有任何外部/用户字段(患者名、登录名、型号)进入 SQL，就必须参数化。
2. **这次写入是单条还是多条/多表**？多条/多表必须在一个事务里，失败整体回滚。
3. **这个 `QSqlDatabase` 连接会被几个线程用**？Qt 要求连接每线程独立。

## Checklist

### 1. SQL 注入 — 最高优先级
- [ ] **绝不用字符串拼接/`QString::arg`/`+` 把外部输入拼进 SQL**。⚠️ 反模式遍布 MySQL 层：`DBUserImpl` 登录查询 `QString("SELECT * FROM %2 WHERE LoginName='%1'").arg(loginName)`(**登录注入最危险**)、`DBCaseImpl` 患者字段 `values += QString("'%1',").arg(...)`、`DBProsthesisRelationImpl` 整行 `INSERT INTO %2 VALUES(%1)`。一个单引号就能破坏语句结构。
- [ ] **正解：`prepare()` + 占位符 + `bindValue()`/`addBindValue()`**。正例 `SQLiteUserManager`：`q.prepare(... " WHERE LoginName=:name"); q.bindValue(":name", loginName);`、`SQLiteCaseManager::addCase` 全字段绑定。审查时把 MySQL 层每条拼接 SQL 标红，对照 SQLite 层改写。

### 2. 事务边界
- [ ] **批量/多表写入必须包在 `transaction()`/`commit()`/`rollback()` 里，失败回滚**。正例 `DBProsthesisRelationImpl::addProsthesisRelationInfo`：`m_db->transaction(); for(...){ if(err){ rollback(); return; } } commit();`。⚠️ 反模式：`DBCaseImpl::deleteCaseInfo(QStringList)`、`DBUserImpl` 批量删用户——**无事务**，中途失败前面已写/删的无法回滚。
- [ ] **嵌套/被打断的事务**。⚠️ 陷阱 `MysqlInterfaceImpl::execsql`：每次 update/delete **内部自带 `commit()`**，会打断外层 `transaction()`——在该框架下嵌套事务不成立，外层 `rollback()` 可能失效。审查：外层开了事务，里面调的函数是否偷偷 commit。

### 3. 错误处理
- [ ] **`exec()` 返回值必须检查，失败要记 `lastError()` 并让调用方能区分"失败"与"无数据"**。⚠️ 反模式 `MysqlInterfaceImpl::selectRows`：`if(!query.exec(sqlStr)) return resultListLst;`——失败只返回空 list、连 `lastError` 都不打印，上层把"查询失败"误当"无数据"，对患者/案例数据是危险静默失败。
- [ ] **正解**：判返回 + `lastError().text()` 记日志 + 用 `numRowsAffected()` 判定写入结果。正例 `SQLiteUserManager`：`if(!q.exec()){ m_lastError=q.lastError().text(); DB_LOG_ERROR(...); return false; } return q.numRowsAffected()>0;`。失败时的保守策略也算正例（`SQLiteCaseManager` 查询异常时返回"存在关联"以拒绝删除）。
- [ ] **裸 `QSqlQuery query;` 用默认连接**而非传入目标库(`MysqlInterfaceImpl::insertOneRow`)——会连错库或失败。审查 query 是否绑定了正确的 `QSqlDatabase`。

### 4. 并发访问
- [ ] **`QSqlDatabase` 连接每线程必须独立；单例共享一个连接是错的**。⚠️ 反模式：`DBMgt`/`DataBaseSingleton` 是进程单例(被约 19 个文件引用，含设备/登录/GUI 等可能不同线程的模块)，底层 `MysqlInterfaceImpl` 用 **`addDatabase("QMYSQL")` 不带连接名**(唯一默认连接)，全 `DataBase/` 目录**无任何 `QMutex` 保护**。多线程并发 = 数据竞争 + 连接失效。
- [ ] **正解方向**：每线程 `addDatabase(driver, 唯一连接名)`，或对单例访问加锁串行化。SQLite 层用**具名连接** `kConnectionName` + `QSqlDatabase::database(name)` 复用(至少可定位)，但**同样没做每线程独立**，仍需修。深入并发分析转 [[cpp-concurrency-review]]。

### 5. 数据完整性
- [ ] **主键不要用 `SELECT MAX(id)+1` 生成**。⚠️ 反模式 `DBUserImpl`/`SQLiteUserManager`/`SQLiteCaseManager` 都这么干——**非原子，并发下撞主键**。正解：自增列/UUID/序列。
- [ ] **建表要有约束**：`PRIMARY KEY` + `NOT NULL` + `UNIQUE` + `FOREIGN KEY`。正例 SQLite 建表(`SQLiteAliasManager`)有 `NOT NULL/UNIQUE/PRIMARY KEY`；⚠️ 但建表配置(`dependence/cfg/.../database.xml`)**未见 `FOREIGN KEY`**——case 与 prosthesis 跨表关联无外键级联，完整性全靠应用层自觉。
- [ ] **写入前关键字段校验**。⚠️ `DBCaseImpl::addCaseInfo` 仅"跳过空字段"，对患者 ID 等关键字段无非空/格式校验，可能写出残缺记录。审查：关键字段为空时是拒绝写入还是静默写残。

### 6. 资源管理
- [ ] **连接/查询不要泄漏**。⚠️ 反模式 `DataBaseSingleton::~DataBaseSingleton`：`if (m_pDBCase == nullptr) { delete m_pDBCase; }` ——**判空写反**，只在指针为空时 delete，非空对象永不释放(泄漏)。这是真实 bug，审查析构时核对每个 `delete` 的判空方向。
- [ ] 正例：`SQLiteAliasManager::closeDatabase` `close()` + 置空 + `removeDatabase(kConnectionName)`，析构调用它。

### 7. 鉴权 / 敏感数据 — 最高危
- [ ] **密码绝不能明文存储或明文比对**。⚠️ 反模式：`SQLiteUserManager::verifyLogin` `if(outUser.loginPass != loginPass)`、`LoginImpl.cpp` `record.loginPass != passWord`——**两条登录路径都是明文比对**；`SQLiteAliasManager` 把 `admin@123456`/`support@123456` **明文 INSERT** 进库。正解：存"加盐 hash"，登录时 hash 后比对。
- [ ] **存储与比对算法必须一致**。⚠️ 真实 bug：`createInitialAdmin` 把密码存为 **SHA-256 hex**，而 `verifyLogin` 用**明文**比对 → 通过该函数创建的管理员**永远登录不上**；同时默认账号又是明文 → 全系统密码格式不统一。审查：写入侧和校验侧的哈希算法/编码是否同一套。
- [ ] **弱哈希**：`UpgradeControlBoardWgt.cpp` 用 MD5、`SpineCaseManager.cpp` 用 SHA-1。鉴权/完整性场景应避免 MD5/SHA-1。
- [ ] **患者 PHI 保护**：`caseinfotb` 的 `PatientName`/`PatientID`/`CtPatientName` 及 CT/X 光路径**明文落库，无加密/脱敏**。审查是否符合医疗数据合规要求（至少敏感字段加密、访问审计）。

## 本项目易错点速查（识别这些"形状"）

| 形状 | 在哪 | 风险 |
|---|---|---|
| `QString("...'%1'...").arg(userInput)` 拼 SQL | `DBUserImpl`(登录)、`DBCaseImpl`、`DBProsthesisRelationImpl` | SQL 注入，登录可被绕过 |
| 明文密码比对 `loginPass != loginPass` | `SQLiteUserManager::verifyLogin`、`LoginImpl.cpp` | 库泄露即密码全泄 |
| 存 SHA-256 却比明文 | `createInitialAdmin` vs `verifyLogin` | 加 hash 的账号反而登不上 + 格式不统一 |
| 明文/硬编码默认账号入库 | `SQLiteAliasManager` 默认 admin | 弱口令明文 |
| `if(!exec()) return 空集合`，不记 lastError | `MysqlInterfaceImpl::selectRows` | "失败"被当"无数据" |
| 批量/多表写入无事务 | `DBCaseImpl::deleteCaseInfo`、`DBUserImpl` 批量删 | 半截写入无法回滚 |
| `execsql` 内部强制 commit 打断外层事务 | `MysqlInterfaceImpl::execsql` | 嵌套事务失效，rollback 无效 |
| `addDatabase("QMYSQL")` 无名默认连接被单例跨线程共享、无锁 | `MysqlInterfaceImpl` + `DBMgt` | 跨线程数据竞争/连接失效 |
| `SELECT MAX(id)+1` 生成主键 | `DBUserImpl`、`SQLiteCaseManager` | 并发撞主键 |
| 析构 `if(p==nullptr){ delete p; }` 判空写反 | `DataBaseSingleton::~DataBaseSingleton` | DAO 对象泄漏 |
| 建表无 `FOREIGN KEY`、关键字段无 NOT NULL 校验 | `database.xml`、`DBCaseImpl::addCaseInfo` | 跨表完整性靠自觉、写残记录 |
| 患者 PHI 明文落库 | `caseinfotb` 各字段 | 隐私/合规风险 |

> 安全范例（"应该怎么写"的对照）：SQLite 层 `SQLiteUserManager`/`SQLiteCaseManager` 的 `prepare()+bindValue()`、`exec()` 判返回 + `lastError()` + `numRowsAffected()`、具名连接、建表 `NOT NULL/UNIQUE/PRIMARY KEY`、`DBProsthesisRelationImpl` 的 `transaction/rollback/commit` 包裹。

## 输出格式（审查报告）

```
# 数据库/数据完整性审查报告

## 数据流梳理（我的理解）
- 涉及的表/DAO，输入字段来源(是否含外部输入)
- 写操作是单条/多条/多表，事务覆盖情况
- 连接对象与其被访问的线程

## 🔴 阻断级（SQL 注入 / 明文密码 / 存hash比明文 / 跨线程共享连接 / 半截写入不回滚）
1. [文件:函数] 问题：...
   被攻击/被破坏时会发生什么：...
   修法：...

## 🟠 严重（exec 失败静默返回空 / MAX+1 主键 / 析构泄漏 / 缺约束）
...

## 🟡 建议（PHI 加密 / 统一哈希 / 弱哈希升级 / 写前字段校验）
...

## 验证建议
- 对每条 SQL 问"输入可控吗"，可控就必须参数化
- 对每个写操作问"中途失败会留下半截数据吗"
- 对每个连接问"会被别的线程用吗"
- 登录路径：存储侧与校验侧哈希算法是否同一套、是否明文
```

## 重要提醒
- **注入和明文密码是底线问题**——MySQL 层的拼接 SQL 和登录明文比对应优先于一切其它清理项修复。
- 数据库单例常被设备/GUI/登录多线程访问，连接线程归属与锁问题配合 [[cpp-concurrency-review]] 一起审。
- 患者/案例数据写入失败若被静默吞掉，会与手术流程的"用空数据继续"叠加成临床风险，与 [[surgical-safety-review]] 相关。
- 不确定某 SQL 驱动(QMYSQL/QSQLITE)的线程/事务语义时，明确提示需查 Qt SQL 模块文档的 "Threads and the SQL Module" 段落，不要臆断。
