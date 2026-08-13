---
name: incident
description: UniUni 渥太华仓每日 incident（客服/商家投诉）处理流水线——判断 POD 合格性，给司机发 WhatsApp、填 Google Sheets 追踪表、在官网 incident_form 记 lost。触发词：处理今天的 incident / 处理投诉 / incident 流程。
---

# UniUni Incident 每日处理流水线

## 前提条件（开始前向用户确认）
1. Chrome 已登录 UniUni 内部系统（uniexpress 后台）和 Google 账号（能打开追踪表）
2. WhatsApp Windows 桌面客户端已打开并登录
3. 用 **Claude in Chrome**（mcp__claude-in-chrome__*）操作用户真实浏览器；WhatsApp 桌面端用 computer-use
4. 登录、验证码永远由用户自己完成

## 全流程总览

```
第0步 5天到期扫描（Google Sheets） → 过期未修复的单加入"记lost队列"
第1步 官网刷 incident 列表（筛当天）
第2步 逐单调查（Edit Order → POD 照片 + 地址核对 + Canada Post）
第3步 分类判定（决策树）→ 生成判定表给用户过目
第4步 执行三件事：WhatsApp 发司机 / Google Sheets 记行 / 官网 incident_form 处理
```

## 第1步：官网入口与筛选

- 入口：左侧地图页 → 选仓库（7. Ottawa Warehouse）→ 橙色磁贴 **INCIDENT MANAGEMENT**
- 顶部单选：Incident Management（默认）｜Key Account｜Compensated Order
- 筛选条件：From Date / To Date 设为当天（或用户指定范围），其余默认 All → SEARCH
- 逐条记录列表里的 tracking number、Incident Type、Recorded By、Notes

## 第2步：逐单调查（Edit Order）

1. 打开 Edit Order，搜索框输入单号，Search Type 选 **Ant Parcel订单号**
2. 读 **Order Information → Important Info(XXX)**：
   - `SHEIN DISTRIBUTION CANADA LIMITED` / Partner: SHEIN → **Shein 单**
   - 其他（如 Jiayou - Mail）→ 非 Shein
3. 读关键字段：Destination Info 的 **Address / Unit / Buzz Code**、Other Info 的 **Driver ID（号码 - 姓名）/ Driver Phone**、Batch Info、Tracking Info 时间线（199/200/202/203、FAILED_DELIVERY 距离）
4. 滚到页面底部 **Signature / Photo Review / Photo** 区看 POD 照片（POD1/POD2/POD3，可点开放大）
5. POD 合格性判断：
   - 照片里门牌/墙上/标签上的 unit 号是否与系统 Unit 一致
   - 是否放进邮箱/邮箱房（多户楼放邮箱 = Shein 直接 lost；非 Shein 让司机补证明）
   - 是否"随意角落"（random corner，无明确门址特征）
   - 需签字的单：照片是否拍到**人手交接**，只有签名没有手 = 不合格
   - POD 没拍到 unit 号也没拍到放置位置 = No dropping location
6. **Canada Post 核查**（当怀疑地址是多 unit 楼时）：新标签打开 Canada Post 网站查该地址——若地址下有多个 unit 而包裹放在公共门口/门厅 = 不合格
7. 特殊类型：
   - **破损/漏液**：司机不赔，但需发消息让司机把破损件带回仓库检查
   - **GPS 漂移**（Delivered Coordinates 离地址远但 POD 照片正确）：司机不赔，让司机修复/更新 tracking
   - **断更**（司机长时间未更新 in-transit 包裹）：发限期消息，逾期记 lost

## 第3步：判定决策树

| 场景 | Shein? | 结论 | WhatsApp | Sheets 处理结果 | 官网动作 |
|---|---|---|---|---|---|
| POD 放邮箱/邮箱房 | 是 | 直接记 lost | 不发 | 15-Invalid POD | Proceed to Penalty |
| POD 随意角落/无 unit/错 unit | 是 | 直接记 lost | 不发 | 15-Invalid POD 或 12-Wrong address | Proceed to Penalty |
| POD 不合格（各类） | 否 | 发司机限 5 天修复 | 发对应模板 | 先记行、处理结果留空 | 暂不动；5 天后未修复→Proceed to Penalty |
| 送错地址（unit 不符） | 否 | 发 Wrong Address 模板 | 发 | 12-Delivered to the wrong address（记lost时） | 同上 |
| GPS 漂移、POD 本身正确 | - | 司机修复即可，不赔 | 发提醒 | Fixed/no penalty + 备注"Wrong GPS,司机不赔，已让司机去修复" | Update Tracking Status / Not Applicable |
| 破损/漏液 | - | 不赔，返仓 | 发 Damaged 模板 | 按现行习惯 | 不记赔 |
| 已有 message proof / POD 已补 | - | 已解决 | - | 对应结果 | Valid POD Available Upon Request 等 |
| 该单已记过一次丢 | - | 不重复记 | - | 备注"这单已经记丢过一次了" | 本 incident 不记丢 |

**置信度规则**：POD 照片看不清 unit 号、地址歧义、无法判断是否多 unit 楼 → 标记为"待人工"，单独列出让用户拍板。**凡涉及司机赔钱（Proceed to Penalty）的判定，最终由用户确认。**

## 第4步：三个输出

> **可以用仓库根目录的 `index.html`（Incident 处理台）**：把 Edit Order 页面 Ctrl+A 的文字粘进去，
> 自动解析字段并生成 4a 的 WhatsApp 消息、4b 的表格行（10 列 TSV）、4c 的 Notes。
> Shein 单会自动切成不含客户信息的简洁消息；中介列由内置名单按 Driver ID 自动填。

### 4a. WhatsApp 消息（桌面客户端，computer-use）
联系人：按司机名/司机群（用户指定）。消息格式（见 whatsapp消息模版1.png）：

```
<单号>
 DELIVERED
<事件日期 YYYY-MM-DD>
<路线号> -<站点号>
Consignee: <收件人>
Phone: <电话>
Extension Num: <N/A或分机>
email: <N/A或邮箱>

Destination Info
Address: <完整地址>
Unit: <unit号>
Postal Code: <邮编>

<场景模板句（见下方模板库）>
```
并附：订单详情页截图 + POD 照片。**每条消息填好后必须等用户确认再发送。**

### 4b. Google Sheets 追踪表
沿用现有列（见 google sheets 模版.png）：
记录日期 | 事件日期 | Handled by（Lin，下拉）| 司机名 | 司机ID | 路线号 | 站点号 | 单号 | 地址 | 处理结果（下拉）| 原因备注

- 处理结果下拉常用值：`15-Invalid POD (Proof of Delivery)`、`12-Delivered to the wrong address`、`Fixed/no penalty`
- 非 Shein 待修复的单：处理结果**留空**，5 天后扫描时若仍空且未修复 → 补记 lost
- 原因备注常用值：`Invalid POD`、`Wrong address`、`Wrong GPS,司机不赔，已让司机去修复`

### 4c. 官网 incident_form（记 lost / 关单）
路径：Edit Order 右侧 **Incident Management 面板 → MORE INFO** → 弹出 incident_form。

1. **Basic Information**（多伦多已填好，只读）：Incident Type、Recorded By、Notes 等
2. 点 **START INVESTIGATION** 展开 In Progress 表单：
   - Handled By: `Lin_Cao`
   - Caused By*: `Driver`（按实际）
   - Root Cause*: `Parcel Missing/ Lost`（按实际）
   - Corrective Action Record 下拉选项：Update Tracking Status / Fix the Address & Redeliver / Valid POD Available Upon Request / **Proceed to Penalty** / Quick Review Training / Transshipped through Canada Post / UPS·Truck·Flight Transhipment / Not Applicable
   - 选 Proceed to Penalty 时追加：Driver Penalty*: `YES`、Check Complete*: `YES`
   - Notes：中文说明 + `//` 备注短语（如 `送错LPC 确认赔偿//已和司机沟通`）
   - 点 **SAVE INVESTIGATION**
3. **Review** 段（保存后出现）：
   - Reviewed & Confirmed*: `Confirm Penalty - Charge to Caused By Party & Confirm Penalty to Client`（记赔时）
   - Consignee Refund Status / Client Refund Status: 按实际（常用 `No Deduction`）
   - Close Date: 当天
   - 提交关单

**SAVE INVESTIGATION 和 Review 提交属不可逆写入，点击前必须逐单向用户确认。**

## 模板库（来自 incident原因.txt）

### 官网 Notes 用备注短语（// 开头，可组合）
`//已和司机沟通` `//lost` `//delivered, message proof uploaded` `//delivered. POD can not be updated` `//delivered. POD updated` `//已归仓` `//已提醒司机` `//fixed, can not update POD` `//这单已经记丢过一次了，这个incident里面不记丢` `//已提醒司机需要拍到交接的照片` `//如果之后有投诉没送到就记丢`

### 场景消息模板
- **Shein 邮箱**: This parcel is lost, it is a shein package. This parcel is delivered to mailbox
- **Shein 随意角落**: This parcel is lost, it is a shein package. This parcel is delivered to a random corner
- **邮箱房（修复）**: This parcel is delivered to mailbox room. Cx is complaining not able to find it. Please try to fix it. Thanks
- **邮箱房（要证明）**: This parcel is delivered to mailbox room. Please try to contact the customer and get a message proof with parcel ID in it showing the parcel was delivered. Thanks
- **随意角落（要证明）**: This parcel is delivered to a random corner. Please try to contact the customer and get a message proof with parcel ID in it showing the parcel was delivered. Thanks
- **无放置点**: The POD does not include unit number nor dropping location, please get a message proof from cx, thanks
- **错地址**: The correct address is (unit xx), The POD is showing (unit xx). This parcel is delivered to the wrong address. Cx is complaining not able to find it. Please try to fix it.
- **签名无手（要证明）**: POD is invalid since the POD does not include person's hand. Since there is a signature, I would assume the parcel was delivered. Can you please try to contact the customer and get a message proof with parcel ID in it showing the parcel was delivered? Thanks
- **签名无手（提醒司机）**: POD is invalid since the POD does not include person's hand. Please make sure that you take a picture of the person's hand when you need a signature from cx next time. Thanks
- **签名无手（提醒 DSP）**: ...that your drivers take a picture of the person's hand when they need a signature from cx next time. Thanks
- **断更（司机）**: If you still have these parcels, please try to deliver them or return them to the office by the noon of <日期>, or it will be marked lost, thanks
- **断更（DSP）**: If your drivers still have these parcels, ...（同上，drivers 版本）
- **催更新**: Please update this parcel, or it will be marked as lost soon, thanks.
- **司机停留过久**: Hello X, your driver xxx still have x parcels in transit, and he has stopped for x minutes, if he has already finished them please make sure he will upload all the PODs, thanks.
- **破损/漏液**: This parcel is reported leaked, please note if your driver find any parcel that is leaked or damaged, please ask the driver to return it to office and let office check before delivering in the morning. For this parcel, we will not charge your driver, but please notify your drivers about damaged and leaked parcels, thanks
- **给客户的短信（地址缺 unit）**: Dear xxx, This is UniUni Ottawa warehouse. We attempted to deliver your parcel (tracking: [xxx]); however, the address is incorrect or lacks a unit number (xxx) in our system. Please confirm the street#, unit#, and buzzer code if required by replying to this text message. Thank you!

## 执行模式与边界（不可省略）

1. **批量呈现**：第3步结束后先输出一张判定总表（单号 / Shein? / 场景 / 结论 / 置信度 / 三个输出的草稿），用户过目后再进入第4步
2. **必须逐项确认**：WhatsApp 发送、官网 SAVE INVESTIGATION / Review 提交。Sheets 写入可在用户批准总表后直接执行
3. **低置信度强制转人工**：照片模糊、unit 歧义、拿不准是否多 unit 楼
4. 运行结束输出小结：处理了几单、几单记 lost、几单待 5 天跟进、几单转人工

## 校准期（前两周）
影子运行：只产出判定表和草稿，不执行；用户对照手工判断打分，一致率 >90% 后转正式模式。发现判定规则和实际不符时，更新本文件的决策树。
