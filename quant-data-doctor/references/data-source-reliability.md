# 数据源可靠性对照表

## 常见数据源实测特征

| 数据源 | 覆盖 | 速度 | 稳定性 | 单位 | 主要坑 |
|---|---|---|---|---|---|
| akshare | 全 A 股 | 慢 | 差 | 手 | 接口随时会变；批量接口静默返回空 |
| pytdx | 全 A 股 | 快 | 中 | 手 | 需装客户端；批量接口返回空不报错 |
| 腾讯 HTTP | 常见股 | 中 | 中 | 股 | 覆盖不全 |
| 东财 HTTP | 常见股 | 中 | 中 | 股 | 覆盖不全 |
| mootdx | 全 A 股 | 快 | 中 | 手 | 需要本地数据服务 |
| 通达信本地 | 全 A 股 | 最快 | 高 | 手 | 需要客户端 + 下载数据 |

**结论**：**永远不要只用一个源。** 主源 + 备源 + 兜底，至少三层。

## 单位统一规则

```
存储层：统一用「手」
展示层：需要「股」时 × 100
接口层：读取时按来源转换，不要信任上游的原始单位
```

混用单位的后果：成交额、资金流向、成交量类指标全部系统性偏移，且**不会报错**。

## 常见根因排查顺序

按实测频率排序：

1. **批量接口静默返回空** —— 最常见。不报错，脚本正常结束，数据为0
2. **逐只串行太慢** —— 当日任务没跑完
3. **代理/DNS 拦截** —— 特定域名被拦
4. **接口字段变更** —— 上游改了返回结构
5. **并发过高被限流** —— 需要降并发

## 静默失败检测代码

```python
def verify_update(db, today=None):
    """更新后立即校验，失败立即告警，不要等回测。"""
    cur_max = db.execute("SELECT MAX(date) FROM daily_bars").fetchone()[0]
    meta = db.execute(
        "SELECT date, rows, status FROM update_meta ORDER BY rowid DESC LIMIT 1"
    ).fetchone()
    if meta and cur_max and meta[0] != cur_max:
        raise RuntimeError(f"静默失败：同步记录 {meta[0]} 但数据最新 {cur_max}")
    if meta and meta[2] == "failed":
        raise RuntimeError(f"更新失败：{meta[0]} {meta[1]}")
    return True
```

## 完整性基线

```sql
-- A 股日线：正常每个交易日约 3800 行
SELECT date, COUNT(*) FROM daily_bars
WHERE date >= DATE('now','-7 day')
GROUP BY date ORDER BY date;
```

低于 3000 → 疑似缺失，低于 1000 → 严重缺失，需要立即补数。