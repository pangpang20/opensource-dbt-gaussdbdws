# dbt-gaussdbdws介绍

## 开源地址

在github的开源地址 [dbt-gaussdbdws](https://github.com/pangpang20/dbt-gaussdbdws.git)

项目已经编译提交到`pypi`, 支持以下方式安装

```bash
pip install dbt-gaussdbdws
```

## 什么是dbt-gaussdbdws?

dbt-gaussdbdws 是一个专为 华为 GaussDB 和 GaussDB（DWS） 数据库设计的扩展包，提供了使 dbt-core 高效适配华为数据库的全部代码支持。它通过标准化和模块化的方式，帮助用户更轻松地完成数据建模、转换与管理任务，满足现代数据分析的高效性和可扩展性需求。

## 主要功能

使用 dbt-gaussdbdws，用户能够：

### 便捷的数据建模

通过配置文件快速定义表结构与数据模型，自动生成 SQL 脚本，无需手动编写复杂的代码。

### 高效的数据转换

支持 ELT（Extract-Load-Transform）流程中的数据清洗、去标准化、过滤、重命名和预聚合操作，使原始数据转换为适合分析的数据。

### 无缝的数据库集成

深度适配 华为 GaussDB 和 GaussDB（DWS），充分发挥数据库的分布式计算能力和性能优势。

### 与开发者工具链集成

遵循与软件工程类似的最佳实践，例如版本控制、模块化设计和 CI/CD 集成，使数据团队能够更快速、更安全地交付数据产品。

### 灵活的团队协作

通过定义清晰的数据依赖关系，提升跨团队协作的效率，减少数据开发过程中的重复劳动和错误。
