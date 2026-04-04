# Lipa 语言参考手册 v0.5

> **Lipa**：Language for Intelligent Pipeline Agents
>
> 主流框架是庞大而松散的组件库；Lipa 是精密的微内核语言。

Lipa 是一门专为 AI agent 工作流设计的语言。它建立在一个核心判断上： agent 工作流的本质，是数据在不确定性下的流动。
语言的全部设计都从这个判断推导出来。

---

## 0. 设计哲学

### 正交性

Lipa 的每个机制解决恰好一类问题，机制之间可以自由组合，没有重叠，没有例外。

| 机制 | 解决的问题 |
|------|------------|
| 管道 `->` | 数据如何从一步流向下一步 |
| `?` 值 | 计算结果不确定时怎么办 |
| `[ ]` Table | 多个值如何并排放置 |
| `{ }` Block | 逻辑如何被打包和传递 |
| `tool` | 外部能力如何被 LLM 识别和调用 |
| `memory` | 知识如何跨会话持久存在 |
| `agent` | 自主决策循环如何被封装 |

七个机制。每个都不可或缺，每个都不多余。

### 三条基本原则

**原则一：错误即坍缩，不是异常。**
任何操作失败，结果变为 `?`，附带溯源信息。`?` 静默传播，不打断管道。
程序员在关心的地方统一处理，而不是在每一步防御。

**原则二：管道即拓扑。**
串行、并行、条件分支，全部用管道和 Table 组合表达。
不引入 DAG 声明语法，不引入 `flow` 关键字——管道就是拓扑。

**原则三：声明式外壳，管道式内核。**
`tool`、`memory`、`agent` 是声明式的外壳，描述"是什么"。
执行体和编排层是管道式的内核，描述"怎么流动"。
两层之间没有阻抗失配。

---

## 1. 核心：管道

管道是 Lipa 的脊梁。所有计算都是数据从左到右流动，每步把值"推入"下一个操作的第一个参数位。

```Lipa
-- 基本形式
"量子计算" -> web_search() - 等价于 web_search("量子计算")
value -> operation(extra_args)

-- 链式管道
topic -> web_search()
      -> researcher()
      -> editor();
```

### 1.1 中间捕获 `:>`

在管道中途保存快照，不打断流动：

```Lipa
topic -> researcher() :> draft
      -> editor()     :> final;

-- draft 和 final 之后都可以引用
draft -> checkpoint("draft_ready");
final -> db.store("reports");
```

### 1.2 透明异步

Lipa 的所有操作统一用 `->` 表达，无论同步还是异步。
异步调度由运行时处理，不暴露给程序员。
超时和重试通过调用参数控制：

```Lipa
topic -> web_search(timeout: 5.0, retry: 2);
draft -> human.review(timeout: 7200.0);
```

超时或重试耗尽，结果变为 `?`，管道继续。

---

## 2. 不确定性：`?` 值

`?` 是 Lipa 最重要的设计。它不是 null，不是异常，是 **所有类型值域中都存在的"待定"状态**。

### 2.1 产生 `?` 的场景

| 场景 | `.__trace__.kind` |
|------|-------------------|
| 运算失败（除零、类型转换失败） | `"failure"` |
| 工具调用超时 / 重试耗尽 | `"failure"` |
| Agent 超出最大步数 | `"failure"` |
| LLM 输出不满足 `done` 约束 | `"uncertain"` |
| 人工审阅超时未响应 | `"waiting"` |
| Bus 订阅超时 | `"waiting"` |

### 2.2 `?` 静默传播

接受 `?` 的操作跳过执行，直接把 `?` 传给下一步：

```Lipa
topic -> web_search()   -- 假设超时，返回 ?
      -> researcher()   -- 收到 ?，跳过
      -> editor()       -- 收到 ?，跳过
      -> recover({ : "处理失败" });  -- 在这里统一兜底
```

不需要在每一步检查错误。

### 2.3 处理 `?`

**recover**：将 `?` 替换为默认值，正常值穿透：

```Lipa
result -> recover({ : [] });          -- ? → 空 Table
result -> recover({ : "默认内容" });  -- ? → 字符串
```

**is(?)**：检查是否为 `?`，返回 Bool，自身永不为 `?`：

```Lipa
result -> is(?);    -- → true 或 false
```

**on_pending**：管道探针，遇 `?` 触发副作用，值本身不变：

```Lipa
result -> on_pending({
    t: "失败: " -> add(t.__trace__.reason) -> log();
}) -> recover({ : [] });
```

### 2.4 `.__trace__`：失败溯源

每个 `?` 值自动携带只读的 `.__trace__` Table：

```Lipa
result.__trace__ = [
    kind:      "failure";           -- 失败类型
    operation: "web_search";        -- 产生 ? 的操作
    reason:    "timeout after 5s";  -- 人类可读的原因
    chain:     ["researcher()", "editor()"];  -- 被跳过的步骤
];

-- 正常值的 .__trace__ 是空 Table，零开销
```

### 2.5 三路分派处理 `?`

利用 `?` 的整数映射（true=0，false=1，?=2）精确处理三种情况：

```Lipa
result -> is(?) -> [
    { : result },                                      -- true：（is(?) 不在此分支）
    { : result },                                      -- false：非 ?，继续
    { r: r.__trace__.kind -> [
        "failure":   { : default_val },
        "uncertain": { : r -> retry_with_hint() },
        "waiting":   { : r -> escalate() }
    ][r.__trace__.kind] -> recover({ : default_val }) -> run() }
] -> run();
```

---

## 3. Table `[ ]`

Table 是 Lipa 唯一的集合类型，统一了列表和键值对。

```Lipa
-- 序列（整数索引，从 0 开始）
sources: [result_a, result_b, result_c];
sources[0];       -- → result_a
sources[99];      -- → ?（越界）

-- 命名（字符串键）
config: [topic: "量子计算", limit: 10, date: "2025"];
config["topic"];  -- → "量子计算"
config.topic;     -- → "量子计算"（点号是语法糖）
config.missing;   -- → ?（键不存在）

-- 混合
record: [result_a, result_b, topic: "量子计算"];
record[0];        -- → result_a
record.topic;     -- → "量子计算"
record[2];        -- → "量子计算"
```

Table 在 Lipa 中有两个核心用途：

**用途一：结构化数据**

```Lipa
[topic: topic, result: draft, date: today -> as(String)]
    -> research_kb.store();
```

**用途二：并发算子的输入**

```Lipa
[
    topic -> web_search(limit: 10),
    topic -> arxiv_search(max: 5),
    topic -> research_kb.retrieve(top: 3)
] -> all();
```

### 3.1 Table 操作

```Lipa
table -> .length();           -- 元素数量
table -> .has("key");         -- 键是否存在 → Bool
table -> .get("key");         -- 同 table["key"]
table -> .keys();             -- 所有键名
table -> .values();           -- 所有值
table -> .merge(other);       -- 合并，右侧优先
```

---

## 4. Block `{ }`

Block 是打包的逻辑单元，尚未执行。它是 Lipa 中传递行为的方式。

```Lipa
-- 有参数的 Block
{ x: x -> mul(2) }

-- 无参数的 Block（接收上游值，用 : 表示）
{ : "默认值" }

-- 多参数
{ acc, x: acc -> add(x) }
```

Block 通过 `run()` 激活，或传递给接受 Block 的操作：

```Lipa
-- 直接激活
{ : "hello" -> log() } -> run();

-- 传递给 loop、span、done 等
queries -> span({ q: q -> web_search() });
[] -> loop { h: h -> .append(obs) } until({ h: h -> .is_final() });
```

### 4.1 命名 Block（函数定义）

编排逻辑包装成命名 Block，供复用：

```Lipa
research: { topic:
    topic -> researcher()
          -> editor();
}

-- 调用
"量子计算" -> research();
"机器学习" -> research();
```

---

## 5. `tool` 块

`tool` 声明一个外部能力。它同时做两件事：
- 告诉 **LLM**："这个工具能做什么"（`desc`、`param`）
- 告诉**运行时**："怎么执行"（执行体）、"出错怎么办"（`timeout`、`retry`）

### 5.1 结构

```
tool <名称> {
    desc:       "<LLM 看到的描述>";
    param       <名>: <类型>  "<说明>";
    param       <名>: <类型> = <默认值>  "<说明>";
    poll:       <秒数>;       -- 有 poll → 定时感知，无需 LLM 调用
    idempotent: <Bool>;       -- 默认 true；false 表示有不可逆副作用
    timeout:    <秒数>;
    retry:      <次数>;

    <执行体：管道代码，参数在作用域内>;
}
```

### 5.2 三种用途，一个关键字

`tool` 通过字段组合表达三种不同的行为方向：

| 用途 | `desc` | `poll` | `idempotent` |
|------|--------|--------|--------------|
| 主动工具（LLM 按需调用） | ✓ | — | true |
| 被动感知（定时拉取环境） | — | ✓ | true |
| 有副作用执行（发送/写入） | ✓ | — | **false** |

不同的用途用**字段**区分，不用不同的关键字。这是正交性原则的体现：行为差异是程度问题，不是种类问题。

### 5.3 示例

```Lipa
-- 主动工具：LLM 按需调用
tool web_search {
    desc:    "搜索互联网，获取最新资讯";
    param    query: String  "搜索关键词";
    param    limit: Int = 5 "最多返回条数";
    timeout: 5.0;
    retry:   2;

    query -> http.get("https://search.api",
                      params: [q: query, n: limit])
          -> .json()
          -> .get("results")
          -> recover({ : [] });
}

-- 被动感知：定时拉取，LLM 不主动调用
tool news_feed {
    poll:    60.0;   -- 每 60 秒拉取一次
    timeout: 8.0;

    http.get("https://news.rss")
        -> .xml()
        -> .get("items")
        -> filter({ item: item.date -> gte(today -> sub(1)) });
}

-- 有副作用的执行：发送邮件，禁止自动重试
tool send_email {
    desc:       "发送电子邮件给指定收件人";
    param       to:      String "收件人地址";
    param       subject: String "邮件主题";
    param       body:    String "邮件正文";
    idempotent: false;
    timeout:    10.0;

    [to: to, subject: subject, body: body]
        -> smtp.send();
}
```

### 5.4 调用

```Lipa
"量子计算" -> web_search();
"量子计算" -> web_search(limit: 20);
"量子计算" -> web_search(limit: 20, timeout: 8.0);
-- 调用方式与普通管道函数完全相同
```

---

## 6. `memory` 块

`memory` 声明一个跨会话持久的知识单元。

与 `tool` 的根本区别：**运行时保证写入的内容在下次调用时仍然可读**，即使进程重启。`tool` 的执行体每次从头执行；`memory` 有持久状态。

### 6.1 结构

```
memory <名称> {
    backend:  <存储配置>;
    schema:   [<字段名>: <类型>, …];
    ttl:      <秒数>;       -- 可选，数据过期时间
    capacity: <条数>;       -- 可选，超出后淘汰最旧的
}
```

### 6.2 两种 backend

```Lipa
-- 语义记忆：向量存储，支持相似度检索
memory research_kb {
    backend:  vector(collection: "research");
    schema:   [topic: String, result: String, date: String];
    ttl:      2592000.0;   -- 30 天
    capacity: 10000;
}

-- 对话缓冲：顺序存储，保留最近 N 条
memory conversation {
    backend: buffer(max: 50);
    schema:  [role: String, content: String];
}
```

### 6.3 读写操作

```Lipa
-- 写入
[topic: "量子计算", result: draft, date: "2025"]
    -> research_kb.store();

-- 语义检索（vector backend）
"量子计算最新进展" -> research_kb.retrieve(top: 5);

-- 精确读取
research_kb.get(key: "量子计算");

-- 追加（buffer backend）
[role: "user", content: message] -> conversation.append();

-- 读取最近 N 条
conversation.recent(n: 20);
```

### 6.4 `memory` 和 `tool` 的分工

判断原则：**写入者是谁。**

```Lipa
-- tool：外部系统写入，agent 只读取
tool crm_lookup {
    desc:  "查询客户信息";
    param  id: String "客户 ID";
    crm.get(customer_id: id);
}

-- memory：agent 自己写入，自己之后读取
memory research_kb {
    backend: vector(collection: "research");
    schema:  [topic: String, result: String];
}
```

---

## 7. `agent` 块

`agent` 声明一个能自主循环工作的决策主体。它持有 LLM、工具授权、记忆引用，以及终止条件。

### 7.1 结构

```
agent <名称> {
    llm:    <模型配置>;
    tools:  [<tool名>, …];    -- 授权可调用的工具
    memory: [<memory名>, …];  -- 可访问的记忆单元
    role:   "<系统提示 / 角色描述>";
    steps:  <最大步数>;       -- 默认 20，超出后结果为 ?
    done:   { <变量>: <条件> };  -- 终止条件（可选）
}
```

### 7.2 `done`：终止条件

每次 act-observe 之后，运行时对 `done` Block 求值。满足则提前退出，返回当前输出。

```Lipa
done: { out: out -> .has("conclusion") };
```

不写 `done`：跑满 `steps` 步后停止，返回最后输出。

**为什么 `done` 是显式 Block，而不是让 LLM 自己判断？**
LLM 说"完成了"不可靠——它可能过早停止，也可能无效循环。
把终止条件写成可客观求值的 Block，让运行时判断，消除歧义。
这和 `?` 的设计同源：不信任隐式语义，只信任可求值的表达式。

### 7.3 `llm` 配置

```Lipa
llm: claude(temperature: 0.3);
llm: claude(model: "claude-opus", temperature: 0.1);
llm: openai(model: "gpt-4o", temperature: 0.5);
```

### 7.4 调用

```Lipa
"写量子计算综述" -> researcher();
"写量子计算综述" -> researcher(context: sources);
"写量子计算综述" -> researcher(steps: 10, timeout: 120.0);
-- 返回值是 agent 最后输出的值
-- 超出 steps 或出错 → ?
```

### 7.5 示例

```Lipa
agent researcher {
    llm:    claude(temperature: 0.3);
    tools:  [web_search, arxiv_search];
    memory: [research_kb, conversation];
    role:   "综合多方资料，给出有据可查的研究结论，输出包含 conclusion 字段";
    steps:  15;
    done:   { out: out -> .has("conclusion") };
}

agent editor {
    llm:    claude(temperature: 0.7);
    tools:  [send_email];
    memory: [];
    role:   "将研究结论改写为流畅的中文报告，保持事实准确";
    steps:  5;
}
```

---

## 8. 并发

### 8.1 `all()`：等全部完成

```Lipa
[
    topic -> web_search(limit: 10),
    topic -> arxiv_search(max: 5),
    topic -> research_kb.retrieve(top: 3)
] -> all();
-- → [web结果, arxiv结果, kb结果]
-- 某项失败 → 对应位置为 ?，不影响其他项
```

### 8.2 `race()`：取最快的

```Lipa
[
    topic -> search_engine_a(),
    topic -> search_engine_b()
] -> race();
-- → 最先返回的结果，其余取消
```

### 8.3 `any(n)`：等 n 个完成

```Lipa
[source_a, source_b, source_c, source_d] -> any(2);
-- → 最先完成的 2 个结果
```

### 8.4 `span()`：并发 map

对列表中每个元素并发执行相同操作：

```Lipa
queries -> span({ q: q -> web_search() });
-- → [结果1, 结果2, …]，顺序与输入对应
-- 某项失败 → 对应位置为 ?
```

`span` 与 `all` 的选择原则：
- 同一操作应用于列表 → `span`
- 不同操作并发执行 → `all`

---

## 9. `loop` 块

`loop` 是 Lipa 中迭代的唯一形式，有三种形态。
它在语义上等价于 TCO 递归，保证不会栈溢出。

### 9.1 固定次数

```Lipa
0 -> loop(3) { acc, i:
    "第 " -> add(i) -> add(" 次") -> log();
    acc -> add(1);
};
-- acc 从 0 开始，i 从 0 到 2
-- 每次把 body 的返回值作为下次的 acc
-- → 3
```

### 9.2 条件终止

```Lipa
[] -> loop { history:
    history -> llm.think() :> thought
            -> dispatch_tool()  :> obs;
    history -> .append([thought: thought, obs: obs]);
} until({ h: h -> .last() -> .get("obs") -> .is_final() });
```

### 9.3 条件终止 + 步数上限（防死循环）

```Lipa
[] -> loop(max: 20) { history:
    history -> researcher.think() :> thought
            -> web_search(query: thought.query) :> obs;
    history -> .append([thought: thought, obs: obs]);
} until({ h: h -> .last() -> .has("final_answer") });
-- 超出 max 步 → ? (trace: "loop exceeded max steps")
```

---

## 10. 人工介入

```Lipa
-- 审阅：不改值，添加注释，超时 → ?
draft -> human.review(
    prompt:  "请审阅草稿，标注需要修改的地方",
    timeout: 7200.0
) -> recover({ : draft });         -- 超时则用原稿继续

-- 补充输入：返回用户输入的值，超时 → ?
human.input(
    prompt:  "搜索结果不足，请补充参考资料",
    timeout: 3600.0
) -> recover({ : [] });

-- 审批：返回 Bool，超时 → ?
plan -> human.approve(
    prompt:  "是否执行以下计划？",
    timeout: 86400.0
) -> recover({ : false });         -- 超时则拒绝
```

人工介入超时统一产生 `?`，`recover` 指定跳过时的默认行为，管道正常继续。
这和工具超时的处理方式完全一致——不引入新的错误处理机制。

---

## 11. 持久化：`checkpoint`

`checkpoint` 将当前管道状态旁路保存。如果后续步骤失败，从最近的 checkpoint 恢复，无需重新执行已完成的步骤。

```Lipa
topic -> researcher()
      -> checkpoint("draft_ready")    -- 旁路保存，透传值
      -> editor()
      -> checkpoint("final_ready");

-- 从检查点恢复
topic -> resume_from("draft_ready")  -- 跳过 researcher，直接从这里继续
      -> editor()
      -> checkpoint("final_ready");

-- 配置存储后端（程序入口配置一次）
checkpoint.backend = redis(host: "localhost", ttl: 86400.0);
```

`checkpoint` 是 Tap 模式：值流过时做旁路保存，不改变值本身，管道正常继续。
这和 `on_pending` 是同一种模式，只是触发条件不同。

---

## 12. 多 Agent 协作

多 agent 需要异步通信时，用内置的 `Bus` 对象。
`Bus` 不是新保留字，是运行时提供的内置对象。

```Lipa
pipeline: { topic:
    bus: Bus("task_" -> add(topic -> hash()));

    [
        -- agent A：执行研究，发布结果
        topic -> researcher()
              -> bus.publish("research_done"),

        -- agent B：等待研究结果，执行编辑
        bus.subscribe("research_done", timeout: 300.0)
            -> editor()
            -> bus.publish("edit_done")
    ] -> all();

    -- 收尾
    bus.subscribe("edit_done", timeout: 60.0)
        -> db.store("reports");
}
```

`bus.subscribe` 超时返回 `?`，处理方式与其他超时场景完全一致。

---

## 13. 完整端到端示例

```Lipa
-- ════════════════════════════════════════════
--  声明层
-- ════════════════════════════════════════════

tool web_search {
    desc:    "搜索互联网，获取最新资讯";
    param    query: String  "搜索关键词";
    param    limit: Int = 5 "最多返回条数";
    timeout: 5.0;
    retry:   2;

    query -> http.get("https://search.api",
                      params: [q: query, n: limit])
          -> .json()
          -> .get("results")
          -> recover({ : [] });
}

tool arxiv_search {
    desc:    "搜索学术论文库";
    param    query: String  "论文主题";
    param    max:   Int = 3 "最多返回篇数";
    timeout: 8.0;

    query -> arxiv.search(max_results: max);
}

tool send_report {
    desc:       "将最终报告发送给指定邮箱";
    param       to:      String "收件人";
    param       content: String "报告正文";
    idempotent: false;
    timeout:    10.0;

    [to: to, body: content] -> smtp.send();
}

memory research_kb {
    backend:  vector(collection: "research");
    schema:   [topic: String, result: String, date: String];
    ttl:      2592000.0;
    capacity: 10000;
}

memory conversation {
    backend: buffer(max: 50);
    schema:  [role: String, content: String];
}

agent researcher {
    llm:    claude(temperature: 0.3);
    tools:  [web_search, arxiv_search];
    memory: [research_kb, conversation];
    role:   "综合多方资料，给出有据可查的研究结论，输出包含 conclusion 字段";
    steps:  15;
    done:   { out: out -> .has("conclusion") };
}

agent editor {
    llm:    claude(temperature: 0.7);
    tools:  [send_report];
    memory: [];
    role:   "将研究结论改写为流畅的中文报告，保持事实准确";
    steps:  5;
}

-- ════════════════════════════════════════════
--  编排层
-- ════════════════════════════════════════════

research_pipeline: { topic, recipient:

    -- Step 1：并发多源搜索
    [
        topic -> web_search(limit: 10),
        topic -> arxiv_search(max: 5),
        topic -> research_kb.retrieve(top: 3)
    ] -> all()
      -> filter({ s: s -> is(?) -> not() }) :> sources;

    -- Step 2：来源不足时请求人工补充
    sources -> .length() -> lt(2) -> [
        { : human.input(
              prompt:  "搜索结果不足，请补充参考资料",
              timeout: 3600.0
          ) -> recover({ : sources }) },
        { : sources },
        { : sources }
    ] -> run() :> final_sources;

    -- Step 3：研究 agent 生成结论
    "综合以下资料，写关于 " -> add(topic) -> add(" 的研究结论")
        -> researcher(context: final_sources)
        -> checkpoint("draft_ready")
        -> on_pending({ t:
            "研究失败: " -> add(t.__trace__.reason) -> log();
        }) :> draft;

    -- Step 4：写入长期记忆
    [topic: topic, result: draft, date: today -> as(String)]
        -> research_kb.store();

    -- Step 5：人工审阅
    draft -> human.review(
        prompt:  "请审阅草稿，标注需要修改的地方",
        timeout: 7200.0
    ) -> recover({ : draft }) :> reviewed;

    -- Step 6：编辑润色
    reviewed -> editor()
             -> checkpoint("final_ready")
             -> on_pending({ t:
                 "编辑失败: " -> add(t.__trace__.reason) -> log();
             }) :> final;

    -- Step 7：结果分派
    final -> is(?) -> [
        { : [status: "failed", trace: final.__trace__]
              -> db.store("failures") },
        { f:
            f -> send_report(to: recipient,
                             content: f -> .get("report"));
            [status: "done", result: f] -> db.store("reports");
        },
        { : ? }
    ] -> run();
}
```

---

## 14. 语法速查

```
── 基础类型 ───────────────────────────────────────────────────
String   Int   Float   Bool   ?
[…]      Table（序列 / 命名 / 混合）
{ … }    Block（打包的逻辑，未执行）

── 管道 ───────────────────────────────────────────────────────
a -> f(b)           a 填入 f 的第一参数
… :> name           中途捕获为变量
f(timeout: N)       超时 N 秒，失败 → ?
f(retry: N)         失败后重试 N 次

── Table 操作 ─────────────────────────────────────────────────
t[0]                整数索引
t["key"] / t.key    字符串索引
t -> .length()
t -> .has("key")
t -> .get("key")
t -> .keys() / .values()
t -> .merge(other)

── ? 处理 ─────────────────────────────────────────────────────
val -> is(?)                        检查是否为 ?
val -> recover({ : default })       ? → 默认值
val -> on_pending({ t: … })         探针，不改变值
val.__trace__                       溯源 Table
  .kind      "failure" | "uncertain" | "waiting"
  .operation  产生 ? 的操作名
  .reason     人类可读原因
  .chain      被跳过的操作序列

── 分支 ───────────────────────────────────────────────────────
cond -> [b_false, b_true, b_pending] -> run()
table[key] -> recover({ : default }) -> run()

── 并发 ───────────────────────────────────────────────────────
[a, b, c] -> all()        等全部完成，失败项为 ?
[a, b, c] -> race()       取最快的，其余取消
[a, b, c] -> any(n)       等 n 个完成
list -> span({ x: … })   并发 map，失败项为 ?
list -> each({ x: … })    串行 map

── loop ───────────────────────────────────────────────────────
init -> loop(n) { acc, i: … }
init -> loop { acc: … } until({ acc: cond })
init -> loop(max: n) { acc: … } until({ acc: cond })

── tool 声明 ──────────────────────────────────────────────────
tool name {
    desc:       "…";               -- 有 desc → LLM 可调用
    param       p: Type  "…";      -- 无歧义时，末尾的分号可以省略
    param       p: Type = default  "…"
    poll:       N;                 -- 有 poll → 定时感知
    idempotent: Bool;              -- false → 禁止自动重试
    timeout:    N;
    retry:      N;
    <执行体>;
}
name(args)                         -- 调用：同普通函数

── memory 声明 ────────────────────────────────────────────────
memory name {
    backend:  vector(collection: "…") | buffer(max: N);
    schema:   [field: Type, …];
    ttl:      N;
    capacity: N;
}
val  -> name.store()               写入
val  -> name.retrieve(top: N)      语义检索
name.get(key: "…")                 精确读取
val  -> name.append()              追加（buffer）
name.recent(n: N)                  最近 N 条（buffer）

── agent 声明 ─────────────────────────────────────────────────
agent name {
    llm:    provider(options);
    tools:  [tool_a, tool_b];
    memory: [mem_a, mem_b];
    role:   "…";
    steps:  N;
    done:   { out: cond };
}
task -> name()                     调用
task -> name(context: data)
task -> name(steps: N, timeout: T)

── 人工介入 ───────────────────────────────────────────────────
val  -> human.review(prompt, timeout)    审阅，超时 → ?
        human.input(prompt, timeout)     输入，超时 → ?
val  -> human.approve(prompt, timeout)   审批 → Bool，超时 → ?

── 持久化 ─────────────────────────────────────────────────────
val  -> checkpoint("name")         旁路保存，透传值
task -> resume_from("name")        从检查点恢复

── 多 Agent 通信 ──────────────────────────────────────────────
bus: Bus("channel_name")
val  -> bus.publish("topic")       发布（非阻塞）
bus.subscribe("topic", timeout: N) 订阅，超时 → ?
```

---

## 附：设计决策备忘

| 问题 | 决策 | 理由 |
|------|------|------|
| 保留字数量 | 3个：`tool` `memory` `agent` | 正交性原则：每个保留字解决一类不可替代的问题 |
| `sensor`/`actuator` | 合并入 `tool`，用字段区分 | 行为差异是程度问题，不是种类问题 |
| `flow` | 不引入 | 管道即拓扑，`flow` 是零价值包装 |
| 异步表达 | 透明异步，`->` 统一 | 运行时比程序员更懂调度；标注收益低于认知负担 |
| 终止条件 | `done` 字段（Block） | LLM 自述"完成"不可靠；Block 客观可求值 |
| 长期记忆 | 独立 `memory` 块 | 跨会话持久性需要运行时特殊支持，`tool` 无法提供 |
| 迭代 | `loop` 语法糖 | TCO 递归是正确语义，但直写不直观；`loop` 是友好外壳 |
| 并发 map | `span` | 高频模式；手展开 `all` 需要额外 Block 包装 |
| 错误处理 | `?` + `recover` + `on_pending` | 统一机制：超时、失败、人工未响应，全部产生 `?` |
| 分号 | 声明字段行末加 `;` | 区分"声明字段"与"执行体代码"，视觉上更有代码感 |
| class/mixin/Tensor/模块 | 不引入 | Lipa 是编排层语言；这些属于通用计算层，不在关注点内 |
