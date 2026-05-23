# CareerVerse: Agent Archives 架构设计文档

> 日期: 2026-05-23
> 状态: 审查通过，待实施
> 范围: agent_archives.json + interview_agents fallback + ontology 优化

---

## 1. 总体结论

### agent_archives.json 应该做吗？

应该做。但不是"保存记忆"系统，而是"整理已有数据，生成报告阶段可用的 Agent 档案摘要"。当前模拟结束后，原始数据（actions.jsonl、SQLite、profiles、Zep 图谱）全部保留在磁盘上，但格式分散、结构不统一，报告 Agent 很难直接用。agent_archives.json 的价值是：把分散数据提炼成一份结构化、带证据溯源的档案，让第三步的 archived interview 可以直接消费。

### 它是否符合当前项目规划？

完全符合。CareerVerse 的核心交付物是"平行人生报告"。报告的质量取决于报告 Agent 能拿到多少高质量素材。agent_archives.json 就是素材的中间层。

### 它是否是 interview_agents 超时问题的长期解决方案之一？

是的。超时问题的根因是 live IPC 依赖模拟进程存活。agent_archives.json 提供了 offline fallback，让报告生成不再依赖进程是否还活着。

### 当前人物型 Agent 不足是否会影响产品体验？

会。调查数据证实：sim_a2aa9c27508e 跑了 10 轮，5 个 agent 总共只产生 3 条帖子。其中 Company 类型（乐歌人体工学）只有 sign_up，没有任何实际行为。ProductManager 有 1 条帖子和 1 次采访。空转严重。

根因有两个：
- Agent 活跃度配置偏低（Company 的 activity_level 只有 0.25，active_hours 是工作时间）
- 更重要的是：5 个 agent 中只有 2 个是人物型（Professor、Person），1 个是 ProductManager（半人物），1 个是 Company（纯组织），还有一个 ProductManager 只有 sign_up+interview

### ontology_generator 是否需要向"职业关系网络生成器"升级？

需要，但不是重写，是最小改造。当前 ontology_generator 生成的是"知识图谱实体类型"，不直接控制生成多少人物、多少组织。真正决定 agent 组成的环节是：ontology_generator 定义类型 -> LLM 生成具体实体 -> 这些实体全部变成 agent。所以改造点应该在两处：提示词引导生成更多人物型实体，以及 profile 生成阶段对人物型 agent 给更高活跃度。

---

## 2. agent_archives.json 推荐 Schema

```json
{
  "metadata": {
    "simulation_id": "sim_xxx",
    "generated_at": "ISO时间戳",
    "data_sources": ["actions.jsonl", "twitter_simulation.db", "..."],
    "generation_config": {
      "used_zep": true,
      "used_llm_summary": false,
      "fallback_mode": "none / sqlite_only / profile_only"
    }
  },

  "agents": [
    {
      "agent_id": 0,
      "agent_name": "浙大硕导",
      "role": "教授",
      "agent_type": "Professor",
      "category": "person",
      "organization_or_person": "person",

      "profile": {
        "bio": "从 profile 文件提取的简介",
        "age": 45,
        "gender": "male",
        "mbti": "INTJ",
        "profession": "大学教授",
        "career_stage": "senior",
        "interested_topics": ["AI", "教育"],
        "source": "twitter_profiles.csv",
        "source_file": "twitter_profiles.csv",
        "source_line": 2
      },

      "key_experiences": [
        {
          "summary": "在第3轮发布了一条关于追觅vs安克的帖子",
          "round": 3,
          "platform": "twitter",
          "source": "actions.jsonl",
          "record_id": "twitter_actions_line_5",
          "timestamp": "ISO时间",
          "evidence_snippet": "发帖原文前100字...",
          "confidence": "high"
        }
      ],

      "important_actions": [
        {
          "action_type": "CREATE_POST",
          "summary": "发布了关于职业选择的分析帖",
          "round": 0,
          "platform": "twitter",
          "source": "actions.jsonl",
          "record_id": "twitter_actions_line_3",
          "timestamp": "ISO时间",
          "evidence_snippet": "帖子内容摘要...",
          "confidence": "high"
        }
      ],

      "posts": [
        {
          "post_id": 1,
          "content": "完整帖子内容",
          "round": 0,
          "platform": "twitter",
          "source": "twitter_simulation.db/post表",
          "record_id": "post_id=1",
          "timestamp": "ISO时间",
          "num_likes": 0,
          "num_comments": 0,
          "confidence": "high"
        }
      ],

      "comments": [],

      "relationships": [
        {
          "target_agent_id": 1,
          "target_agent_name": "产品经理",
          "relationship_type": "interacted_with",
          "summary": "点赞了产品经理的帖子",
          "source": "twitter_simulation.db/like表",
          "record_id": "like_id=xxx",
          "confidence": "high"
        }
      ],

      "opinions": [
        {
          "topic": "追觅vs安克的职业选择",
          "stance": "分析性",
          "summary": "从学术角度分析了两个选择",
          "source": "post_id=1",
          "evidence_snippet": "帖子原文片段...",
          "confidence": "medium"
        }
      ],

      "evaluations": [],

      "final_state": {
        "total_posts": 1,
        "total_comments": 0,
        "total_likes_given": 0,
        "total_likes_received": 0,
        "total_follows": 0,
        "active_rounds": [0],
        "inactive_rounds": [1,2,3,4,5,6,7,8,9],
        "participation_rate": 0.1
      },

      "data_sufficiency": "low",

      "limitations": [
        "该 Agent 在10轮模拟中仅在第0轮有1条行为记录",
        "没有评论、点赞、关注等社交互动",
        "无法形成可靠的第一人称评价",
        "archived interview 只能基于 profile 和少量行为记录回答"
      ],

      "evidence_index": {
        "total_evidence_count": 2,
        "high_confidence_count": 2,
        "medium_confidence_count": 0,
        "low_confidence_count": 0,
        "sources_used": ["twitter_profiles.csv", "actions.jsonl", "twitter_simulation.db"],
        "sources_unavailable": []
      }
    }
  ]
}
```

### 关键字段说明

- **category**: "person" 或 "organization"，由 agent_type 对照 INDIVIDUAL_ENTITY_TYPES / GROUP_ENTITY_TYPES 判断
- **confidence**: high（有原始记录为证）、medium（从行为推断）、low（从 profile 推断）
- **data_sufficiency**: high（>=5条有效行为+有互动）、medium（2-4条有效行为）、low（<2条有效行为或无互动）
- **limitations**: 必须明确列出信息不足的方面，archived interview 必须遵守这个约束
- **evidence_index**: 统计层面的证据质量概览，方便报告 Agent 快速判断哪些 agent 值得深入采访

---

## 3. build_agent_archives 设计

### 输入

```
build_agent_archives(simulation_dir: str, force: bool = False) -> dict
```

- simulation_dir: 模拟目录的绝对路径，例如 /path/to/sim_xxx/
- force: 是否强制重新生成（即使 agent_archives.json 已存在）

### 输出

- 返回完整的 agent_archives.json dict
- 同时写入 simulation_dir/agent_archives.json

### 数据源读取优先级（从高到低）

```
1. simulation_config.json    -> agent_configs（agent_id, name, type, activity_level...）
2. twitter_profiles.csv      -> profile 详情
3. reddit_profiles.json      -> profile 详情
4. twitter/actions.jsonl     -> 行为记录
5. reddit/actions.jsonl      -> 行为记录
6. twitter_simulation.db     -> 帖子、评论、点赞、关注等结构化数据
7. reddit_simulation.db      -> 同上
8. run_state.json            -> 模拟运行状态（总轮数、完成情况）
9. Zep 图谱（可选）          -> 关系、事实补充
```

### 降级策略

```
if simulation_config.json 不存在:
    -> 无法生成，返回空档案 + 错误信息

if profiles 文件不存在:
    -> 从 simulation_config.json 的 entity_name/entity_type 生成最小 profile
    -> limitations 加 "缺少 profile 文件，基本信息不完整"

if actions.jsonl 不存在或为空:
    -> key_experiences / important_actions / posts 全部为空
    -> data_sufficiency 直接标 low
    -> limitations 加 "无行为记录"

if SQLite DB 不存在:
    -> relationships / comments 无法提取
    -> 从 actions.jsonl 的 FOLLOW/LIKE 等动作推断关系（降级）

if Zep 不可用:
    -> 跳过 Zep 关系补充，完全依赖本地数据
    -> generation_config.used_zep = false
    -> 不影响核心功能，Zep 只是补充
```

### 调用时机

**时机1: 模拟结束后自动调用**
- 位置：simulation_runner.py 的 _monitor_simulation() 中，当 run_state.json 变为 completed 时
- 条件：如果 simulation_dir/agent_archives.json 不存在，自动触发
- 不阻塞 IPC 等待循环，在监控进程中异步完成

**时机2: 报告生成前补生成**
- 位置：report_agent.py 的初始化阶段或 zep_tools.py 的 interview_agents 方法中
- 条件：需要档案但 agent_archives.json 不存在时，实时生成
- 这是兜底，正常情况下时机1已经生成了

### 是否需要 LLM

不需要。build_agent_archives 是纯数据整理函数：
- 读文件 -> 提取字段 -> 分类汇总 -> 计算 data_sufficiency -> 写 JSON
- 不调用 LLM 做摘要，避免增加延迟和成本
- opinions 字段从帖子内容提取关键词，不做深度分析

### 模块位置

新建文件：
```
backend/app/services/agent_archiver.py
```

包含：
- build_agent_archives() 主函数
- _read_profiles() 读 profile 文件
- _read_actions() 读 actions.jsonl
- _read_sqlite() 读 SQLite 数据库
- _calculate_sufficiency() 计算 data_sufficiency
- _build_evidence_index() 构建证据索引

约 300-400 行代码。

---

## 4. interview_agents fallback 设计

### 模式检测流程

```
interview_agents 被调用
    |
    +-- Step 0: 预检查（已完成，第一步修复）
    |   +-- simulation_dir 不存在 -> 返回错误
    |   +-- env_status.json 不存在 -> 跳到 fallback
    |   +-- env_status 不是 alive -> 跳到 fallback
    |
    +-- Step 1A: live IPC 尝试
    |   +-- 发 IPC 命令，超时 60 秒（已从 180 秒降低）
    |   +-- 成功 -> 返回结果，标记 mode: "live"
    |   +-- 失败/超时 -> 跳到 fallback
    |
    +-- Step 1B: archived interview fallback
        +-- 读取 agent_archives.json
        |   +-- 不存在 -> 先调用 build_agent_archives() 补生成
        +-- 找到目标 agent 的档案
        +-- 检查 data_sufficiency
        |   +-- "high" -> LLM 用完整档案生成第一人称回答
        |   +-- "medium" -> LLM 回答但标注"信息有限"
        |   +-- "low" -> 返回"该角色在模拟中行为记录有限，无法给出可靠回答"
        +-- 返回结果，标记 mode: "archived"
        +-- 日志标记 ARCHIVED_INTERVIEW
```

### archived interview 的实现方式

不直接从档案拼回答，而是构造一个 LLM 调用，把档案作为上下文注入：

```
system: 你是{agent_name}，一个{role}。
你刚刚经历了一场职业平行宇宙模拟。
请根据你的经历和记忆，回答以下问题。

重要规则：
- 只能根据下面提供的【档案记录】回答
- 如果档案中没有相关信息，明确说"我在模拟中没有经历过这个"
- 不要编造档案中没有的事实
- 可以表达态度和观点，但必须有档案依据

【档案记录】
{agent_archive 的 key_experiences + posts + opinions 的 JSON}

【数据充分度】{data_sufficiency}
【局限性】{limitations}

user: {interview_question}
```

### data_sufficiency 对回答的控制

```
data_sufficiency = "high":
  -> 正常生成第一人称回答
  -> 回答可以包含观点、评价、建议
  -> 标注 source_count: N

data_sufficiency = "medium":
  -> 生成回答，但开头声明"我在这段时间的记录不算完整"
  -> 只对有证据的话题给出明确回答
  -> 无证据的话题说"我记得不太清楚"

data_sufficiency = "low":
  -> 不生成第一人称回答
  -> 返回结构化结果：
    {
      "mode": "archived",
      "status": "insufficient_data",
      "agent_name": "xxx",
      "available_info": "仅有 profile 信息和 X 条行为记录",
      "limitations": [...],
      "summary": "基于有限档案的简短概述（从 profile 推断，不冒充第一人称）"
    }
```

### 日志区分

```
[INTERVIEW] mode=live agent=浙大硕导 question="..." duration=2.3s
[INTERVIEW] mode=archived agent=产品经理 question="..." data_sufficiency=high sources=5
[INTERVIEW] mode=archived agent=乐歌公司 question="..." data_sufficiency=low status=insufficient_data
```

### 保证不再超时

三重保护：
1. Step 0 预检查（0 秒跳过，已实现）
2. IPC 超时从 180 秒降到 60 秒（已实现）
3. archived fallback 无 IPC 调用，纯文件读取 + LLM 调用，总耗时 < 30 秒

最坏情况：60 秒 IPC 超时 + 30 秒 archived fallback = 90 秒。
之前最坏情况：180 秒 x 3 次 = 540 秒。

---

## 5. 人物型 Agent 与职业平行宇宙适配性审查

### 当前组织/公司 Agent 过多的问题是否真实存在？

真实存在。以 sim_a2aa9c27508e 为例：

```
agent_id=0  浙大硕导                              Professor       person
agent_id=1  产品经理                               ProductManager  person
agent_id=2  高级产品经理                            ProductManager  person
agent_id=3  乐歌人体工学科技股份有限公司              Company         organization
agent_id=4  蒋子俊                                 Person          person
```

5 个 agent 中 4 个是"人"，1 个是"公司"。看起来比例还行，但实际行为：

```
浙大硕导:        1条帖子, 0评论, 0点赞, 0关注   -> 活跃度极低
产品经理:        1条帖子, 0评论, 0点赞, 0关注   -> 有采访记录
高级产品经理:     0条帖子, 0评论, 0点赞, 0关注   -> 仅 sign_up + interview
乐歌公司:        0条帖子, 0评论, 0点赞, 0关注   -> 仅 sign_up
蒋子俊:          1条帖子, 0评论, 0点赞, 0关注   -> 活跃度极低
```

问题不只是"组织太多"，而是"所有 agent 都不够活跃"。

### 这会导致模拟内容单薄吗？

会。10 轮模拟只产生 3 条帖子、0 条评论、0 条点赞、0 条关注。这意味着：
- Zep 图谱几乎没有数据可以提取
- agent_archives 几乎是空的
- 报告生成时 insight_forge / panorama_search 搜不到有意义的实体关系
- interview_agents 即使能工作，agent 也"无话可说"

### 为什么职业平行宇宙需要更多人物型 Agent？

产品定位是"如果选了另一条路会怎样"。用户想知道的不是"字节跳动的公司简介"，而是：
- 面试官会怎么评价我
- 同事会怎么和我相处
- 上司会怎么指导我
- 竞争对手会怎么和我竞争
- 猎头会怎么说服我跳槽

这些都是"人的故事"，不是"组织的描述"。

### 公司/组织 Agent 应该扮演什么角色？

世界实体、机会来源、规则系统。具体来说：
- 定义岗位要求和薪资范围（规则）
- 定义文化氛围和晋升机制（环境）
- 定义行业趋势和市场机会（背景）
- 不需要高频发帖，但应该在 profile 中包含丰富的"世界规则"信息
- 其他人物 Agent 的行为应该受这些规则影响

### 人物型 Agent 应该扮演什么角色？

互动主体、叙事载体、采访对象。具体来说：
- HR Agent: 发布招聘信息、筛选简历、面试候选人
- 面试官 Agent: 提技术问题、评价回答、给出录用建议
- 同事 Agent: 日常交流、分享经验、职场八卦
- 竞争者 Agent: 竞争同一岗位、展示不同选择
- 导师 Agent: 给职业建议、分享行业洞察
- 猎头 Agent: 推荐机会、分析利弊

---

## 6. ontology_generator 最小改造方案

### 当前问题

ontology_generator 只生成"实体类型"（如 Professor、Company），不生成"具体人物"。具体实体是由 LLM 在后续阶段生成的，但提示词里没有引导 LLM 生成"围绕公司/岗位的人物网络"。

### 改造思路：不改 ontology_generator 的类型系统，改实体生成阶段的提示词

当前流程：
```
ontology_generator -> 10个实体类型 -> LLM生成具体实体 -> 全部变成agent
```

改造后流程：
```
ontology_generator -> 10个实体类型（不变）
  -> LLM生成具体实体时，提示词引导：
     - 2-3个组织/公司实体（世界背景）
     - 5-7个人物实体（围绕这些公司的职业关系网络）
```

### 推荐新增/强化的人物 Agent 类型

不需要新增 entity_type。当前已有的 Person、Colleague、Mentor、Competitor、Leader、Entrepreneur 就够了。问题在于 LLM 生成具体实体时的比例控制。

改造点在两个地方：

#### 改造点 A：ontology_generator 的提示词追加指引

在 _build_user_message() 的追加规则中加入：

```
7. 实体设计必须服务于"职业平行宇宙"的叙事需求：
   - 组织型实体不超过3个（提供世界背景和机会来源）
   - 人物型实体至少5个（提供互动主体和叙事视角）
   - 人物必须围绕用户的职业目标构成关系网络
   - 每个人物应代表一种职业选择的可能性或视角
```

#### 改造点 B：oasis_profile_generator 生成 profile 后，对人物型 agent 提高活跃度

在 simulation_config_generator.py 的 agent_config 生成中：
- 人物型 agent：activity_level 0.6-0.9，active_hours 覆盖晚间
- 组织型 agent：activity_level 0.1-0.3，active_hours 仅工作时间，posts_per_hour 极低

当前代码已有这个逻辑（官方机构 vs 个人的区分），但需要确保对"组织"类型都走低活跃度分支。

### 每类 Agent 的行为策略

```
人物型 (person/colleague/mentor/competitor/leader/entrepreneur):
  activity_level: 0.6-0.9
  active_hours: [9,10,11,12,13,14,15,16,17,18,19,20,21,22]
  posts_per_hour: 0.3-1.0
  comments_per_hour: 0.5-2.0
  动作偏好: CREATE_POST, CREATE_COMMENT, LIKE_POST, FOLLOW

组织型 (company/organization/institution):
  activity_level: 0.1-0.3
  active_hours: [9,10,11,14,15,16]
  posts_per_hour: 0.01-0.05
  comments_per_hour: 0.0-0.02
  动作偏好: CREATE_POST（招聘/公告类）, DO_NOTHING（大部分时间）
```

### 如何控制 Agent 数量

保持 5 个 agent 不变（OASIS 引擎限制）。但调整比例：
- 1-2 个组织型（公司/行业）
- 3-4 个人物型（围绕核心公司的关键人物）

如果未来需要更多 agent，可以在 simulation_config 中增加 agent_configs 数组长度。但 MVP 阶段 5 个够用。

### 如何保证生成结果服务于用户职业路径

ontology_generator 的提示词已经包含用户的 simulation_requirement（职业经历和分叉选择）。只需要在提示词中加强引导：

```
你设计的人物实体应该围绕用户的核心职业问题展开：
- 用户的分叉选择涉及哪些公司和岗位？
- 这些公司里会有哪些关键人物影响用户的职业发展？
- 谁会是支持者、谁会是质疑者、谁会是竞争对手？
```

### 是否需要区分 agent_type（organization/person/event/mechanism）？

当前架构已经通过 INDIVIDUAL_ENTITY_TYPES 和 GROUP_ENTITY_TYPES 做了区分，这已经够用。不建议新增 event_agent 和 mechanism_agent 类型，这会增加复杂度但没有增加 MVP 价值。组织和人两种就够了。

---

## 7. 最小改动文件清单

```
新增文件：
──────────────────────────────────────────────────────
backend/app/services/agent_archiver.py          [约 350 行]
  职责：读取模拟数据，生成 agent_archives.json
  包含：build_agent_archives() 及辅助函数
  不依赖 LLM，纯数据整理

需要修改的文件：
──────────────────────────────────────────────────────
backend/app/services/zep_tools.py               [约 80 行改动]
  职责：interview_agents 方法增加 archived fallback
  改动点：
  - Step 0 预检查（已完成）
  - IPC 失败后跳到 archived fallback
  - 读取 agent_archives.json
  - 调用 LLM 生成第一人称回答（基于档案）
  - 返回结果标记 mode: "archived"
  - 日志区分 live/archived

backend/app/services/simulation_runner.py       [约 20 行改动]
  职责：模拟结束后自动触发 archive 生成
  改动点：
  - _monitor_simulation() 中 run_state 变为 completed 时
  - 检查 agent_archives.json 是否存在
  - 不存在则调用 build_agent_archives()

backend/app/services/ontology_generator.py      [约 15 行改动]
  职责：提示词追加人物比例指引
  改动点：
  - ONTOLOGY_SYSTEM_PROMPT 或 _build_user_message()
  - 追加"人物型实体不少于5个，组织型不超过3个"的规则

backend/app/services/simulation_config_generator.py [约 10 行改动]
  职责：确保组织型 agent 走低活跃度分支
  改动点：
  - _generate_agent_config_by_rule() 中
  - 确认 GROUP_ENTITY_TYPES 的 agent 走"官方机构"分支

不需要修改的文件：
──────────────────────────────────────────────────────
run_parallel_simulation.py    模拟引擎核心不动
oasis_profile_generator.py    profile 生成逻辑不动
simulation_ipc.py             IPC 通信不动
report_agent.py               报告 Agent 逻辑不动（它通过 zep_tools 间接使用）
```

---

## 8. 测试方案

### 测试 1: agent_archives.json 生成

```
前提：有一个已完成模拟的 sim_dir
步骤：
  1. 调用 build_agent_archives(sim_dir)
  2. 检查 sim_dir/agent_archives.json 是否生成
  3. 验证 JSON 格式正确
  4. 验证 metadata 字段完整
  5. 验证 agents 数组长度 = simulation_config.json 中 agent_configs 数量
预期：生成成功，每个 agent 都有档案
```

### 测试 2: 每个 Agent 的 evidence

```
步骤：
  1. 读取 agent_archives.json
  2. 遍历每个 agent
  3. 检查 key_experiences、important_actions、posts 中每条记录
  4. 确认每条都有 source、round、platform、confidence
  5. 抽查 3 条记录，回到原始文件验证 evidence_snippet 正确
预期：所有 evidence 可溯源到原始数据
```

### 测试 3: data_sufficiency 判断

```
步骤：
  1. 用 sim_a2aa9c27508e（已知数据稀少）测试
  2. 验证所有 agent 的 data_sufficiency 都是 "low"
  3. 验证每个 agent 的 limitations 非空
  4. 构造一个有丰富数据的模拟目录（手动添加 actions 和 posts）
  5. 验证 data_sufficiency 正确变为 "medium" 或 "high"
预期：sufficiency 与实际数据量匹配
```

### 测试 4: interview_agents 不再超时

```
步骤：
  1. 确保无模拟进程运行
  2. 通过 API 调用 interview_agents（用之前修复的预检查）
  3. 计时，确认 < 5 秒返回
  4. 验证返回 mode: "archived"
预期：不再超时，快速返回 archived 结果
```

### 测试 5: archived interview 正常回答

```
步骤：
  1. 准备一个 data_sufficiency="high" 的 agent 档案（手动构造）
  2. 调用 interview_agents 的 archived fallback
  3. 验证返回第一人称回答
  4. 验证回答内容与档案 evidence 一致
  5. 验证返回标记 mode: "archived"
预期：archived interview 生成合理的角色回答
```

### 测试 6: data_sufficiency=low 时不编造

```
步骤：
  1. 用 sim_a2aa9c27508e 的真实数据（全是 low）
  2. 调用 archived interview
  3. 验证返回 status: "insufficient_data"
  4. 验证没有编造的第一人称叙述
预期：低数据 agent 不强行生成虚假回答
```

### 测试 7: 人物型 Agent 活跃度提升

```
步骤：
  1. 修改 ontology_generator 提示词（改造点 A）
  2. 用测试职业经历重新生成 ontology
  3. 验证人物型实体占比 >= 60%
  4. 验证人物型 agent 的 activity_level >= 0.6
  5. 验证组织型 agent 的 activity_level <= 0.3
预期：人物型 agent 数量和活跃度明显提升
```

---

## 9. 不建议做的事情

1. **从零重构 Agent Memory 系统** -- 现有数据已经够用，不需要新的存储系统，只需要整理和提炼

2. **把 archive 逻辑绑死在 IPC 生命周期里** -- IPC 不稳定，archive 应该独立生成，build_agent_archives 必须能在任意时刻调用

3. **让 archived interview 在证据不足时强行编造** -- data_sufficiency=low 时必须明确声明信息不足，不能用 LLM 的"创造力"弥补数据空白

4. **删除所有组织/公司 Agent** -- 组织是"世界规则"的载体，需要保留，改的是比例和行为策略，不是删除

5. **一次性生成过多人物 Agent（>10 个）** -- OASIS 引擎有性能限制，MVP 阶段 5-8 个 agent 足够

6. **大改模拟引擎核心（run_parallel_simulation.py）** -- 模拟引擎是 MiroFish 的核心，不动，改的是配置和提示词，不是引擎逻辑

7. **在 agent_archiver 中使用 LLM 做摘要** -- 增加延迟和成本，摘要工作留给报告 Agent 的 LLM 调用，archiver 只做数据整理

8. **把 Zep 图谱作为 agent_archives 的必要依赖** -- Zep 是补充，不是必要条件，本地数据（SQLite + actions.jsonl）必须能独立工作

---

## 10. 后续实现提示词

### 阶段 1: agent_archiver.py（独立模块，不依赖其他改动）

在 /Users/a/MiroFish/backend/app/services/ 下新建 agent_archiver.py，实现 build_agent_archives(simulation_dir, force=False) 函数。

输入：simulation_dir 绝对路径。

数据源读取顺序：
1. simulation_config.json -> agent_configs（agent_id, entity_name, entity_type）
2. twitter_profiles.csv 和 reddit_profiles.json -> profile 详情
3. twitter/actions.jsonl 和 reddit/actions.jsonl -> 行为记录
4. twitter_simulation.db 和 reddit_simulation.db -> 帖子、评论、点赞、关注
5. run_state.json -> 总轮数

agent_type 判断：entity_type.lower() 在 ["person","colleague","mentor","competitor","leader","entrepreneur","rolemodel","employee","manager","student","expert","professional"] 中的为 person，在 ["company","organization","industry","university","institution","educationinstitution","governmentagency","ngo","community","group","mediaoutlet"] 中的为 organization。

从 actions.jsonl 提取行为：过滤 event_type 不是 simulation_start/end/round_start/round_end 的记录。按 agent_id 分组。统计每个 agent 的 CREATE_POST、LIKE_POST、FOLLOW、CREATE_COMMENT 等动作。

从 SQLite 提取帖子：SELECT post_id, user_id, content, created_at, num_likes, num_dislikes, num_shares FROM post。按 user_id 关联到 agent_id。

从 SQLite 提取关系：SELECT * FROM follow（关注关系）、SELECT * FROM `like`（点赞关系）。这些表示 agent 之间的社交关系。

data_sufficiency 计算：
- high: 有效行为 >= 5 且有互动（点赞/评论/关注他人或被他人互动）
- medium: 有效行为 2-4
- low: 有效行为 < 2 或无互动

输出 schema 按本文档第 2 节的结构。写入 simulation_dir/agent_archives.json。

降级策略：如果某个数据源不存在，跳过并记录到 metadata.generation_config。如果 simulation_config.json 不存在，抛异常。

不用 LLM。纯 Python 数据处理。

完成后用 sim_a2aa9c27508e（路径 /Users/a/projects/MiroFish/backend/uploads/simulations/sim_a2aa9c27508e/）测试，验证所有 agent 的 data_sufficiency 都是 low。

### 阶段 2: interview_agents fallback（依赖阶段 1）

修改 /Users/a/MiroFish/backend/app/services/zep_tools.py 的 interview_agents 方法。

在已有的 Step 0 预检查之后、Step 1 live IPC 尝试之后，加入 fallback 逻辑：

如果 live IPC 不可用（env_status 不是 alive，或 IPC 超时），进入 archived fallback：
1. 检查 simulation_dir/agent_archives.json 是否存在
2. 不存在则调用 build_agent_archives(simulation_dir) 补生成
3. 读取目标 agent 的档案
4. 检查 data_sufficiency：
   - high/medium: 构造 LLM prompt，把档案作为上下文，要求第一人称回答
   - low: 返回 insufficient_data 结构
5. LLM prompt 模板：
   - system 角色设定为该 agent
   - 注入档案中的 key_experiences + posts + opinions
   - 注入 limitations
   - 要求只能基于档案回答，不能编造
6. 返回结果标记 mode: "archived"
7. 日志标记 ARCHIVED_INTERVIEW

确保三重超时保护生效：Step 0 预检查(0s) -> IPC 超时(60s) -> archived fallback(纯LLM, <30s)。

### 阶段 3: simulation_runner 自动触发（依赖阶段 1）

修改 /Users/a/MiroFish/backend/app/services/simulation_runner.py 的 _monitor_simulation() 方法。

在检测到 run_state.json 变为 completed 时，检查 simulation_dir/agent_archives.json 是否存在。如果不存在，调用 build_agent_archives(simulation_dir)。

这是非阻塞的，不影响模拟监控流程。

### 阶段 4: ontology_generator 提示词优化（独立，不依赖其他阶段）

修改 /Users/a/MiroFish/backend/app/services/ontology_generator.py。

在 _build_user_message() 的规则追加部分（第 268 行附近），加入：

```
7. 实体设计必须服务于职业平行宇宙的叙事需求：
   - 人物型实体至少5个（Colleague、Mentor、Competitor、Leader、Entrepreneur等）
   - 组织型实体不超过3个（Company、Industry等作为世界背景）
   - 每个人物应围绕用户的分叉选择，代表一种职业可能性
   - 例如：目标公司的HR、面试官、同事、竞争者、行业导师等
```

验证：用之前的测试职业经历重新生成 ontology，检查人物型实体占比。

---

## 实施优先级

```
阶段 1 (agent_archiver.py)     -> 最先做，独立模块，无风险
阶段 2 (interview fallback)    -> 第二做，解决超时的长期方案
阶段 3 (自动触发)              -> 第三做，5 行代码
阶段 4 (ontology 优化)         -> 可以和阶段 1 并行，独立无依赖
```

总代码量估计：约 500 行新增 + 100 行改动。

---

## 附录：当前数据流图

```
+----------------------------------------------------------------------+
|                         模拟启动阶段                                   |
|                                                                      |
|  ontology_generator.py                                               |
|     -> 生成 10 个实体类型（entity_types）                               |
|     -> LLM 生成具体实体（instances）                                   |
|                                                                      |
|  oasis_profile_generator.py                                          |
|     -> 每个 entity 生成 OASIS profile（人物/组织分支）                   |
|     -> 写入 twitter_profiles.csv + reddit_profiles.json               |
|                                                                      |
|  simulation_config_generator.py                                      |
|     -> 每个 agent 生成 activity_level、active_hours 等行为参数          |
|     -> 写入 simulation_config.json                                   |
|     -> 创建 Zep 图谱（graph_id）并写入初始实体                         |
+------------------------------+---------------------------------------+
                               |
                               v
+----------------------------------------------------------------------+
|                       模拟运行阶段                                     |
|                                                                      |
|  run_parallel_simulation.py (子进程)                                  |
|     |                                                                |
|     +-- OASIS 引擎运行                                                |
|     |   -> twitter_simulation.db / reddit_simulation.db (SQLite)      |
|     |   -> 记录: user, post, trace(sign_up/create_post/interview)...  |
|     |                                                                |
|     +-- PlatformActionLogger                                         |
|     |   -> twitter/actions.jsonl + reddit/actions.jsonl               |
|     |   -> 记录: round, agent_id, action_type, action_args            |
|     |                                                                |
|     +-- env_status.json 心跳                                         |
|                                                                      |
|  simulation_runner.py (监控进程，2s 轮询)                              |
|     |                                                                |
|     +-- 读取 actions.jsonl -> 更新 run_state.json                     |
|     |                                                                |
|     +-- ZepGraphMemoryUpdater (后台线程)                              |
|         +-- 读取 actions.jsonl 中的新活动                             |
|         +-- 转换为自然语言描述                                        |
|         +-- client.graph.add() -> Zep Cloud 图谱                      |
+------------------------------+---------------------------------------+
                               |
                               v
+----------------------------------------------------------------------+
|                       档案生成阶段（新增）                              |
|                                                                      |
|  agent_archiver.py                                                   |
|     -> 读取 simulation_config.json, profiles, actions.jsonl,          |
|        SQLite DB, run_state.json                                     |
|     -> 输出 agent_archives.json（每个 Agent 的结构化档案）              |
|     -> 包含 profile、经历、行为、关系、观点、数据充分度、证据索引        |
+------------------------------+---------------------------------------+
                               |
                               v
+----------------------------------------------------------------------+
|                       报告生成阶段                                     |
|                                                                      |
|  report_agent.py (ReACT 模式)                                        |
|     |                                                                |
|     +-- 阶段1: plan_outline()                                        |
|     |   +-- zep_tools.get_simulation_context(graph_id)               |
|     |                                                                |
|     +-- 阶段2: _generate_section_react() x N 章节                    |
|     |   +-- insight_forge / panorama_search / quick_search           |
|     |   +-- interview_agents()                                       |
|     |       +-- live IPC（如果模拟进程存活）                           |
|     |       +-- archived fallback（如果进程已退出，读 agent_archives） |
|     |                                                                |
|     +-- 阶段3: 组装报告                                              |
|         -> reports/{report_id}/section_XX.md + full_report.md        |
+----------------------------------------------------------------------+
```
