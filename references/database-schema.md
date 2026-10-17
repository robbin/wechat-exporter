# 数据库结构参考

以下来自本项目旧版技能的实测记录，不是跨版本契约。先检查目标库的 `sqlite_master` 与 `PRAGMA table_info`，确认字段、类型与语义，再运行查询。数据库解密不等于消息正文解析完成；正文编码、压缩、附件关联需要另行实现和验证。

## 消息分片

`message/message_*.db` 中可能包含多个会话表。历史样本大体按时间段分片，但编号与时间不是严格线性关系，新会话可能出现在更高编号的分片。发现时须覆盖全部分片，包括多位数编号，不能只查询 `message_0.db`。

历史样本中，表名为 `Msg_<hash>`，其中 `<hash>` 是联系人 `username` 的 MD5；`Name2Id.user_name` 可用于建立映射：

```python
import hashlib
import sqlite3

conn = sqlite3.connect("file:message_0.db?mode=ro", uri=True)
for (username,) in conn.execute("SELECT user_name FROM Name2Id"):
    table = "Msg_" + hashlib.md5(username.encode()).hexdigest()
    # 检查 table 是否实际存在，再查询；不要把聊天内容写入日志。
conn.close()
```

## 联系人

历史样本中的 `contact/contact.db` 包含 `contact` 表：

| local_type | 历史观察 |
| --- | --- |
| 1 | 真实好友（双向添加） |
| 3 | 有过消息往来但未必是好友，包含群聊中的非好友 |
| 0 | 服务号、系统通知 |

确认当前版本语义后，可参考以下个人好友统计：

```sql
SELECT count(*) FROM contact
WHERE delete_flag = 0 AND local_type = 1
  AND username NOT LIKE 'gh_%'
  AND username NOT LIKE '%@chatroom';
```

## 会话时间

历史样本可从 `session/session.db` 查询：

```sql
SELECT datetime(max(last_timestamp), 'unixepoch', '+8 hours')
FROM SessionTable;
```

此例假设时间戳单位为秒，并展示 UTC+8 时间。必须先验证单位与所需时区。这只是会话表中的最大时间，不是全库完整性、WAL 恢复成功或所有消息均已解析的证明。
