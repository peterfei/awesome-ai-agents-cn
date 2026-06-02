---
title: 用 n8n + ERP API 搭建订单异常自动处理 Agent
author: "@potato"
date: 2026-06
stack: n8n, PostgreSQL, 企业微信, ERP API, Python
difficulty: 进阶
---

## 场景

电商运营每天处理数千订单，其中 5-8% 会出现异常情况：

- 地址不规范（省市区缺失、手机号位数不对）
- 库存不足（下单时有货，付款后缺货）
- 物流不可达（偏远地区、疫情管控区域）
- 价格异常（优惠券叠加导致负价、大额订单需审核）
- 重复下单

人工处理流程：运营每天打开 ERP → 导出异常单 → 逐个判断 → 联系客户/换仓/退款。每天花 2-3 小时在机械性操作上。

目标是：**Agent 自动识别异常、分类处理、能自动解决的直接处理，需人工的推送待办**。

---

## 实现

### 架构

```
订单状态变更 → n8n 定时轮询 → 规则引擎初筛 → LLM 复杂判断 → ① 地址问题 → 调用地图API补全/发短信确认
                                          → ② 库存不足 → 自动拆单/换仓/推采购单
                                          → ③ 物流不可达 → 标记+通知客户改地址
                                          → ④ 价格异常 → 推风控审核
                                          → ⑤ 重复下单 → 自动合并/取消
                                          → ⑥ 无法判定 → 人工待办
```

### 核心实现

**第一步：异常检测规则引擎**

```python
# n8n Function 节点
def detect_anomaly(order):
    issues = []
    
    # 地址校验
    if not validate_address(order["address"]):
        issues.append({"type": "address_invalid", "severity": "high"})
    
    # 手机号校验
    if not re.match(r"^1[3-9]\d{9}$", order["phone"]):
        issues.append({"type": "phone_invalid", "severity": "high"})
    
    # 库存检查
    for item in order["items"]:
        if item["qty"] > get_inventory(item["sku"]):
            issues.append({"type": "out_of_stock", "sku": item["sku"], "severity": "critical"})
    
    # 价格异常
    if order["total_amount"] < 0 or order["total_amount"] > 50000:
        issues.append({"type": "price_anomaly", "severity": "critical"})
    
    # 重复下单（24h内同手机号同地址同商品）
    if find_duplicate_orders(order):
        issues.append({"type": "duplicate", "severity": "medium"})
    
    # 物流可达性
    if not check_delivery_reachable(order["address"]):
        issues.append({"type": "unreachable", "severity": "high"})
    
    return issues
```

**第二步：自动处理策略**

```python
def auto_resolve(order, issues):
    actions = []
    
    for issue in issues:
        if issue["type"] == "address_invalid":
            # 尝试用地图 API 补全
            fixed = fix_address_with_map_api(order["address"])
            if fixed:
                actions.append({"action": "fix_address", "value": fixed})
            else:
                actions.append({"action": "sms_confirm", "template": "address_confirm"})
                
        elif issue["type"] == "out_of_stock":
            sku = issue["sku"]
            # 检查其他仓库是否有货
            alt_warehouse = find_stock_in_other_warehouse(sku)
            if alt_warehouse:
                actions.append({"action": "switch_warehouse", "warehouse": alt_warehouse})
            else:
                # 检查是否可以部分发货
                available_qty = get_inventory(sku)
                if available_qty > 0:
                    actions.append({"action": "partial_ship", "qty": available_qty})
                actions.append({"action": "push_purchase_order", "sku": sku})
                actions.append({"action": "notify_customer", "template": "delay_notice"})
                
        elif issue["type"] == "duplicate":
            dup = find_duplicate_orders(order)[0]
            if dup["status"] == "paid":
                actions.append({"action": "cancel_duplicate", "order_id": order["id"]})
            else:
                actions.append({"action": "merge_orders", "order_ids": [order["id"], dup["id"]]})
                
        elif issue["type"] == "unreachable":
            actions.append({"action": "hold_order", "reason": "物流不可达"})
            actions.append({"action": "notify_customer", "template": "change_address"})
            
        elif issue["type"] == "price_anomaly":
            actions.append({"action": "flag_review", "team": "风控"})
    
    return actions
```

**第三步：需人工处理的情况**

```python
def needs_human_review(order, issues, actions):
    """以下情况必须人工介入"""
    
    # 1. 自动处理失败（地图API也补不全地址）
    if any(i["type"] == "address_invalid" for i in issues) and \
       not any(a["action"] == "fix_address" for a in actions):
        return True, "地址无法自动修复，需客户确认"
    
    # 2. 大额订单（>1万）即使正常也人工复核
    if order["total_amount"] > 10000:
        return True, "大额订单自动复核"
    
    # 3. 客户标记（VIP/黑名单）
    if order["customer_tag"] in ["VIP", "blacklist"]:
        return True, f"{order['customer_tag']} 客户订单需人工处理"
    
    # 4. 多异常叠加（>=3 种异常）
    if len(set(i["type"] for i in issues)) >= 3:
        return True, "多异常叠加，建议人工判断"
    
    return False, None
```

**第四步：企微通知模板**

```
⚠️ 订单异常需处理

订单号：{{order_id}}
客户：{{customer_name}} {{customer_tag}}
金额：¥{{amount}}
异常：{{issue_types}}

已自动执行：
{{auto_actions}}

需人工决策：
{{human_reason}}

[处理] [查看详情] [转交同事]
```

---

## 效果

在日单量 3000+ 的电商场景中，预期效果：

| 指标 | 变化 |
|------|------|
| 异常订单自动处理率 | 68% |
| 运营每日处理异常时间 | 3h → 45min |
| 异常订单响应时间 | 平均 4h → 平均 10min（自动） |
| 库存换仓成功率 | 85%（自动找到替代仓库） |
| 重复订单取消准确率 | 97% |
| 人工介入后处理时长 | 15min/单 → 3min/单（前置信息已整理） |

---

## 要点

1. **规则引擎先过滤，LLM 处理复杂判断** — 80% 的异常可以用简单规则处理（手机号格式、库存数字比对），不需要调用 LLM。LLM 用在地址语义理解、客户备注意图分析等复杂场景，节省成本。

2. **自动处理必须有兜底** — 自动换仓、自动拆单这类操作如果出错代价大。建议设置"自动处理但不自动提交"模式：Agent 给出处理建议，运营一键确认后执行。

3. **客户通知话术要统一** — 延迟发货、地址确认这类短信/站内信，话术要经法务/客服审核后固化成模板，不要让 Agent 自由发挥，避免纠纷。

4. **日志留痕是刚需** — 自动处理的所有操作必须留痕（谁/何时/做了什么/为什么），客户投诉时可以回溯。这也是财务审计的要求。

---

## 扩展思路

- 接入物流 API 实时查询，异常单自动推荐可达的替代物流方案
- 智能采购建议：根据缺货 SKU 的销量趋势，自动生成补货建议并推送给采购
- 客户画像联动：高价值客户的异常单自动升级处理优先级
- 预售模式：缺货 SKU 自动转预售，计算预计到货时间并告知客户

---

## 相关资源

- [n8n 工作流编排](https://docs.n8n.io)
- [高德地图地址解析 API](https://lbs.amap.com/api/webservice/guide/api/georegeo)
- [快递鸟物流可达性查询](https://www.kdniao.com)
- [PostgreSQL 触发器实现订单变更监听](https://www.postgresql.org/docs/current/sql-createtrigger.html)
