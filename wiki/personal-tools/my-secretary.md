# my-secretary

> Sources: my-secretary project, 2024
> Raw: [my-secretary extracted-knowledge](../../raw/my-secretary/extracted-knowledge.md); [my-secretary primary-report](../../raw/my-secretary/primary-report.md)

## Overview

my-secretary 是一个**个人秘书系统**，用于管理联系人和记录与联系人的互动事件。核心特点是**联系人 + 事件数据模型**、**姓名字段唯一性策略**（应用层自动加后缀）、**多联系人存储**（逗号分隔简化查询）、以及**SQLite 轻量存储**。

## 系统设计

### 模块：联系人管理 (contacts)

**数据模型**：
- 核心字段：name, category (work/friend/family), company, position, phone, email, nickname
- 组织字段：dept_level1, dept_level2（部门层级）
- 状态字段：is_onsite, has_left, left_date（驻场/离职状态）

**Feature: 联系人分类管理**
- 分类枚举：work/friend/family 三种分类
- 昵称支持：多个昵称用逗号分隔

**Feature: 姓名唯一性处理**
- 自动加后缀：当姓名重复时自动加后缀区分
- 避免强制唯一导致用户必须改名的困扰

**Feature: 联系人状态跟踪**
- 支持记录是否离职及离职日期
- 对于同事类联系人，离职状态很重要

### 模块：事件记录 (events)

**数据模型**：
- 核心字段：contacts, type, subject, content, occurred_at
- type 支持：email/chat/phone/meeting/微信/钉钉/线下等

**Feature: 多联系人关联**
- contacts 字段用逗号分隔多个联系人
- 简化查询，不用关联表

**接口**：
- 事件查询：按联系人姓名（模糊）、事件类型（过滤）
- 事件统计：按分类维度统计

## 关键设计决策

### 1. 姓名字段唯一性策略

不强制数据库唯一，而是在应用层自动加后缀处理冲突，避免强制唯一导致用户必须改名的困扰。

### 2. 多联系人存储

用逗号分隔而不是关联表，简化查询但增加复杂度，适合轻量级应用。

### 3. SQLite 作为存储

轻量、零配置、适合本地小规模数据。

## 可复用的提示词模板

```
创建一个个人联系人+事件记录系统，包含：
- contacts 表：姓名、分类（work/friend/family）、公司、职位、联系方式、昵称、部门、状态（驻场/离职）
- events 表：关联联系人（多对多，用逗号分隔）、事件类型（email/chat/phone/meeting等）、主题、内容、时间
- 功能：分类管理、昵称搜索、事件记录、统计、联系人状态跟踪
```
