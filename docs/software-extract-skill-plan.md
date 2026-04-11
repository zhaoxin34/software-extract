# software-extract Skill 完善计划

## 背景

当前 `software-extract` skill 只有一个粗略的工作流程骨架，通过三个真实项目（my-secretary、memory、plato）的萃取实验，验证了萃取格式并明确了"精华"的定义。

**精华的定义**（已验证）：
- 架构设计模式（Clean Architecture、分层模式）
- 核心抽象接口（Provider、Storage 等）
- Pipeline 编排模式
- 关键技术决策
- 可复用的提示词模板

**不是精华**：
- 详细的业务规则（这是 SDD Spec 的职责）
- 具体的 CLI 命令格式
- 输入/输出格式的每个细节

---

## 待办事项

### 已完成 ✅

- [x] **补充萃取输出格式定义**
  - 通过 memory 项目验证了格式
  - 定义了每个层级的萃取内容（description, purpose, responsibilities, features, interfaces, tech_stack）

- [x] **添加输出示例**
  - my-secretary: `raw/my-secretary/extracted-knowledge.md`
  - memory: `raw/memory/extracted-knowledge.md`

- [x] **完善 SKILL.md 文档**
  - 补充步骤 5 的详细萃取逻辑
  - 添加萃取格式模板
  - 添加 algorithms 和 fallback_strategies 维度

- [x] **优化 description**
  - 让描述更"pushy"，提高触发准确性
  - 明确 skill 触发条件

### 进行中 🔄

暂无进行中的任务

### 待完成 ⏳

- [ ] **添加边界情况处理**
  - 非 Git 项目（纯本地目录）如何处理？
  - 二进制文件或大文件如何处理？
  - 萃取过程中出错如何处理？

- [ ] **创建测试用例**
  - 编写 2-3 个真实场景的测试 prompt
  - 保存到 `evals/evals.json`

- [ ] **运行测试并迭代优化**
  - 根据测试结果完善 skill

- [ ] **创建测试用例**
  - 编写 2-3 个真实场景的测试 prompt
  - 保存到 `evals/evals.json`

- [ ] **运行测试并迭代优化**
  - 根据测试结果完善 skill

---

## 萃取内容模板（已验证）

### 整体结构

```yaml
项目名称:
  description: 项目描述
  purpose: 存在目的/解决的问题

  模块1:
    description: 模块功能描述
    purpose: 存在目的
    responsibilities: [职责1, 职责2]

    features:
      - name: Feature名称
        description: 功能描述
        specs:
          - name: Spec名称
            description: 规格描述
            decision: 关键设计决策

    interfaces:
      - name: 接口名称
        input: 输入
        output: 输出
        usage: 使用场景

    tech_stack: [技术1, 技术2]

  模块2:
    ...

  关键设计决策:
    - 决策1: 理由
    - 决策2: 理由

  可复用的提示词模板: |
    创建类似系统...
```

### Feature/Spec 结构

```yaml
features:
  - name: Feature名称
    description: 功能描述
    specs:
      - name: Spec名称
        description: 规格描述
        implementation: 实现要点（如有源码）
        decision: 关键设计决策
```

---

## 工作流程（当前 SKILL.md）

1. 获取分析目标（URL 或目录）
2. 初步分析项目结构和意图
3. 创建存放目录
4. 保存初级分析报告
5. 根据初级报告生成萃取结果

---

## 优先级

1. **高**: 添加边界情况处理
2. **中**: 测试用例创建
3. **低**: 运行测试并迭代优化

---

## 验证记录

| 项目 | 类型 | 萃取内容 | 验证结果 |
|------|------|----------|----------|
| my-secretary | 产品设计文档 | 数据模型、产品概念 | ✅ 适合萃取设计文档 |
| memory | 完整 Python 项目 | Clean Architecture、抽象接口 | ✅ 适合萃取代码项目 |
| plato | 前后端分离项目 | 三层LLM架构、版本控制、前端架构 | ✅ 适合萃取复杂项目 |
