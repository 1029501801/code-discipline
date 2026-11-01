---
name: code-discipline
description: |
  Constraints for writing or modifying code inside an existing codebase. Prevents four
  recurring AI failure modes: re-implementing utilities that already exist, blanket
  defensive coding (null checks / try-catch / default values everywhere), premature
  abstraction (interface + Factory + Strategy for a single implementation), and code
  that is locally correct but globally unreadable.
  TRIGGER when: adding a feature to an existing project, modifying business logic,
  writing helper or util functions, refactoring, reviewing a diff/PR, deciding whether
  to extract an abstraction, add validation, or catch an exception, designing module
  and class layout, or when the user says 复用 / 抽象 / 重构 / 过度设计 / 代码太乱 /
  看不懂这段代码 / 这段能不能抽出来.
  DO NOT TRIGGER when: writing a throwaway script, single-file demo or prototype,
  or the user explicitly asks for a full greenfield architecture design.
description_zh: "克制的代码：约束重复造轮子、过度防御、过度抽象、结构不自明"
description_en: "Code discipline: no reinvention, no blanket defensiveness, no premature abstraction"
license: MIT
metadata:
  category: code-quality
  version: "1.0.0"
---

# 克制的代码

## 核心判断

这四类问题同源：**用局部上下文做全局决策**。

| 症状 | 真实原因 |
|---|---|
| 重复造轮子 | 看不见项目里已有的实现 |
| 无差别防御 | 不知道校验、异常、非空约束的责任边界在哪 |
| 提前抽象 | 不确定未来会不会变，用抽象对冲焦虑 |
| 结构不自明 | 无法确认影响范围，只能层层包裹，谁也不敢删 |

因此本 skill 的主线不是"写得更优雅"，而是：**先扩大上下文，再缩小改动面**。凡是拿不准的地方，优先去读代码，而不是加一层保险。

## 最小执行路径

时间紧、只想记住三件事时，执行这三条即可覆盖大部分收益：

1. 动手前跑一次 `scripts/scan_reuse.py`，确认项目里没有现成实现。
2. 只在校验入口做校验，函数内部信任契约；配置缺失就抛错。
3. 交付前做一次删减试验：删掉新加的辅助方法，主流程仍读得懂就真的删掉。

## 第 0 步：定性

两类任务的规则相反，先判定再动手。

| | 业务实现 | 通用能力 |
|---|---|---|
| 典型 | 订单取消、报表导出、页面接口 | 脚手架、工具库、中间件、SDK |
| 目标 | 尽快正确落地 | 可被复用和演进 |
| 抽象 | 越少越好，能内联就内联 | 值得投入，公开接口要稳 |
| 校验 | 只在系统入口做 | 每个公开 API 都做 |
| 命名 | 贴业务语义 | 贴能力，不带业务词 |

判错方向的代价是不对称的：业务代码里堆架构 → 长期维护地狱；工具库里图省事 → 全项目跟着改。

## 第 1 步：侦察（写第一行代码之前）

三个必查，缺一不可：

1. **已有实现** — 运行 `python scripts/scan_reuse.py <项目根> --terms "动词,名词"`（路径相对本 skill 目录），或直接 grep。重点看 `utils/ common/ shared/ helpers/ lib/ core/`。
2. **项目约定** — 读同目录 2–3 个已有文件：命名风格、分层方式、返回结构、错误处理、日志格式。新代码要长得像邻居，而不是像教程。
3. **既有约束** — 全局异常处理器、参数校验框架或中间件、ORM 的非空与唯一约束、配置加载方式、鉴权装饰器。这些直接决定第 2 步里哪些防御是重复劳动。

侦察结束，先输出一句结论再动手：

> 复用 `common/utils/date.ts` 的 `formatDate`；新增 `OrderCancelService.cancel`，放 `service/order/`，因为已有 service 都在这一层。

确实没找到就明说"未发现已有实现"，不要默默跳过。

## 第 2 步：四条铁律

### 1. 复用优先于新建

- 吻合度 ≥70% → 扩展现有实现（加可选参数、加重载），不新建。
- 30–70% → 新建，但放**公共层**，命名不带业务词。
- 业务模块内只允许存在"仅服务这一个业务"的私有函数。

反例：`OrderDateUtils.formatOrderDate()` —— 通用能力被业务名绑死，第二个模块要用时只能复制一份。

### 2. 只在信任边界校验

校验一次，之后信任契约。信任边界 = 外部输入进入系统的那一圈：HTTP 入参、MQ 消息、第三方回调、文件上传、配置加载。数据库若已声明非空，读出来就不必再判。

边界内部不重复判空。上游已校验的参数、ORM 已保证非空的字段、刚 `new` 出来的对象 —— 再判 null 是噪音，会让真正的空值问题失去信号。

配置缺失直接抛错，不给默认值（除非默认值本身是业务语义，如 `pageSize=20`）。

### 3. 抽象等第三次

- 第 1 次：直接写。
- 第 2 次：允许复制，留一行 `// 与 xxx 重复，第三次出现时提取`。
- 第 3 次：提取。此时才知道真正的变化点在哪。

不许因为"以后可能要加"而抽接口、策略、工厂。判断标准只有一个：**现在有没有第二个真实实现，或第二种真实变化？** 没有就用具体类 + 参数。

### 4. 主线优先

读代码的人应该能从上往下读完主流程，不用来回跳转十次。

- 一个函数只做一个抽象层级的事，不要既发 HTTP、又拼字符串、又写审计日志。
- 主流程缩进超过 3 层 → 抽"步骤函数"，函数名说清做什么（`applyDiscount`），而不是怎么做（`loopAndMultiply`）。
- 用早返回代替嵌套。
- 变量名承载信息：`overdueOrders` 优于 `list2`、`data`、`res`、`temp`。
- 注释写 why（这个判断为什么存在、这里为什么跳过），不写 what。
- 删掉某个辅助方法后主流程仍然读得懂 → 它本来就该内联。

**这四条铁律是默认值，不是教条。** 用户明确要求分层/接口/防御（对外 SDK、公共库），或团队规范如此，照做并把理由写清楚。要避免的是"没有理由的默认动作"，不是"有理由的例外"。

## 第 3 步：自检（交付前）

运行 `python scripts/audit_code.py <改动文件>`，再回答五个问题：

1. 我新写的函数里，有没有哪个项目里可能已经有了？
2. 每个判空或默认值，能指出它防的是哪个**真实存在**的空值来源吗？
3. 每个新增的接口或抽象类，现在有几个实现？只有一个，为什么不用具体类？
4. 每个 catch，是不是在做本该由上层统一做的事？catch 之后错误还可见吗？
5. 把我新加的三个辅助方法删掉，主流程还读得懂吗？

答不上来的项，就是要改的项。

## 红线

出现以下情况直接判定不合格，不讨论：

- 业务模块里出现通用工具，且命名带业务词（`OrderXxxUtils`、`UserHelper`）
- `catch (e) { log(e) }` 之后继续往下执行 —— 静默降级
- `process.env.X || 'default'`、`os.getenv('X', 'default')` 用于必填配置
- 只有一个实现，却抽出 interface + Factory
- 方法超过 80 行，或缩进超过 4 层
- 新增 `Manager` / `Provider` / `Handler` 类，方法体只是转发给另一个类

## 详细规则

按当前卡点选一份读，不要一次全读：

| 你在纠结什么 | 读哪份 |
|---|---|
| 这个函数是不是已经有人写过、该放哪个目录 | `references/reuse.md` |
| 要不要判空、要不要 catch、配置怎么读、日志打在哪 | `references/boundaries.md` |
| 要不要抽接口、为什么这段代码读不懂 | `references/shape.md` |

## 与其他 skill 的关系

这是**横向约束**，不替代具体技术栈 skill。与 `fullstack-dev` 同时生效时，架构骨架听 `fullstack-dev`，代码粒度听本 skill。
