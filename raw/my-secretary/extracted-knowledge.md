# 萃取的精华

## my-secretary 系统设计

**description**: 个人秘书系统 - 管理联系人和记录与联系人的互动事件

**purpose**: 解决工作和生活中需要与各种人（同事、朋友、家人）保持联系和记录互动的需求，方便日后查询和总结

---

### 模块：联系人管理 (contacts)

**description**: 存储和管理联系人的基本信息

**responsibilities**:
- 维护联系人基本资料
- 支持分类管理（work/friend/family）
- 处理姓名唯一性（自动加后缀）
- 跟踪联系人状态（驻场、离职）

**features**:

- **Feature: 联系人分类管理**
  - **Spec: 分类枚举**
    - description: 支持 work/friend/family 三种分类
    - decision: 分类简洁，覆盖主要场景
  - **Spec: 昵称支持**
    - description: 支持多个昵称，用逗号分隔
    - decision: 用逗号分隔简单直接，避免复杂关联表

- **Feature: 姓名唯一性处理**
  - **Spec: 自动加后缀**
    - description: 当姓名重复时自动加后缀区分
    - decision: 避免强制唯一导致用户必须改名的困扰

- **Feature: 联系人状态跟踪**
  - **Spec: 离职状态**
    - description: 支持记录是否离职及离职日期
    - decision: 对于同事类联系人，离职状态很重要

**interfaces**:

- **联系人搜索**
  - input: 姓名或昵称关键词
  - output: 匹配的联系人列表
  - usage: 快速找到特定联系人

**tech_stack**:
- SQLite
- 字段包括：name, category, company, position, phone, email, nickname, contract_entity, dept_level1, dept_level2, entry_date, is_onsite, has_left, left_date, created_at, updated_at

---

### 模块：事件记录 (events)

**description**: 记录与联系人的各种互动事件

**responsibilities**:
- 记录互动事件（类型、主题、内容、时间）
- 支持多联系人关联
- 支持按类型筛选和时间排序

**features**:

- **Feature: 事件类型支持**
  - **Spec: 多种交互渠道**
    - description: 支持 email/chat/phone/meeting/微信/钉钉/线下等类型
    - decision: 覆盖主要沟通渠道

- **Feature: 多联系人关联**
  - **Spec: 逗号分隔存储**
    - description: contacts 字段用逗号分隔多个联系人
    - decision: 简化查询，不用关联表

**interfaces**:

- **事件查询**
  - input: 联系人姓名（模糊）、事件类型（过滤）
  - output: 事件列表（按时间倒序）
  - usage: 查看与某人的历史互动

- **事件统计**
  - input: 分类维度
  - output: 统计结果
  - usage: 了解联系和互动频率

**tech_stack**:
- SQLite
- 字段包括：id, contacts, type, subject, content, occurred_at, created_at

---

## 关键设计决策

1. **姓名字段唯一性策略**: 不强制数据库唯一，而是在应用层自动加后缀处理冲突
2. **多联系人存储**: 用逗号分隔而不是关联表，简化查询但增加复杂度
3. **SQLite 作为存储**: 轻量、零配置、适合本地小规模数据

## 可复用的提示词模板

```
创建一个个人联系人+事件记录系统，包含：
- contacts 表：姓名、分类（work/friend/family）、公司、职位、联系方式、昵称、部门、状态（驻场/离职）
- events 表：关联联系人（多对多，用逗号分隔）、事件类型（email/chat/phone/meeting等）、主题、内容、时间
- 功能：分类管理、昵称搜索、事件记录、统计、联系人状态跟踪
```
