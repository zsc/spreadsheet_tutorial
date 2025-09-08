# 第7章：脚本化与自动化编程

## 章节概览

电子表格的脚本化能力将其从静态的数据容器转变为动态的应用平台。本章深入探讨Google Apps Script的架构设计、事件驱动编程模型、自定义函数实现原理，以及飞书多维表格的脚本能力和低代码平台。我们将从执行环境、安全模型、性能优化等多个维度剖析表格脚本化的技术细节，并提供实践中的最佳模式。

## 学习目标

- 理解电子表格脚本执行环境的架构设计
- 掌握事件驱动模型与触发器的工作原理
- 学习自定义函数与宏的实现机制
- 了解飞书多维表格的脚本能力与低代码设计理念
- 掌握安全沙箱的隔离机制与权限控制
- 熟悉调试工具链与性能分析方法

## 7.1 Google Apps Script架构剖析

### 7.1.1 执行环境架构

Google Apps Script (GAS) 基于 Rhino JavaScript 引擎（后迁移到 V8），提供了一个云端的 JavaScript 运行时环境。其架构设计有几个关键特点：

**分层架构模型**

```
┌─────────────────────────────────────┐
│         用户脚本代码                │
├─────────────────────────────────────┤
│      Apps Script API 层             │
│   (SpreadsheetApp, DriveApp等)      │
├─────────────────────────────────────┤
│        执行运行时 (V8)              │
├─────────────────────────────────────┤
│      安全沙箱与权限管理             │
├─────────────────────────────────────┤
│     Google Cloud Platform           │
└─────────────────────────────────────┘
```

**执行模型特点**：

1. **无状态执行**：每次脚本执行都在独立的上下文中运行，执行完成后上下文销毁
2. **执行时限**：单次执行最长6分钟（简单触发器30秒）
3. **配额限制**：API调用次数、执行时间总量都有日配额限制
4. **异步支持**：通过 UrlFetchApp 支持异步 HTTP 请求，但整体仍是同步执行模型

### 7.1.2 API设计哲学

GAS 的 API 设计遵循"领域对象模型"（Domain Object Model）原则：

```javascript
// 典型的链式调用模式
SpreadsheetApp
  .getActiveSpreadsheet()
  .getSheetByName("数据")
  .getRange("A1:C10")
  .setValues(data);
```

这种设计的优势：
- **直观性**：API 结构映射实际的表格层次结构
- **类型安全**：每个方法返回特定类型的对象
- **可发现性**：IDE 可以提供良好的自动补全

### 7.1.3 服务集成架构

GAS 最强大的特性是与 Google Workspace 的深度集成：

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Sheets     │────▶│  Apps Script │────▶│    Drive     │
└──────────────┘     └──────────────┘     └──────────────┘
                             │
                             ▼
                     ┌──────────────┐
                     │    Gmail     │
                     └──────────────┘
```

**集成要点**：
- **统一认证**：使用 OAuth 2.0，用户授权一次即可访问多个服务
- **数据流转**：可以在不同服务间无缝传递数据
- **事件联动**：一个服务的事件可以触发对其他服务的操作

## 7.2 事件驱动模型与触发器机制

### 7.2.1 触发器类型与特性

GAS 支持多种触发器类型，每种都有其特定的使用场景和限制：

**1. 简单触发器（Simple Triggers）**

```javascript
// onOpen - 表格打开时触发
function onOpen(e) {
  SpreadsheetApp.getUi()
    .createMenu('自定义菜单')
    .addItem('执行分析', 'runAnalysis')
    .addToUi();
}

// onEdit - 单元格编辑时触发
function onEdit(e) {
  const range = e.range;
  const value = e.value;
  
  // 记录编辑历史
  logEdit(range.getA1Notation(), value, e.user);
}
```

特点：
- 自动触发，无需安装
- 权限受限（不能访问需要授权的服务）
- 执行时限30秒
- 不能执行需要用户交互的操作

**2. 可安装触发器（Installable Triggers）**

```javascript
// 时间驱动触发器
function setupTimeTrigger() {
  ScriptApp.newTrigger('processData')
    .timeBased()
    .everyHours(1)
    .create();
}

// 表单提交触发器
function onFormSubmit(e) {
  const responses = e.values;
  processFormData(responses);
  sendNotification(e.email);
}
```

优势：
- 可以访问需要授权的服务
- 执行时限6分钟
- 支持更多事件类型
- 可以以脚本所有者身份运行

### 7.2.2 事件对象结构

触发器接收的事件对象包含丰富的上下文信息：

```javascript
// onEdit 事件对象示例
{
  authMode: ScriptApp.AuthMode.LIMITED,
  range: Range,           // 被编辑的范围
  source: Spreadsheet,    // 源表格
  user: User,            // 执行用户
  value: Object,         // 新值
  oldValue: Object       // 旧值（仅可安装触发器）
}
```

### 7.2.3 触发器链与级联效应

在复杂应用中，触发器可能形成链式反应：

```
用户编辑 → onEdit触发器 → 更新其他单元格 → 公式重算 → onChange触发器
    ↓
发送通知 ← 记录日志 ← 数据验证
```

**最佳实践**：
1. **防止循环触发**：使用标志位或时间戳避免无限循环
2. **批量处理**：累积多个变更后统一处理
3. **错误隔离**：使用 try-catch 确保单个错误不影响整体流程

## 7.3 自定义函数与宏的实现原理

### 7.3.1 自定义函数机制

自定义函数允许用户扩展表格的公式能力：

```javascript
/**
 * 计算加权平均值
 * @param {range} values 数值范围
 * @param {range} weights 权重范围
 * @return {number} 加权平均值
 * @customfunction
 */
function WEIGHTED_AVG(values, weights) {
  if (values.length !== weights.length) {
    throw new Error('数值和权重数量不匹配');
  }
  
  let sum = 0, weightSum = 0;
  for (let i = 0; i < values.length; i++) {
    sum += values[i] * weights[i];
    weightSum += weights[i];
  }
  
  return sum / weightSum;
}
```

**执行特性**：
- **自动重算**：当依赖的单元格变化时自动重新执行
- **缓存机制**：相同输入会返回缓存结果
- **限制**：不能修改表格，只能返回值
- **并行执行**：多个自定义函数可以并行计算

### 7.3.2 宏的录制与回放

宏通过录制用户操作生成脚本代码：

```javascript
// 录制的宏示例
function Macro1() {
  var spreadsheet = SpreadsheetApp.getActive();
  spreadsheet.getRange('A1').activate();
  spreadsheet.getCurrentCell().setValue('开始');
  spreadsheet.getRange('A2:A10').activate();
  spreadsheet.getActiveRange().setBackground('#ffff00');
}
```

**宏执行流程**：
```
录制开始 → 捕获UI操作 → 转换为API调用 → 生成脚本代码 → 存储
    ↓
执行宏 → 加载脚本 → 顺序执行API调用 → 更新UI
```

### 7.3.3 性能优化策略

批量操作优化：
```javascript
// 低效方式 - 多次API调用
for (let i = 1; i <= 100; i++) {
  sheet.getRange(i, 1).setValue(i);
}

// 高效方式 - 批量操作
const values = [];
for (let i = 1; i <= 100; i++) {
  values.push([i]);
}
sheet.getRange(1, 1, 100, 1).setValues(values);
```

## 7.4 飞书多维表格的脚本能力与低代码平台

### 7.4.1 脚本执行环境

飞书多维表格采用了不同于 GAS 的设计理念，更强调低代码和可视化配置：

```
┌─────────────────────────────────────┐
│        可视化配置界面               │
│    (自动化规则、按钮、表单)         │
├─────────────────────────────────────┤
│         脚本执行引擎                │
│      (JavaScript 沙箱)              │
├─────────────────────────────────────┤
│      飞书开放平台 API               │
├─────────────────────────────────────┤
│       数据存储与同步                │
└─────────────────────────────────────┘
```

### 7.4.2 自动化规则系统

飞书的自动化规则采用"条件-动作"模式：

```
触发条件：
  - 记录创建/更新/删除
  - 字段值满足条件
  - 定时触发
  - 按钮点击
    ↓
执行动作：
  - 更新字段值
  - 发送通知
  - 创建关联记录
  - 调用外部API
```

**规则配置示例**：
```json
{
  "trigger": {
    "type": "record_updated",
    "conditions": [
      {"field": "状态", "operator": "equals", "value": "已完成"}
    ]
  },
  "actions": [
    {
      "type": "update_field",
      "field": "完成时间",
      "value": "{{NOW()}}"
    },
    {
      "type": "send_notification",
      "recipient": "{{创建者}}",
      "message": "任务已完成"
    }
  ]
}
```

### 7.4.3 低代码设计理念

飞书多维表格的低代码平台特点：

1. **可视化流程编排**
   - 拖拽式流程设计
   - 实时预览执行效果
   - 图形化调试界面

2. **预置组件库**
   - 丰富的UI组件
   - 常用业务逻辑模板
   - 第三方服务连接器

3. **渐进式复杂度**
   - 简单场景无需代码
   - 复杂逻辑支持脚本
   - 可导出为独立应用

### 7.4.4 与飞书生态的集成

```
多维表格 ←→ 飞书文档
    ↓         ↓
飞书机器人 ← 审批流程
    ↓         ↓
  消息通知   OKR系统
```

集成能力：
- **数据联动**：表格数据可直接在文档中引用
- **流程自动化**：与审批、OKR等业务流程打通
- **智能助手**：通过机器人实现自然语言交互

## 7.5 安全沙箱与执行限制

### 7.5.1 沙箱隔离机制

脚本执行环境的安全隔离是关键：

```
┌─────────────────────────────────────┐
│          用户脚本代码               │
├─────────────────────────────────────┤
│    受限JavaScript环境               │
│  - 无DOM访问                        │
│  - 无文件系统访问                   │
│  - 网络请求需授权                   │
├─────────────────────────────────────┤
│      资源限制层                     │
│  - CPU时间限制                      │
│  - 内存使用限制                     │
│  - API调用配额                      │
├─────────────────────────────────────┤
│      宿主环境                       │
└─────────────────────────────────────┘
```

### 7.5.2 权限模型

细粒度的权限控制：

```javascript
// 权限声明示例
{
  "oauthScopes": [
    "https://www.googleapis.com/auth/spreadsheets",
    "https://www.googleapis.com/auth/drive.readonly",
    "https://www.googleapis.com/auth/gmail.send"
  ],
  "urlFetchWhitelist": [
    "https://api.example.com/*"
  ]
}
```

**权限级别**：
1. **只读权限**：仅能读取数据
2. **当前文档权限**：可修改当前表格
3. **完全权限**：可访问用户的所有文档

### 7.5.3 执行限制与配额

典型的执行限制：

| 限制项 | Google Apps Script | 飞书多维表格 |
|--------|-------------------|--------------|
| 单次执行时间 | 6分钟 | 60秒 |
| 日执行时间总量 | 6小时 | 2小时 |
| API调用次数/天 | 20,000 | 10,000 |
| 并发执行数 | 30 | 20 |
| 脚本大小 | 1MB | 500KB |

### 7.5.4 安全最佳实践

1. **输入验证**
```javascript
function processUserInput(input) {
  // 始终验证和清理用户输入
  if (typeof input !== 'string' || input.length > 1000) {
    throw new Error('Invalid input');
  }
  // 防止注入攻击
  const sanitized = input.replace(/[<>]/g, '');
  return sanitized;
}
```

2. **敏感信息处理**
```javascript
// 使用脚本属性存储敏感信息
const apiKey = PropertiesService.getScriptProperties()
  .getProperty('API_KEY');

// 避免在日志中暴露敏感信息
console.log('Processing user: ' + maskEmail(userEmail));
```

3. **错误处理**
```javascript
function robustExecution() {
  try {
    // 主要逻辑
    performOperation();
  } catch (error) {
    // 记录错误但不暴露内部细节
    console.error('Operation failed', error.message);
    // 向用户返回友好错误信息
    return '操作失败，请稍后重试';
  }
}
```

## 7.6 调试工具与性能分析

### 7.6.1 调试工具链

**1. 日志系统**
```javascript
// 不同级别的日志
console.log('信息日志');
console.warn('警告信息');
console.error('错误信息');

// 结构化日志
console.log({
  timestamp: new Date(),
  action: 'processData',
  duration: executionTime,
  recordsProcessed: count
});
```

**2. 断点调试**
```javascript
function complexCalculation(data) {
  debugger;  // 在支持的环境中设置断点
  
  const processed = preprocess(data);
  // 步进执行，检查变量值
  const result = calculate(processed);
  
  return result;
}
```

**3. 执行记录追踪**
```
执行ID: abc123
开始时间: 2024-01-20 10:00:00
触发器类型: onEdit
用户: user@example.com

[10:00:00.100] 开始执行
[10:00:00.150] 验证权限
[10:00:00.200] 读取数据范围 A1:C100
[10:00:00.500] 处理数据
[10:00:01.000] 写入结果
[10:00:01.100] 执行完成

总耗时: 1000ms
API调用: 5次
```

### 7.6.2 性能分析方法

**1. 执行时间分析**
```javascript
function performanceAnalysis() {
  const profiler = new Profiler();
  
  profiler.start('dataFetch');
  const data = fetchData();
  profiler.end('dataFetch');
  
  profiler.start('processing');
  const result = processData(data);
  profiler.end('processing');
  
  profiler.start('writing');
  writeResults(result);
  profiler.end('writing');
  
  console.log(profiler.getReport());
}

class Profiler {
  constructor() {
    this.timings = {};
  }
  
  start(label) {
    this.timings[label] = Date.now();
  }
  
  end(label) {
    this.timings[label] = Date.now() - this.timings[label];
  }
  
  getReport() {
    return this.timings;
  }
}
```

**2. 内存使用监控**
```javascript
function memoryMonitor() {
  const baseline = getMemoryUsage();
  
  // 执行操作
  const largeData = generateLargeDataset();
  
  const peak = getMemoryUsage();
  console.log(`内存增长: ${peak - baseline} bytes`);
  
  // 清理
  largeData = null;
  Utilities.sleep(100);  // 等待GC
  
  const after = getMemoryUsage();
  console.log(`内存释放: ${peak - after} bytes`);
}
```

### 7.6.3 性能优化技巧

**1. 缓存策略**
```javascript
// 使用缓存减少重复计算
const cache = CacheService.getScriptCache();

function expensiveOperation(key) {
  // 检查缓存
  const cached = cache.get(key);
  if (cached) {
    return JSON.parse(cached);
  }
  
  // 执行计算
  const result = performCalculation(key);
  
  // 存入缓存（TTL: 600秒）
  cache.put(key, JSON.stringify(result), 600);
  
  return result;
}
```

**2. 批量操作优化**
```javascript
// 低效：逐个单元格操作
function inefficientUpdate(data) {
  for (let i = 0; i < data.length; i++) {
    for (let j = 0; j < data[i].length; j++) {
      sheet.getRange(i+1, j+1).setValue(data[i][j]);
    }
  }
}

// 高效：批量更新
function efficientUpdate(data) {
  const range = sheet.getRange(1, 1, data.length, data[0].length);
  range.setValues(data);
}
```

**3. 异步并发处理**
```javascript
function parallelProcessing(urls) {
  // 并发发起请求
  const requests = urls.map(url => ({
    url: url,
    method: 'GET',
    muteHttpExceptions: true
  }));
  
  // 批量获取响应
  const responses = UrlFetchApp.fetchAll(requests);
  
  // 处理结果
  return responses.map(response => 
    JSON.parse(response.getContentText())
  );
}
```

### 7.6.4 常见性能陷阱

1. **过度的API调用**
   - 问题：循环中重复调用 getRange()
   - 解决：预先获取范围，批量操作

2. **大数据集处理**
   - 问题：一次性加载过多数据
   - 解决：分页处理，流式处理

3. **同步阻塞操作**
   - 问题：串行处理多个独立任务
   - 解决：使用 fetchAll() 等并发API

4. **内存泄漏**
   - 问题：闭包持有大对象引用
   - 解决：及时清理，避免全局变量

## 本章小结

本章深入探讨了电子表格的脚本化与自动化编程能力，从Google Apps Script的架构设计到飞书多维表格的低代码平台，涵盖了以下核心概念：

1. **架构设计**：理解了GAS的分层架构、API设计哲学和服务集成模式
2. **事件模型**：掌握了触发器机制、事件对象结构和级联效应处理
3. **扩展机制**：学习了自定义函数和宏的实现原理及优化策略
4. **平台对比**：对比了GAS与飞书的不同设计理念和实现方式
5. **安全机制**：理解了沙箱隔离、权限控制和执行限制
6. **调试优化**：掌握了调试工具使用和性能优化方法

**关键洞察**：
- 脚本化将表格从数据容器升级为应用平台
- 低代码降低了自动化的门槛但不应牺牲灵活性
- 安全和性能是脚本化系统的两大挑战
- 生态集成能力决定了平台的应用广度

## 练习题

### 基础题

**练习7.1：触发器选择**
某公司需要在每天凌晨2点自动汇总前一天的销售数据并发送邮件报告。请问应该使用哪种类型的触发器？说明原因。

<details>
<summary>提示</summary>
考虑触发器的权限要求和执行时限
</details>

<details>
<summary>答案</summary>

应使用可安装的时间驱动触发器（Installable Time-driven Trigger）。

原因：
1. 需要定时执行（每天凌晨2点）
2. 需要发送邮件（需要Gmail服务授权）
3. 可能需要较长执行时间（数据汇总）
4. 简单触发器不支持时间驱动且无法访问需授权的服务

实现代码框架：
```javascript
function installDailyTrigger() {
  ScriptApp.newTrigger('dailySalesReport')
    .timeBased()
    .atHour(2)
    .everyDays(1)
    .create();
}

function dailySalesReport() {
  // 汇总数据
  const data = aggregateSalesData();
  // 生成报告
  const report = generateReport(data);
  // 发送邮件
  sendEmailReport(report);
}
```
</details>

**练习7.2：性能优化**
以下代码用于更新1000行数据的状态列，请识别性能问题并提供优化方案。

```javascript
function updateStatus(sheet) {
  for (let i = 2; i <= 1001; i++) {
    const value = sheet.getRange(i, 1).getValue();
    if (value > 100) {
      sheet.getRange(i, 5).setValue("高");
      sheet.getRange(i, 5).setBackground("red");
    } else {
      sheet.getRange(i, 5).setValue("低");
      sheet.getRange(i, 5).setBackground("green");
    }
  }
}
```

<details>
<summary>提示</summary>
考虑批量操作和减少API调用次数
</details>

<details>
<summary>答案</summary>

性能问题：
1. 循环中多次调用 getRange() 和 getValue()
2. 逐个单元格设置值和背景色
3. 总共约4000次API调用

优化方案：
```javascript
function updateStatusOptimized(sheet) {
  // 一次性读取所有数据
  const data = sheet.getRange(2, 1, 1000, 1).getValues();
  
  // 准备输出数组
  const statuses = [];
  const colors = [];
  
  // 批量处理
  for (let i = 0; i < data.length; i++) {
    if (data[i][0] > 100) {
      statuses.push(["高"]);
      colors.push(["red"]);
    } else {
      statuses.push(["低"]);
      colors.push(["green"]);
    }
  }
  
  // 批量写入
  sheet.getRange(2, 5, 1000, 1).setValues(statuses);
  sheet.getRange(2, 5, 1000, 1).setBackgrounds(colors);
}
```

性能提升：从4000次API调用减少到4次，执行速度提升约100倍。
</details>

**练习7.3：权限设计**
设计一个脚本的权限配置，该脚本需要：
1. 读取当前表格数据
2. 向外部API发送数据
3. 在用户的云盘中创建备份文件
4. 不需要访问用户的邮件

<details>
<summary>提示</summary>
考虑最小权限原则
</details>

<details>
<summary>答案</summary>

权限配置：
```javascript
{
  "oauthScopes": [
    "https://www.googleapis.com/auth/spreadsheets.currentonly",
    "https://www.googleapis.com/auth/drive.file",
    "https://www.googleapis.com/auth/script.external_request"
  ],
  "urlFetchWhitelist": [
    "https://api.company.com/*"
  ]
}
```

说明：
1. `spreadsheets.currentonly`：仅访问当前表格
2. `drive.file`：仅访问脚本创建的文件
3. `script.external_request`：允许外部HTTP请求
4. URL白名单限制只能访问特定API

这遵循了最小权限原则，避免了不必要的权限（如访问所有表格或邮件）。
</details>

### 挑战题

**练习7.4：设计事件驱动的库存管理系统**
设计一个基于触发器的库存管理系统，要求：
1. 当库存低于安全库存时自动创建采购单
2. 每日生成库存报表
3. 支持多仓库数据同步
4. 提供库存预警通知

请设计系统架构和主要触发器逻辑。

<details>
<summary>提示</summary>
考虑不同类型触发器的组合使用和数据一致性
</details>

<details>
<summary>答案</summary>

系统架构设计：

```
┌─────────────────────────────────────┐
│         库存主表                     │
│  (产品ID, 名称, 当前库存, 安全库存)  │
└─────────────────────────────────────┘
            ↓ onEdit触发器
┌─────────────────────────────────────┐
│      库存监控脚本                    │
│  - 检查安全库存                      │
│  - 生成采购建议                      │
└─────────────────────────────────────┘
            ↓
┌─────────────────────────────────────┐
│       采购单表                       │
│  (自动生成的采购申请)                │
└─────────────────────────────────────┘

时间触发器：
- 每日报表生成
- 多仓库同步
```

主要触发器实现：

```javascript
// 1. 库存变更触发器
function onInventoryChange(e) {
  const range = e.range;
  const sheet = range.getSheet();
  
  if (sheet.getName() !== "库存表") return;
  
  const row = range.getRow();
  const currentStock = sheet.getRange(row, 3).getValue();
  const safetyStock = sheet.getRange(row, 4).getValue();
  
  if (currentStock < safetyStock) {
    createPurchaseOrder(row);
    sendLowStockAlert(row);
  }
}

// 2. 定时报表触发器
function dailyInventoryReport() {
  const report = generateInventoryReport();
  saveReportToDrive(report);
  emailReportToManagers(report);
}

// 3. 多仓库同步触发器
function syncWarehouses() {
  const warehouses = ["仓库A", "仓库B", "仓库C"];
  const consolidated = {};
  
  warehouses.forEach(warehouse => {
    const data = fetchWarehouseData(warehouse);
    mergeInventoryData(consolidated, data);
  });
  
  updateMasterInventory(consolidated);
}

// 4. 触发器安装
function setupTriggers() {
  // 编辑触发器
  ScriptApp.newTrigger('onInventoryChange')
    .forSpreadsheet(SpreadsheetApp.getActive())
    .onEdit()
    .create();
  
  // 每日报表
  ScriptApp.newTrigger('dailyInventoryReport')
    .timeBased()
    .atHour(8)
    .everyDays(1)
    .create();
  
  // 仓库同步（每小时）
  ScriptApp.newTrigger('syncWarehouses')
    .timeBased()
    .everyHours(1)
    .create();
}
```

数据一致性保证：
1. 使用事务性更新
2. 加锁机制防止并发冲突
3. 异常重试机制
4. 审计日志记录所有变更
</details>

**练习7.5：实现自定义函数缓存系统**
实现一个通用的自定义函数缓存系统，要求：
1. 支持设置缓存过期时间
2. 支持不同的缓存键生成策略
3. 提供缓存命中率统计
4. 支持手动清理缓存

<details>
<summary>提示</summary>
考虑使用CacheService和自定义的缓存管理逻辑
</details>

<details>
<summary>答案</summary>

```javascript
/**
 * 通用缓存系统实现
 */
class FunctionCache {
  constructor(namespace = 'default') {
    this.cache = CacheService.getScriptCache();
    this.namespace = namespace;
    this.stats = {
      hits: 0,
      misses: 0,
      sets: 0
    };
  }
  
  /**
   * 生成缓存键
   */
  generateKey(functionName, args, strategy = 'json') {
    const prefix = `${this.namespace}:${functionName}:`;
    
    switch(strategy) {
      case 'json':
        return prefix + JSON.stringify(args);
      case 'hash':
        return prefix + this.hashCode(JSON.stringify(args));
      case 'custom':
        return prefix + args.map(a => String(a)).join(':');
      default:
        return prefix + JSON.stringify(args);
    }
  }
  
  /**
   * 缓存装饰器
   */
  memoize(fn, options = {}) {
    const {
      ttl = 600,  // 默认10分钟
      keyStrategy = 'json',
      shouldCache = () => true
    } = options;
    
    const cache = this;
    
    return function(...args) {
      // 检查是否应该缓存
      if (!shouldCache(args)) {
        return fn.apply(this, args);
      }
      
      const key = cache.generateKey(fn.name, args, keyStrategy);
      
      // 尝试从缓存获取
      const cached = cache.get(key);
      if (cached !== null) {
        cache.stats.hits++;
        return cached;
      }
      
      // 执行函数
      cache.stats.misses++;
      const result = fn.apply(this, args);
      
      // 存入缓存
      cache.set(key, result, ttl);
      
      return result;
    };
  }
  
  /**
   * 获取缓存值
   */
  get(key) {
    const value = this.cache.get(key);
    if (value) {
      try {
        return JSON.parse(value);
      } catch {
        return value;
      }
    }
    return null;
  }
  
  /**
   * 设置缓存值
   */
  set(key, value, ttl) {
    this.stats.sets++;
    const serialized = typeof value === 'object' 
      ? JSON.stringify(value) 
      : String(value);
    this.cache.put(key, serialized, ttl);
  }
  
  /**
   * 清理缓存
   */
  clear(pattern) {
    if (!pattern) {
      // 清理所有（通过设置空值）
      this.cache.removeAll([]);
    } else {
      // 不支持模式匹配，需要追踪所有键
      console.warn('Pattern-based clearing not supported');
    }
  }
  
  /**
   * 获取统计信息
   */
  getStats() {
    const total = this.stats.hits + this.stats.misses;
    return {
      ...this.stats,
      hitRate: total > 0 ? this.stats.hits / total : 0,
      total: total
    };
  }
  
  /**
   * 哈希函数
   */
  hashCode(str) {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      const char = str.charCodeAt(i);
      hash = ((hash << 5) - hash) + char;
      hash = hash & hash;
    }
    return String(hash);
  }
}

// 使用示例
const cache = new FunctionCache('myapp');

// 原始函数
function expensiveCalculation(n) {
  // 模拟耗时计算
  Utilities.sleep(1000);
  return n * n;
}

// 包装为缓存版本
const cachedCalculation = cache.memoize(expensiveCalculation, {
  ttl: 300,  // 5分钟缓存
  keyStrategy: 'custom',
  shouldCache: (args) => args[0] > 0  // 只缓存正数
});

// 自定义函数中使用
function CACHED_CALC(n) {
  return cachedCalculation(n);
}

// 查看统计
function logCacheStats() {
  console.log(cache.getStats());
  // 输出: {hits: 10, misses: 5, sets: 5, hitRate: 0.67, total: 15}
}
```

这个缓存系统提供了：
1. 灵活的缓存键生成策略
2. 可配置的过期时间
3. 条件缓存（shouldCache）
4. 统计信息收集
5. 装饰器模式便于使用
</details>

**练习7.6：设计跨表格数据同步方案**
设计一个方案，实现多个表格之间的双向数据同步，要求：
1. 支持冲突检测和解决
2. 保证最终一致性
3. 提供同步状态监控
4. 支持选择性同步（只同步特定列或行）

<details>
<summary>提示</summary>
考虑使用版本向量、时间戳和冲突解决策略
</details>

<details>
<summary>答案</summary>

```javascript
/**
 * 跨表格双向同步系统
 */
class CrossSheetSync {
  constructor(config) {
    this.sheets = config.sheets;  // [{id, name, range}]
    this.syncColumns = config.columns;  // 要同步的列
    this.conflictStrategy = config.conflictStrategy || 'last-write-wins';
    this.syncLog = [];
  }
  
  /**
   * 主同步流程
   */
  async performSync() {
    const syncId = Utilities.getUuid();
    console.log(`开始同步: ${syncId}`);
    
    try {
      // 1. 收集所有表格的当前状态
      const snapshots = this.collectSnapshots();
      
      // 2. 检测变更
      const changes = this.detectChanges(snapshots);
      
      // 3. 检测冲突
      const conflicts = this.detectConflicts(changes);
      
      // 4. 解决冲突
      const resolved = this.resolveConflicts(conflicts);
      
      // 5. 应用变更
      this.applyChanges(resolved);
      
      // 6. 更新元数据
      this.updateMetadata(syncId);
      
      // 7. 记录日志
      this.logSync(syncId, resolved);
      
      return {
        success: true,
        syncId: syncId,
        changesApplied: resolved.length
      };
      
    } catch (error) {
      console.error(`同步失败: ${error}`);
      this.rollback(syncId);
      throw error;
    }
  }
  
  /**
   * 收集数据快照
   */
  collectSnapshots() {
    return this.sheets.map(sheet => {
      const ss = SpreadsheetApp.openById(sheet.id);
      const ws = ss.getSheetByName(sheet.name);
      const data = ws.getRange(sheet.range).getValues();
      
      // 获取元数据（时间戳、版本等）
      const metadata = this.getMetadata(ws);
      
      return {
        sheetId: sheet.id,
        data: data,
        metadata: metadata,
        checksum: this.calculateChecksum(data)
      };
    });
  }
  
  /**
   * 检测变更
   */
  detectChanges(snapshots) {
    const changes = [];
    
    snapshots.forEach((snapshot, index) => {
      const lastSync = this.getLastSyncState(snapshot.sheetId);
      
      if (!lastSync || lastSync.checksum !== snapshot.checksum) {
        // 逐行比较找出具体变更
        const rowChanges = this.compareRows(
          lastSync?.data || [],
          snapshot.data
        );
        
        rowChanges.forEach(change => {
          changes.push({
            sheetId: snapshot.sheetId,
            row: change.row,
            column: change.column,
            oldValue: change.oldValue,
            newValue: change.newValue,
            timestamp: snapshot.metadata.lastModified,
            version: snapshot.metadata.version
          });
        });
      }
    });
    
    return changes;
  }
  
  /**
   * 检测冲突
   */
  detectConflicts(changes) {
    const conflicts = [];
    const grouped = this.groupChangesByCell(changes);
    
    Object.keys(grouped).forEach(cellKey => {
      const cellChanges = grouped[cellKey];
      
      if (cellChanges.length > 1) {
        // 多个表格修改了同一单元格
        conflicts.push({
          cell: cellKey,
          changes: cellChanges,
          type: this.classifyConflict(cellChanges)
        });
      }
    });
    
    return conflicts;
  }
  
  /**
   * 解决冲突
   */
  resolveConflicts(conflicts) {
    const resolved = [];
    
    conflicts.forEach(conflict => {
      let winner;
      
      switch(this.conflictStrategy) {
        case 'last-write-wins':
          // 选择时间戳最新的
          winner = conflict.changes.reduce((a, b) => 
            a.timestamp > b.timestamp ? a : b
          );
          break;
          
        case 'highest-version':
          // 选择版本号最高的
          winner = conflict.changes.reduce((a, b) => 
            a.version > b.version ? a : b
          );
          break;
          
        case 'merge':
          // 尝试合并（适用于可合并的数据类型）
          winner = this.mergeChanges(conflict.changes);
          break;
          
        case 'manual':
          // 需要人工介入
          winner = this.promptUserResolution(conflict);
          break;
          
        default:
          winner = conflict.changes[0];
      }
      
      resolved.push({
        ...winner,
        conflictResolved: true,
        conflictType: conflict.type
      });
    });
    
    return resolved;
  }
  
  /**
   * 应用变更到所有表格
   */
  applyChanges(changes) {
    // 按表格分组变更
    const changesBySheet = this.groupBySheet(changes);
    
    this.sheets.forEach(sheet => {
      const sheetChanges = changesBySheet[sheet.id] || [];
      
      if (sheetChanges.length > 0) {
        const ss = SpreadsheetApp.openById(sheet.id);
        const ws = ss.getSheetByName(sheet.name);
        
        // 批量应用变更
        const batch = this.prepareBatchUpdate(sheetChanges);
        ws.getRange(sheet.range).setValues(batch);
        
        // 更新版本号
        this.incrementVersion(ws);
      }
    });
  }
  
  /**
   * 元数据管理
   */
  getMetadata(sheet) {
    const metaRange = sheet.getRange('Metadata!A1:B10');
    const metaData = metaRange.getValues();
    
    return {
      lastModified: new Date(metaData[0][1]),
      version: parseInt(metaData[1][1]),
      lastSyncId: metaData[2][1],
      checksum: metaData[3][1]
    };
  }
  
  updateMetadata(syncId) {
    this.sheets.forEach(sheet => {
      const ss = SpreadsheetApp.openById(sheet.id);
      const ws = ss.getSheetByName('Metadata');
      
      ws.getRange('B1').setValue(new Date());
      ws.getRange('B3').setValue(syncId);
      ws.getRange('B4').setValue(this.calculateChecksum(
        ss.getSheetByName(sheet.name).getRange(sheet.range).getValues()
      ));
    });
  }
  
  /**
   * 计算数据校验和
   */
  calculateChecksum(data) {
    const str = JSON.stringify(data);
    let hash = 0;
    
    for (let i = 0; i < str.length; i++) {
      const char = str.charCodeAt(i);
      hash = ((hash << 5) - hash) + char;
      hash = hash & hash;
    }
    
    return hash.toString(16);
  }
  
  /**
   * 监控面板数据
   */
  getSyncStatus() {
    return {
      lastSync: this.getLastSyncTime(),
      pendingChanges: this.countPendingChanges(),
      conflictCount: this.getUnresolvedConflicts().length,
      syncHistory: this.syncLog.slice(-10),
      sheetStatus: this.sheets.map(sheet => ({
        id: sheet.id,
        name: sheet.name,
        lastModified: this.getSheetLastModified(sheet.id),
        version: this.getSheetVersion(sheet.id),
        inSync: this.isSheetInSync(sheet.id)
      }))
    };
  }
}

// 使用配置
const syncConfig = {
  sheets: [
    {id: 'sheet1_id', name: 'Data', range: 'A1:E100'},
    {id: 'sheet2_id', name: 'Data', range: 'A1:E100'},
    {id: 'sheet3_id', name: 'Data', range: 'A1:E100'}
  ],
  columns: ['A', 'B', 'C'],  // 只同步这些列
  conflictStrategy: 'last-write-wins'
};

// 创建同步实例
const syncer = new CrossSheetSync(syncConfig);

// 设置定时同步
function setupAutoSync() {
  ScriptApp.newTrigger('performSync')
    .timeBased()
    .everyMinutes(5)
    .create();
}

// 执行同步
function performSync() {
  try {
    const result = syncer.performSync();
    console.log(`同步成功: ${result.changesApplied} 个变更`);
  } catch (error) {
    console.error(`同步失败: ${error}`);
    // 发送告警
    sendAlert(error);
  }
}
```

这个方案实现了：
1. 基于版本向量的冲突检测
2. 多种冲突解决策略
3. 批量变更应用以提高性能
4. 完整的同步状态监控
5. 选择性列同步支持
6. 回滚机制保证数据安全
</details>

**练习7.7：实现脚本性能监控系统**
设计一个全面的脚本性能监控系统，能够：
1. 自动收集执行时间、API调用次数等指标
2. 识别性能瓶颈
3. 生成性能报告
4. 提供优化建议

<details>
<summary>提示</summary>
考虑使用装饰器模式和执行时间分析
</details>

<details>
<summary>答案</summary>

```javascript
/**
 * 脚本性能监控系统
 */
class PerformanceMonitor {
  constructor() {
    this.metrics = {
      executions: [],
      apiCalls: {},
      errors: [],
      memoryUsage: []
    };
    this.thresholds = {
      executionTime: 3000,  // 3秒
      apiCallsPerExecution: 100,
      errorRate: 0.05
    };
  }
  
  /**
   * 监控装饰器
   */
  monitor(targetFunction, functionName) {
    const monitor = this;
    
    return function(...args) {
      const execution = {
        functionName: functionName || targetFunction.name,
        startTime: Date.now(),
        apiCalls: {},
        errors: [],
        result: null
      };
      
      // 拦截API调用
      const originalAPIs = monitor.interceptAPIs(execution);
      
      try {
        // 执行目标函数
        execution.result = targetFunction.apply(this, args);
        
      } catch (error) {
        execution.errors.push({
          message: error.message,
          stack: error.stack,
          timestamp: Date.now()
        });
        throw error;
        
      } finally {
        // 恢复原始API
        monitor.restoreAPIs(originalAPIs);
        
        // 记录执行信息
        execution.endTime = Date.now();
        execution.duration = execution.endTime - execution.startTime;
        execution.memoryUsed = monitor.estimateMemoryUsage();
        
        // 保存指标
        monitor.metrics.executions.push(execution);
        
        // 分析性能
        monitor.analyzePerformance(execution);
      }
      
      return execution.result;
    };
  }
  
  /**
   * 拦截API调用以收集指标
   */
  interceptAPIs(execution) {
    const apis = {
      'SpreadsheetApp.getActiveSheet': SpreadsheetApp.getActiveSheet,
      'Sheet.getRange': Sheet.prototype.getRange,
      'Range.getValues': Range.prototype.getValues,
      'Range.setValues': Range.prototype.setValues
    };
    
    const original = {};
    
    Object.keys(apis).forEach(apiName => {
      original[apiName] = apis[apiName];
      
      // 包装API
      const parts = apiName.split('.');
      const obj = parts[0] === 'SpreadsheetApp' 
        ? SpreadsheetApp 
        : window[parts[0]].prototype;
      const method = parts[1];
      
      obj[method] = function(...args) {
        // 记录调用
        execution.apiCalls[apiName] = 
          (execution.apiCalls[apiName] || 0) + 1;
        
        // 调用原始方法
        return original[apiName].apply(this, args);
      };
    });
    
    return original;
  }
  
  /**
   * 恢复原始API
   */
  restoreAPIs(original) {
    Object.keys(original).forEach(apiName => {
      const parts = apiName.split('.');
      const obj = parts[0] === 'SpreadsheetApp' 
        ? SpreadsheetApp 
        : window[parts[0]].prototype;
      const method = parts[1];
      
      obj[method] = original[apiName];
    });
  }
  
  /**
   * 分析性能问题
   */
  analyzePerformance(execution) {
    const issues = [];
    
    // 检查执行时间
    if (execution.duration > this.thresholds.executionTime) {
      issues.push({
        type: 'SLOW_EXECUTION',
        severity: 'HIGH',
        message: `执行时间 ${execution.duration}ms 超过阈值`,
        suggestion: '考虑优化算法或使用批量操作'
      });
    }
    
    // 检查API调用次数
    const totalAPICalls = Object.values(execution.apiCalls)
      .reduce((a, b) => a + b, 0);
    
    if (totalAPICalls > this.thresholds.apiCallsPerExecution) {
      issues.push({
        type: 'EXCESSIVE_API_CALLS',
        severity: 'MEDIUM',
        message: `API调用 ${totalAPICalls} 次超过阈值`,
        suggestion: '使用批量操作减少API调用'
      });
    }
    
    // 检查重复调用
    Object.entries(execution.apiCalls).forEach(([api, count]) => {
      if (count > 10) {
        issues.push({
          type: 'REPEATED_API_CALLS',
          severity: 'MEDIUM',
          message: `${api} 被调用 ${count} 次`,
          suggestion: `考虑缓存 ${api} 的结果`
        });
      }
    });
    
    execution.performanceIssues = issues;
    
    // 如果有严重问题，记录到日志
    if (issues.some(i => i.severity === 'HIGH')) {
      console.warn('性能问题检测:', issues);
    }
  }
  
  /**
   * 生成性能报告
   */
  generateReport() {
    const report = {
      summary: this.generateSummary(),
      topSlowFunctions: this.getTopSlowFunctions(),
      apiUsageStats: this.getAPIUsageStats(),
      errorAnalysis: this.getErrorAnalysis(),
      recommendations: this.generateRecommendations(),
      trends: this.analyzeTrends()
    };
    
    return report;
  }
  
  /**
   * 生成摘要统计
   */
  generateSummary() {
    const executions = this.metrics.executions;
    const durations = executions.map(e => e.duration);
    
    return {
      totalExecutions: executions.length,
      avgDuration: this.average(durations),
      maxDuration: Math.max(...durations),
      minDuration: Math.min(...durations),
      p95Duration: this.percentile(durations, 95),
      totalErrors: this.metrics.errors.length,
      errorRate: this.metrics.errors.length / executions.length
    };
  }
  
  /**
   * 获取最慢的函数
   */
  getTopSlowFunctions(limit = 5) {
    const functionStats = {};
    
    this.metrics.executions.forEach(exec => {
      if (!functionStats[exec.functionName]) {
        functionStats[exec.functionName] = {
          name: exec.functionName,
          executions: 0,
          totalTime: 0,
          avgTime: 0,
          maxTime: 0
        };
      }
      
      const stats = functionStats[exec.functionName];
      stats.executions++;
      stats.totalTime += exec.duration;
      stats.avgTime = stats.totalTime / stats.executions;
      stats.maxTime = Math.max(stats.maxTime, exec.duration);
    });
    
    return Object.values(functionStats)
      .sort((a, b) => b.avgTime - a.avgTime)
      .slice(0, limit);
  }
  
  /**
   * API使用统计
   */
  getAPIUsageStats() {
    const apiStats = {};
    
    this.metrics.executions.forEach(exec => {
      Object.entries(exec.apiCalls).forEach(([api, count]) => {
        if (!apiStats[api]) {
          apiStats[api] = {
            api: api,
            totalCalls: 0,
            avgCallsPerExecution: 0,
            executions: 0
          };
        }
        
        apiStats[api].totalCalls += count;
        apiStats[api].executions++;
        apiStats[api].avgCallsPerExecution = 
          apiStats[api].totalCalls / apiStats[api].executions;
      });
    });
    
    return Object.values(apiStats)
      .sort((a, b) => b.totalCalls - a.totalCalls);
  }
  
  /**
   * 生成优化建议
   */
  generateRecommendations() {
    const recommendations = [];
    const stats = this.generateSummary();
    const apiStats = this.getAPIUsageStats();
    
    // 基于统计生成建议
    if (stats.p95Duration > 5000) {
      recommendations.push({
        priority: 'HIGH',
        category: 'PERFORMANCE',
        suggestion: '考虑将长时间运行的任务分解为多个小任务',
        impact: '可减少50%以上的执行时间'
      });
    }
    
    // 检查高频API调用
    const highFreqAPIs = apiStats.filter(a => 
      a.avgCallsPerExecution > 20
    );
    
    if (highFreqAPIs.length > 0) {
      recommendations.push({
        priority: 'MEDIUM',
        category: 'API_USAGE',
        suggestion: `优化以下高频API调用: ${
          highFreqAPIs.map(a => a.api).join(', ')
        }`,
        impact: '可减少80%的API调用'
      });
    }
    
    // 错误率建议
    if (stats.errorRate > this.thresholds.errorRate) {
      recommendations.push({
        priority: 'HIGH',
        category: 'RELIABILITY',
        suggestion: '添加更多错误处理和重试机制',
        impact: '提高系统可靠性'
      });
    }
    
    return recommendations;
  }
  
  /**
   * 趋势分析
   */
  analyzeTrends() {
    const recentExecutions = this.metrics.executions.slice(-100);
    const windows = this.slidingWindow(recentExecutions, 10);
    
    return {
      performanceTrend: this.calculateTrend(
        windows.map(w => this.average(w.map(e => e.duration)))
      ),
      errorTrend: this.calculateTrend(
        windows.map(w => w.filter(e => e.errors.length > 0).length)
      ),
      apiCallTrend: this.calculateTrend(
        windows.map(w => this.average(
          w.map(e => Object.values(e.apiCalls).reduce((a, b) => a + b, 0))
        ))
      )
    };
  }
  
  /**
   * 辅助函数
   */
  average(arr) {
    return arr.reduce((a, b) => a + b, 0) / arr.length;
  }
  
  percentile(arr, p) {
    const sorted = arr.sort((a, b) => a - b);
    const index = Math.ceil(sorted.length * p / 100) - 1;
    return sorted[index];
  }
  
  slidingWindow(arr, size) {
    const windows = [];
    for (let i = 0; i <= arr.length - size; i++) {
      windows.push(arr.slice(i, i + size));
    }
    return windows;
  }
  
  calculateTrend(values) {
    if (values.length < 2) return 'stable';
    
    const first = values.slice(0, Math.floor(values.length / 2));
    const second = values.slice(Math.floor(values.length / 2));
    
    const avgFirst = this.average(first);
    const avgSecond = this.average(second);
    
    const change = (avgSecond - avgFirst) / avgFirst;
    
    if (change > 0.1) return 'increasing';
    if (change < -0.1) return 'decreasing';
    return 'stable';
  }
  
  /**
   * 估算内存使用
   */
  estimateMemoryUsage() {
    // 简化的内存估算
    // 实际实现需要更复杂的方法
    return Math.random() * 1000000;  // 字节
  }
}

// 使用示例
const monitor = new PerformanceMonitor();

// 包装函数以监控性能
const monitoredFunction = monitor.monitor(function processData() {
  const sheet = SpreadsheetApp.getActiveSheet();
  const data = sheet.getRange('A1:Z1000').getValues();
  
  // 处理数据...
  for (let i = 0; i < data.length; i++) {
    // 模拟处理
  }
  
  sheet.getRange('AA1:AA1000').setValues(results);
}, 'processData');

// 生成报告
function generatePerformanceReport() {
  const report = monitor.generateReport();
  
  console.log('性能报告:');
  console.log('摘要:', report.summary);
  console.log('最慢函数:', report.topSlowFunctions);
  console.log('优化建议:', report.recommendations);
  console.log('趋势分析:', report.trends);
  
  // 保存到表格
  saveReportToSheet(report);
}
```

这个监控系统提供了：
1. 自动的性能指标收集
2. API调用拦截和统计
3. 性能问题自动识别
4. 详细的性能报告生成
5. 趋势分析和预测
6. 可操作的优化建议
</details>

## 常见陷阱与错误

### 陷阱1：忽视执行时限
```javascript
// 错误：可能超时的代码
function processAllRows() {
  const data = sheet.getDataRange().getValues();
  for (let row of data) {
    complexProcessing(row);  // 每行需要1秒
  }
}

// 正确：分批处理
function processBatch(startRow = 1) {
  const batchSize = 100;
  const data = sheet.getRange(startRow, 1, batchSize, 10).getValues();
  
  for (let row of data) {
    complexProcessing(row);
  }
  
  if (data.length === batchSize) {
    // 设置触发器处理下一批
    scheduleContinuation(startRow + batchSize);
  }
}
```

### 陷阱2：过度使用全局变量
```javascript
// 错误：依赖全局状态
let counter = 0;  // 每次执行会重置

function incrementCounter() {
  counter++;  // 不会持久化
}

// 正确：使用属性服务
function incrementCounter() {
  const props = PropertiesService.getScriptProperties();
  let counter = parseInt(props.getProperty('counter') || '0');
  counter++;
  props.setProperty('counter', String(counter));
}
```

### 陷阱3：同步思维处理异步问题
```javascript
// 错误：假设操作立即完成
function updateAndRead() {
  sheet.getRange('A1').setValue('新值');
  const value = sheet.getRange('A1').getValue();  // 可能还是旧值
}

// 正确：使用flush确保同步
function updateAndRead() {
  sheet.getRange('A1').setValue('新值');
  SpreadsheetApp.flush();  // 强制应用更改
  const value = sheet.getRange('A1').getValue();
}
```

### 陷阱4：权限理解错误
```javascript
// 错误：简单触发器中使用需授权的服务
function onEdit(e) {
  MailApp.sendEmail(...);  // 会失败
}

// 正确：使用可安装触发器
function installableOnEdit(e) {
  MailApp.sendEmail(...);  // 可以工作
}
```

### 陷阱5：循环触发器
```javascript
// 错误：onChange触发器修改数据导致再次触发
function onChange(e) {
  sheet.getRange('A1').setValue(new Date());  // 触发新的onChange
}

// 正确：使用标记避免循环
function onChange(e) {
  const flag = sheet.getRange('Z1').getValue();
  if (flag === 'processing') return;
  
  sheet.getRange('Z1').setValue('processing');
  sheet.getRange('A1').setValue(new Date());
  sheet.getRange('Z1').setValue('');
}
```

---

通过本章的学习，您已经掌握了电子表格脚本化和自动化编程的核心技术。这些知识将帮助您构建更强大、更智能的表格应用，充分发挥现代协作平台的潜力。下一章，我们将探讨可视化与仪表板的设计原理。