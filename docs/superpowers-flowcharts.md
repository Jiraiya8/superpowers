# Superpowers 流程图详解

## 目录大纲

- [1. Superpowers 整体架构](#1-superpowers-整体架构) - 核心层、技能库、平台适配层架构
- [2. 主开发工作流](#2-主开发工作流) - 从用户发起任务到完成的完整流程
- [3. 技能调用流程](#3-技能调用流程) - 技能判断与调用的决策流程
- [4. Brainstorming 技能流程](#4-brainstorming-技能流程) - 苏格拉底式设计精炼流程
- [5. Git Worktrees 技能流程](#5-git-worktrees-技能流程) - 隔离工作空间设置
- [6. Writing Plans 技能流程](#6-writing-plans-技能流程) - 详细实现计划编写
- [7. Subagent-driven Development 技能流程](#7-subagent-driven-development-技能流程) - 两级审查执行流程
- [8. TDD 循环](#8-tdd-test-driven-development-循环) - RED-GREEN-REFACTOR 核心循环
- [9. TDD 验证流程详情](#9-tdd-验证流程详情) - TDD 详细验证步骤
- [10. Systematic Debugging 四阶段流程](#10-systematic-debugging-四阶段流程) - 根因调查、模式分析、假设测试、实现
- [11. Verification Before Completion 流程](#11-verification-before-completion-流程) - 声明前必须有证据
- [12. Code Review 流程](#12-code-review-流程) - 请求审查与接收审查
- [13. Finishing Development Branch 流程](#13-finishing-development-branch-流程) - 合并/PR/保留/丢弃选项
- [14. Writing Skills 流程](#14-writing-skills-流程) - RED-GREEN-REFACTOR 创建技能
- [15. Dispatching Parallel Agents 流程](#15-dispatching-parallel-agents-流程) - 并行分派独立任务
- [16. 技能类型分类](#16-技能类型分类) - 测试类、调试类、协作类、元类
- [17. 平台适配映射](#17-平台适配映射) - Claude Code、OpenCode、Copilot、Gemini 工具映射
- [18. 用户指令优先级](#18-用户指令优先级) - 用户指令 > 技能 > 默认提示
- [19. 技能执行中的红旗信号](#19-技能执行中的红旗信号) - using-superpowers、TDD、调试、验证红旗
- [20. 完整会话生命周期](#20-完整会话生命周期) - 从会话开始到结束的全流程
- [关键概念总结](#关键概念总结) - 各核心组件用途对照表

---

## 1. Superpowers 整体架构

```mermaid
graph TB
    subgraph 核心层["核心层"]
        SkillsSystem["技能系统 (Skills)"]
        Hooks["钩子系统 (Hooks)"]
        Agents["代理配置 (AGENTS.md)"]
    end

    subgraph 技能库["技能库"]
        ProcessSkills["流程技能"]
        DebugSkills["调试技能"]
        CollabSkills["协作技能"]
        MetaSkills["元技能"]
    end

    subgraph 平台适配["平台适配层"]
        ClaudeCode["Claude Code"]
        Codex["Codex CLI/App"]
        OpenCode["OpenCode"]
        Copilot["Copilot CLI"]
        Gemini["Gemini CLI"]
        Cursor["Cursor"]
    end

    SkillsSystem --> 技能库
    Hooks --> SkillsSystem
    Agents --> SkillsSystem
    技能库 --> PlatformAdapt["平台适配工具"]
    PlatformAdapt --> 平台适配
```

## 2. 主开发工作流

```mermaid
graph TB
    Start["用户发起任务"] --> SkillCheck{"是否有技能适用?"}
    
    SkillCheck -->|"是 (哪怕只有1%可能性)"| InvokeSkill["调用 Skill 工具"]
    SkillCheck -->|"绝对没有"| DirectResponse["直接响应"]
    
    InvokeSkill --> Announce["声明: 使用 [技能名] 进行 [目的]"]
    Announce --> HasChecklist{"技能有检查清单?"}
    
    HasChecklist -->|"是"| CreateTodo["创建 TodoWrite 任务项"]
    HasChecklist -->|"否"| FollowSkill["遵循技能指令"]
    
    CreateTodo --> FollowSkill
    
    FollowSkill --> Brainstorm{"需要创意工作?"}
    
    Brainstorm -->|"是"| BrainstormingSkill["brainstorming 技能"]
    Brainstorm -->|"否,有规格"| GitWorktrees["using-git-worktrees 技能"]
    
    BrainstormingSkill --> DesignApproved{"设计已批准?"}
    DesignApproved -->|"否"| ReviseDesign["修订设计"]
    ReviseDesign --> BrainstormingSkill
    
    DesignApproved -->|"是"| GitWorktrees
    GitWorktrees --> WritingPlans["writing-plans 技能"]
    
    WritingPlans --> ChooseExecution{"选择执行方式?"}
    ChooseExecution -->|"子代理驱动 (推荐)"| SubagentDriven["subagent-driven-development"]
    ChooseExecution -->|"内联执行"| ExecutingPlans["executing-plans 技能"]
    
    SubagentDriven --> TDD["test-driven-development 技能"]
    ExecutingPlans --> TDD
    
    TDD --> RequestReview["requesting-code-review 技能"]
    RequestReview --> MoreTasks{"更多任务?"}
    
    MoreTasks -->|"是"| SubagentDriven
    MoreTasks -->|"否"| FinishingBranch["finishing-a-development-branch 技能"]
    
    FinishingBranch --> End["完成"]
```

## 3. 技能调用流程

```mermaid
graph TB
    UserMessage["用户消息接收"] --> PlanModeCheck{"即将进入计划模式?"}
    
    PlanModeCheck -->|"是"| BrainstormedCheck{"已经头脑风暴?"}
    PlanModeCheck -->|"否"| MightApply{"可能有任何技能适用?"}
    
    BrainstormedCheck -->|"否"| InvokeBrainstorm["调用 brainstorming 技能"]
    BrainstormedCheck -->|"是"| MightApply
    
    InvokeBrainstorm --> MightApply
    
    MightApply -->|"是,哪怕1%"| InvokeSkillTool["调用 Skill 工具"]
    MightApply -->|"绝对没有"| Respond["响应 (包括澄清问题)"]
    
    InvokeSkillTool --> AnnounceUse["声明: 使用 [技能] 进行 [目的]"]
    AnnounceUse --> ChecklistCheck{"有检查清单?"}
    
    ChecklistCheck -->|"是"| CreateTodoItem["创建 TodoWrite 每项任务"]
    ChecklistCheck -->|"否"| FollowDirectly["直接遵循技能"]
    
    CreateTodoItem --> FollowDirectly
    FollowDirectly --> Respond
```

## 4. Brainstorming 技能流程

```mermaid
graph TB
    ExploreContext["探索项目上下文"] --> VisualQ{"涉及视觉问题?"}
    
    VisualQ -->|"是"| OfferVisual["提供 Visual Companion (单独消息)"]
    VisualQ -->|"否"| AskQuestions["逐个提问澄清问题"]
    
    OfferVisual --> AskQuestions
    
    AskQuestions --> ProposeApproaches["提出 2-3 种方案"]
    ProposeApproaches --> PresentDesign["分段展示设计"]
    
    PresentDesign --> UserApprove{"用户批准设计?"}
    
    UserApprove -->|"否,修订"| PresentDesign
    UserApprove -->|"是"| WriteDesignDoc["编写设计文档"]
    
    WriteDesignDoc --> SpecSelfReview["规格自审查"]
    SpecSelfReview --> UserReviewSpec{"用户审查规格?"}
    
    UserReviewSpec -->|"请求修改"| WriteDesignDoc
    UserReviewSpec -->|"批准"| InvokeWritingPlans["调用 writing-plans 技能"]
    
    InvokeWritingPlans --> End["进入计划阶段"]
```

## 5. Git Worktrees 技能流程

```mermaid
graph TB
    CheckExisting["检查现有目录"] --> FoundWorktrees{"找到 .worktrees 或 worktrees?"}
    
    FoundWorktrees -->|"是"| UseExisting["使用现有目录"]
    FoundWorktrees -->|"否"| CheckClaudeMd["检查 CLAUDE.md 配置"]
    
    CheckClaudeMd --> HasPreference{"有配置偏好?"}
    HasPreference -->|"是"| UsePreference["使用配置路径"]
    HasPreference -->|"否"| AskUser["询问用户选择"]
    
    AskUser --> UserChoice["用户选择"]
    UseExisting --> VerifyIgnore["验证目录已忽略"]
    UsePreference --> VerifyIgnore
    UserChoice --> VerifyIgnore
    
    VerifyIgnore --> IsIgnored{"git check-ignore 确认?"}
    
    IsIgnored -->|"否"| AddGitignore["添加到 .gitignore + 提交"]
    IsIgnored -->|"是"| CreateWorktree["创建 worktree"]
    
    AddGitignore --> CreateWorktree
    
    CreateWorktree --> ProjectSetup["自动检测并运行项目设置"]
    ProjectSetup --> VerifyTests["验证测试基线"]
    
    VerifyTests --> TestsPass{"测试通过?"}
    TestsPass -->|"是"| ReportReady["报告就绪"]
    TestsPass -->|"否"| ReportFail["报告失败 + 询问是否继续"]
```

## 6. Writing Plans 技能流程

```mermaid
graph TB
    Announce["声明使用 writing-plans 技能"] --> ScopeCheck["检查规格范围"]
    
    ScopeCheck --> TooLarge{"跨多个独立子系统?"}
    TooLarge -->|"是"| SuggestSplit["建议分解为多个计划"]
    TooLarge -->|"否"| FileStructure["映射文件结构"]
    
    SuggestSplit --> UserDecision{"用户同意分解?"}
    UserDecision -->|"是"| CreateFirstPlan["创建第一个子计划"]
    UserDecision -->|"否"| FileStructure
    
    FileStructure --> DefineTasks["定义任务 (2-5分钟每步)"]
    DefineTasks --> WriteHeader["编写计划文档头部"]
    
    WriteHeader --> TaskStructure["编写任务结构"]
    TaskStructure --> NoPlaceholders["确保无占位符"]
    
    NoPlaceholders --> SelfReview["自审查计划"]
    SelfReview --> SpecCoverage{"规格覆盖完整?"}
    
    SpecCoverage -->|"有缺失"| AddMissingTask["添加缺失任务"]
    SpecCoverage -->|"完整"| PlaceholderScan["占位符扫描"]
    
    AddMissingTask --> SelfReview
    PlaceholderScan --> TypeConsistency["类型一致性检查"]
    
    TypeConsistency --> SavePlan["保存计划文档"]
    SavePlan --> OfferExecution["提供执行选项"]
    
    OfferExecution --> SubagentChoice["子代理驱动 (推荐)"]
    OfferExecution --> InlineChoice["内联执行"]
    
    SubagentChoice --> InvokeSubagent["调用 subagent-driven-development"]
    InlineChoice --> InvokeExecuting["调用 executing-plans"]
```

## 7. Subagent-driven Development 技能流程

```mermaid
graph TB
    ReadPlan["读取计划,提取所有任务"] --> CreateTodo["创建 TodoWrite"]
    
    CreateTodo --> MoreTasks{"更多任务?"}
    
    MoreTasks -->|"是"| DispatchImplementer["分派实现子代理"]
    MoreTasks -->|"否"| FinalReview["分派最终代码审查"]
    
    DispatchImplementer --> HasQuestions{"子代理有问题?"}
    
    HasQuestions -->|"是"| AnswerQuestions["回答问题,提供上下文"]
    HasQuestions -->|"否"| ImplementCommit["子代理实现、测试、提交、自审"]
    
    AnswerQuestions --> DispatchImplementer
    ImplementCommit --> DispatchSpecReviewer["分派规格审查子代理"]
    
    DispatchSpecReviewer --> SpecCompliant{"规格合规?"}
    
    SpecCompliant -->|"否"| FixSpecGaps["实现子代理修复规格差距"]
    SpecCompliant -->|"是"| DispatchQualityReviewer["分派代码质量审查子代理"]
    
    FixSpecGaps --> DispatchSpecReviewer
    DispatchQualityReviewer --> QualityApproved{"质量批准?"}
    
    QualityApproved -->|"否"| FixQualityIssues["实现子代理修复质量问题"]
    QualityApproved -->|"是"| MarkComplete["标记任务完成"]
    
    FixQualityIssues --> DispatchQualityReviewer
    MarkComplete --> MoreTasks
    
    FinalReview --> InvokeFinishing["调用 finishing-a-development-branch"]
```

## 8. TDD (Test-Driven Development) 循环

```mermaid
graph LR
    subgraph RED["RED 阶段"]
        WriteFailTest["编写失败测试"]
        VerifyFail["验证测试失败"]
    end
    
    subgraph GREEN["GREEN 阶段"]
        MinimalCode["编写最小代码"]
        VerifyPass["验证测试通过"]
    end
    
    subgraph REFACTOR["REFACTOR 阶段"]
        Cleanup["清理代码"]
        StayGreen["保持测试通过"]
    end
    
    WriteFailTest --> VerifyFail
    VerifyFail -->|"正确失败"| MinimalCode
    VerifyFail -->|"错误失败"| WriteFailTest
    
    MinimalCode --> VerifyPass
    VerifyPass -->|"通过"| Cleanup
    VerifyPass -->|"失败"| MinimalCode
    
    Cleanup --> StayGreen
    StayGreen --> VerifyPass
    VerifyPass -->|"全部通过"| NextTest["下一个测试"]
    
    NextTest --> WriteFailTest
```

## 9. TDD 验证流程详情

```mermaid
graph TB
    StartTDD["开始 TDD"] --> WriteTest["编写一个最小测试"]
    
    WriteTest --> TestQuality{"测试质量检查"}
    
    TestQuality -->|"好"| RunFail["运行测试 - 验证失败"]
    TestQuality -->|"坏"| FixTest["修复测试"]
    
    FixTest --> WriteTest
    
    RunFail --> FailCorrect{"失败正确?"}
    
    FailCorrect -->|"是,正确失败"| WriteCode["编写最小代码"]
    FailCorrect -->|"否,错误失败"| DebugTest["调试测试"]
    FailCorrect -->|"通过 (不好)"| FixExistingTest["测试已存在行为 - 修复"]
    
    DebugTest --> WriteTest
    FixExistingTest --> WriteTest
    
    WriteCode --> RunPass["运行测试 - 验证通过"]
    
    RunPass --> AllPass{"全部通过?"}
    
    AllPass -->|"是"| RefactorCheck{"需要重构?"}
    AllPass -->|"否"| FixCode["修复代码"]
    
    FixCode --> WriteCode
    RefactorCheck -->|"是"| RefactorCode["重构代码"]
    RefactorCheck -->|"否"| CommitCheck{"需要提交?"}
    
    RefactorCode --> RunPass
    CommitCheck -->|"是"| Commit["提交"]
    CommitCheck -->|"否"| NextFeature["下一个功能"]
    
    Commit --> NextFeature
    NextFeature --> StartTDD
```

## 10. Systematic Debugging 四阶段流程

```mermaid
graph TB
    subgraph Phase1["阶段1: 根因调查"]
        ReadErrors["仔细阅读错误信息"]
        Reproduce["稳定复现问题"]
        CheckChanges["检查最近变更"]
        GatherEvidence["多组件系统:收集证据"]
        TraceFlow["追踪数据流"]
    end
    
    subgraph Phase2["阶段2: 模式分析"]
        FindWorking["找到工作示例"]
        CompareRef["对比参考实现"]
        IdentifyDiff["识别差异"]
        UnderstandDep["理解依赖"]
    end
    
    subgraph Phase3["阶段3: 假设测试"]
        FormHypothesis["形成单一假设"]
        TestMinimally["最小化测试"]
        VerifyResult["验证结果"]
        DontKnow["不理解时求助"]
    end
    
    subgraph Phase4["阶段4: 实现"]
        CreateTest["创建失败测试"]
        SingleFix["实现单一修复"]
        VerifyFix["验证修复"]
        FixFailed{"修复失败?"}
        QuestionArch["质疑架构"]
    end
    
    ReadErrors --> Reproduce --> CheckChanges --> GatherEvidence --> TraceFlow
    TraceFlow --> Phase2Start["进入阶段2"]
    
    FindWorking --> CompareRef --> IdentifyDiff --> UnderstandDep
    UnderstandDep --> Phase3Start["进入阶段3"]
    
    FormHypothesis --> TestMinimally --> VerifyResult --> DontKnow
    VerifyResult -->|"成功"| Phase4Start["进入阶段4"]
    VerifyResult -->|"失败"| FormHypothesis
    
    CreateTest --> SingleFix --> VerifyFix --> FixFailed
    FixFailed -->|"成功"| DebugDone["调试完成"]
    FixFailed -->|"失败 (<3次)"| Phase1Return["返回阶段1"]
    FixFailed -->|"失败 (≥3次)"| QuestionArch
    
    QuestionArch --> DiscussPartner["与用户讨论架构"]
```

## 11. Verification Before Completion 流程

```mermaid
graph TB
    AboutToClaim["即将声明完成"] --> IdentifyCommand["确定验证命令"]
    
    IdentifyCommand --> RunFull["运行完整命令 (全新)"]
    RunFull --> ReadOutput["读取完整输出 + 检查退出码"]
    
    ReadOutput --> VerifyClaim{"输出确认声明?"}
    
    VerifyClaim -->|"否"| StateActual["陈述实际状态 + 证据"]
    VerifyClaim -->|"是"| StateWithEvidence["声明 + 证据"]
    
    StateWithEvidence --> ThenClaim["然后才能声明"]
    
    AboutToClaim --> RedFlags{"有红旗信号?"}
    
    RedFlags -->|"使用'应该/可能/似乎'"| RunFull
    RedFlags -->|"表达满意前验证"| RunFull
    RedFlags -->|"提交前无验证"| RunFull
    RedFlags -->|"信任代理报告"| RunFull
```

## 12. Code Review 流程

```mermaid
graph TB
    subgraph Requesting["请求审查"]
        GetSHAs["获取 Git SHAs"]
        DispatchReviewer["分派 code-reviewer 子代理"]
        FillTemplate["填写模板"]
        ActOnFeedback["处理反馈"]
    end
    
    subgraph Receiving["接收审查"]
        ReadComplete["完整读取反馈"]
        Understand["理解需求"]
        Verify["验证代码库现实"]
        Evaluate["评估技术合理性"]
        Respond["技术响应或反驳"]
        ImplementOne["逐项实现"]
    end
    
    GetSHAs --> DispatchReviewer --> FillTemplate --> ActOnFeedback
    
    ReadComplete --> Understand --> Verify --> Evaluate --> Respond --> ImplementOne
    
    Evaluate --> Unclear{"不清楚?"}
    Unclear -->|"是"| AskClarify["停止 - 询问澄清"]
    Unclear -->|"否"| TechnicallyCorrect{"技术上正确?"}
    
    TechnicallyCorrect -->|"是"| ImplementOne
    TechnicallyCorrect -->|"否"| PushBack["技术反驳"]
```

## 13. Finishing Development Branch 流程

```mermaid
graph TB
    VerifyTests["验证测试通过"] --> TestsPass{"测试通过?"}
    
    TestsPass -->|"否"| ShowFailures["显示失败 + 停止"]
    TestsPass -->|"是"| DetermineBase["确定基础分支"]
    
    DetermineBase --> PresentOptions["展示 4 个选项"]
    
    PresentOptions --> Option1["1. 本地合并"]
    PresentOptions --> Option2["2. 推送并创建 PR"]
    PresentOptions --> Option3["3. 保持分支"]
    PresentOptions --> Option4["4. 丢弃工作"]
    
    Option1 --> CheckoutBase["切换到基础分支"]
    CheckoutBase --> PullLatest["拉取最新"]
    PullLatest --> MergeBranch["合并特性分支"]
    MergeBranch --> VerifyMerge["验证合并后测试"]
    VerifyMerge --> DeleteBranch["删除分支"]
    DeleteBranch --> CleanupWorktree["清理 worktree"]
    
    Option2 --> PushBranch["推送分支"]
    PushBranch --> CreatePR["创建 PR"]
    CreatePR --> CleanupWorktree
    
    Option3 --> KeepWorktree["保留 worktree"]
    
    Option4 --> ConfirmDiscard["确认丢弃"]
    ConfirmDiscard --> TypedConfirm{"输入 'discard'?"}
    TypedConfirm -->|"是"| ForceDelete["强制删除分支"]
    TypedConfirm -->|"否"| PresentOptions
    ForceDelete --> CleanupWorktree
```

## 14. Writing Skills 流程

```mermaid
graph TB
    subgraph RED["RED: 编写失败测试"]
        CreatePressure["创建压力场景"]
        RunBaseline["无技能运行 - 记录基线行为"]
        IdentifyPattern["识别合理化模式"]
    end
    
    subgraph GREEN["GREEN: 编写最小技能"]
        ValidName["名称仅用字母数字连字符"]
        YAMLFrontmatter["YAML 前置元数据"]
        GoodDescription["描述以 'Use when...' 开头"]
        Keywords["关键词覆盖"]
        ClearOverview["清晰概述 + 核心原则"]
        AddressBaseline["解决基线失败"]
        OneExample["一个优秀示例"]
        RunWithSkill["有技能运行 - 验证合规"]
    end
    
    subgraph REFACTOR["REFACTOR: 关闭漏洞"]
        IdentifyRationalizations["识别新的合理化"]
        AddCounters["添加明确反制措施"]
        BuildTable["构建合理化表"]
        CreateRedFlags["创建红旗列表"]
        ReTest["重测直到无懈可击"]
    end
    
    CreatePressure --> RunBaseline --> IdentifyPattern --> GREENStart["进入 GREEN"]
    
    ValidName --> YAMLFrontmatter --> GoodDescription --> Keywords --> ClearOverview --> AddressBaseline --> OneExample --> RunWithSkill --> REFACTORStart["进入 REFACTOR"]
    
    IdentifyRationalizations --> AddCounters --> BuildTable --> CreateRedFlags --> ReTest --> QualityChecks["质量检查"]
    
    QualityChecks --> Deploy["部署技能"]
```

## 15. Dispatching Parallel Agents 流程

```mermaid
graph TB
    MultipleFailures["多个失败"] --> AreIndependent{"失败独立?"}
    
    AreIndependent -->|"否 - 相关"| SingleAgent["单个代理调查全部"]
    AreIndependent -->|"是"| CanParallel{"可并行?"}
    
    CanParallel -->|"否 - 共享状态"| SequentialAgents["顺序代理"]
    CanParallel -->|"是"| ParallelDispatch["并行分派"]
    
    ParallelDispatch --> IdentifyDomains["识别独立域"]
    IdentifyDomains --> CreateTasks["创建聚焦代理任务"]
    CreateTasks --> DispatchAll["并行分派所有代理"]
    
    DispatchAll --> ReviewResults["审查结果"]
    ReviewResults --> CheckConflicts{"检查冲突"}
    
    CheckConflicts -->|"有"| ResolveConflicts["解决冲突"]
    CheckConflicts -->|"无"| RunFullSuite["运行完整测试套件"]
    
    ResolveConflicts --> RunFullSuite
    RunFullSuite --> Integrate["整合所有变更"]
```

## 16. 技能类型分类

```mermaid
graph TB
    subgraph Testing["测试类"]
        TDD["test-driven-development\nRED-GREEN-REFACTOR 循环"]
    end
    
    subgraph Debugging["调试类"]
        SysDebug["systematic-debugging\n4阶段根因流程"]
        VerifyComplete["verification-before-completion\n验证后再声明"]
    end
    
    subgraph Collaboration["协作类"]
        Brainstorm["brainstorming\n苏格拉底式设计精炼"]
        WritePlans["writing-plans\n详细实现计划"]
        ExecPlans["executing-plans\n批量执行+检查点"]
        Dispatch["dispatching-parallel-agents\n并行子代理"]
        RequestReview["requesting-code-review\n预审查清单"]
        ReceiveReview["receiving-code-review\n响应反馈"]
        GitWorktrees["using-git-worktrees\n并行开发分支"]
        FinishBranch["finishing-a-development-branch\n合并/PR 决策"]
        SubagentDev["subagent-driven-development\n快速迭代+两级审查"]
    end
    
    subgraph Meta["元类"]
        WritingSkills["writing-skills\n创建新技能"]
        UsingSuper["using-superpowers\n技能系统介绍"]
    end
```

## 17. 平台适配映射

```mermaid
graph TB
    subgraph Tools["工具映射"]
        TodoWrite["TodoWrite"]
        Task["Task 子代理"]
        Skill["Skill 工具"]
        Read["Read"]
        Write["Write"]
        Edit["Edit"]
        Bash["Bash"]
    end
    
    subgraph ClaudeCode["Claude Code"]
        CC_TodoWrite["TodoWrite (原生)"]
        CC_Task["Task 工具"]
        CC_Skill["Skill 工具"]
        CC_Read["Read 工具"]
        CC_Write["Write 工具"]
        CC_Edit["Edit 工具"]
        CC_Bash["Bash 工具"]
    end
    
    subgraph OpenCode["OpenCode"]
        OC_TodoWrite["todowrite (原生)"]
        OC_Task["@mention 子代理"]
        OC_Skill["skill 工具"]
        OC_Native["原生工具"]
    end
    
    subgraph Copilot["Copilot CLI"]
        CP_TodoWrite["skill 工具"]
        CP_Task["skill 工具"]
        CP_Skill["skill 工具"]
    end
    
    subgraph Gemini["Gemini CLI"]
        GM_TodoWrite["activate_skill"]
        GM_Task["activate_skill"]
        GM_Skill["activate_skill"]
    end
    
    TodoWrite --> CC_TodoWrite
    TodoWrite --> OC_TodoWrite
    TodoWrite --> CP_TodoWrite
    TodoWrite --> GM_TodoWrite
    
    Task --> CC_Task
    Task --> OC_Task
    Task --> CP_Task
    Task --> GM_Task
    
    Skill --> CC_Skill
    Skill --> OC_Skill
    Skill --> CP_Skill
    Skill --> GM_Skill
```

## 18. 用户指令优先级

```mermaid
graph TB
    subgraph Priority["优先级顺序"]
        P1["用户明确指令 (最高)\nCLAUDE.md / GEMINI.md / AGENTS.md / 直接请求"]
        P2["Superpowers 技能 (中等)\n覆盖默认系统行为"]
        P3["默认系统提示 (最低)"]
    end
    
    P1 --> P2 --> P3
    
    Example["示例: CLAUDE.md 说 '不使用 TDD'\n但 TDD 技能说 '总是使用 TDD'\n→ 遵循用户指令"]
```

## 19. 技能执行中的红旗信号

```mermaid
graph TB
    subgraph UsingSuperpowers["using-superpowers 红旗"]
        US1["'这只是简单问题'"]
        US2["'需要更多上下文'"]
        US3["'先探索代码库'"]
        US4["'不需要正式技能'"]
        US5["'我记得这个技能'"]
        US6["'技能太过度了'"]
    end
    
    subgraph TDD["TDD 红旗"]
        TDD1["代码先于测试"]
        TDD2["测试立即通过"]
        TDD3["'简单到不需要测试'"]
        TDD4["'事后测试也一样'"]
        TDD5["'保留作为参考'"]
        TDD6["'只是这一次'"]
    end
    
    subgraph Debugging["调试红旗"]
        DBG1["'快速修复,稍后调查'"]
        DBG2["'试试改变 X'"]
        DBG3["'多个修改一起'"]
        DBG4["'假设是 X,先修复'"]
        DBG5["'3+ 次修复后继续'"]
    end
    
    subgraph Verification["验证红旗"]
        VER1["使用 '应该/可能/似乎'"]
        VER2["验证前表达满意"]
        VER3["信任代理成功报告"]
        VER4["依赖部分验证"]
    end
    
    US1 --> STOP_US["停止 → 调用技能"]
    US2 --> STOP_US
    US3 --> STOP_US
    US4 --> STOP_US
    US5 --> STOP_US
    US6 --> STOP_US
    
    TDD1 --> STOP_TDD["停止 → 删除代码,重新开始"]
    TDD2 --> STOP_TDD
    TDD3 --> STOP_TDD
    TDD4 --> STOP_TDD
    TDD5 --> STOP_TDD
    TDD6 --> STOP_TDD
    
    DBG1 --> STOP_DBG["停止 → 返回阶段1"]
    DBG2 --> STOP_DBG
    DBG3 --> STOP_DBG
    DBG4 --> STOP_DBG
    DBG5 --> QuestionArch["质疑架构"]
    
    VER1 --> STOP_VER["停止 → 运行验证"]
    VER2 --> STOP_VER
    VER3 --> STOP_VER
    VER4 --> STOP_VER
```

## 20. 完整会话生命周期

```mermaid
graph TB
    SessionStart["会话开始"] --> LoadAgents["加载 AGENTS.md"]
    
    LoadAgents --> UserInput["用户输入"]
    
    UserInput --> SkillEvaluation["技能评估"]
    
    SkillEvaluation --> CreativeWork{"创意工作?"}
    CreativeWork -->|"是"| BrainstormFlow["Brainstorming 流程"]
    CreativeWork -->|"否"| HasSpec{"有规格?"}
    
    BrainstormFlow --> DesignDone["设计完成"]
    DesignDone --> HasSpec
    
    HasSpec -->|"是"| SetupWorktree["设置 Git Worktree"]
    HasSpec -->|"否"| MoreContext["需要更多上下文"]
    
    MoreContext --> UserInput
    SetupWorktree --> CreatePlan["创建实现计划"]
    
    CreatePlan --> ChooseMode{"选择执行模式"}
    
    ChooseMode -->|"子代理"| SubagentLoop["子代理循环"]
    ChooseMode -->|"内联"| InlineLoop["内联循环"]
    
    SubagentLoop --> TaskExecution["任务执行 (TDD)"]
    InlineLoop --> TaskExecution
    
    TaskExecution --> CodeReview["代码审查"]
    
    CodeReview --> AllComplete{"所有任务完成?"}
    
    AllComplete -->|"否"| SubagentLoop
    AllComplete -->|"是"| FinalVerify["最终验证"]
    
    FinalVerify --> ChooseFinish{"选择完成方式"}
    
    ChooseFinish -->|"合并"| MergeLocal["本地合并"]
    ChooseFinish -->|"PR"| CreatePR["创建 PR"]
    ChooseFinish -->|"保留"| KeepBranch["保留分支"]
    ChooseFinish -->|"丢弃"| DiscardWork["丢弃工作"]
    
    MergeLocal --> Cleanup["清理 Worktree"]
    CreatePR --> Cleanup
    DiscardWork --> Cleanup
    KeepBranch --> SessionEnd["会话结束"]
    
    Cleanup --> SessionEnd
```

---

## 关键概念总结

| 组件 | 用途 |
|------|------|
| **using-superpowers** | 入口点 - 判断是否有/哪些技能适用 |
| **brainstorming** | 通过对话精炼设计 |
| **using-git-worktrees** | 设置隔离工作空间 |
| **writing-plans** | 详细实现计划编写 |
| **subagent-driven-development** | 用全新子代理执行计划 |
| **test-driven-development** | RED-GREEN-REFACTOR 循环 |
| **systematic-debugging** | 4阶段根因调查流程 |
| **verification-before-completion** | 声明前必须有证据 |
| **requesting-code-review** | 提交前审查流程 |
| **finishing-a-development-branch** | 合并/PR/清理工作流 |
| **writing-skills** | TDD方法创建文档技能 |