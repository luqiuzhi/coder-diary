# 关于AutoDDL工具的选型和引入

## 一、现行管理策略

目前主要是在项目的`/sql`使用手动维护DDL文件。以月份或功能对需要执行的数据库脚本进行分类。

在进行测试、发布时需要开发人员自行区分已执行和未执行的脚本，并手动复制到数据库控制台进行执行。

该方式使得同一份脚本可能需要**人工多次执行**，在发布时，也可能存在**遗漏、重复执行**的情况。

## 二、AutoDDL工具的选型

目前常用的数据库控制变更工具有`Flyway`和`Liquibase`，由于项目本身依赖`MyBatis-Plus`，而该框架在`3.5.3+`
版本后提供了自动维护DDL的功能，因此一同纳入选型范围。

### 核心定位差异

1. **MyBatis-Plus DDL自动维护**
    - **定位**：作为ORM框架的扩展功能，**内嵌于Java应用**，通过代码驱动SQL脚本执行，实现表结构动态维护（如创建表、存储过程等）。
    - **特点**：
        - 支持分库分表场景下的动态数据源切换。
        - 自动记录执行历史到`ddl_history`表，但缺乏版本回滚能力。
        - 适合需要**与业务代码深度结合**的简单DDL管理场景（如开发环境快速迭代）。

2. **Flyway/Liquibase**
    - **定位**：**独立数据库迁移工具**，专注于全生命周期的数据库版本管理（创建、升级、回滚），与语言/框架解耦。
    - **特点**：
        - Flyway以纯SQL脚本为核心，Liquibase支持SQL/XML/YAML/JSON多格式，适配跨数据库兼容需求。
        - 提供版本历史追踪、校验、回滚等企业级功能，适合生产环境严格变更管理。

---

### 功能特性对比

| **维度**     | **MyBatis-Plus DDL** | **Flyway**                    | **Liquibase**          |
|------------|----------------------|-------------------------------|------------------------|
| **脚本管理**   | SQL文件（支持存储过程）        | SQL文件（强制版本命名，如`V1__init.sql`） | XML/YAML/JSON（声明式变更描述） |
| **版本控制**   | 仅记录执行历史，无版本顺序控制      | 严格版本顺序，支持基线（Baseline）         | 变更集（ChangeSet）顺序管理     |
| **回滚支持**   | 无                    | 商业版支持回滚                       | 开源版支持回滚                |
| **多数据源支持** | 动态切换数据源执行脚本          | 需独立配置多数据源                     | 支持多数据源，需复杂配置           |
| **执行时机**   | 应用启动时自动执行            | 应用启动前独立执行                     | 独立执行或集成到应用启动流程         |

---

## 三、集成Mybatis-Plus的自动维护DDL功能

根据当下情况的分析，使用`MyBatis-Plus`自带的框架功能<a href="https://baomidou.com/guides/auto-ddl/#_top">自动维护DDL</a>对代码改动最小，并且团队学习成本低，极易上手。

> 若存在复杂的需求，也可考虑使用Flyway。在使用`.sql`类型脚本的情况下极易切换。

### 1、集成方式

1. 升级`MyBatis-Plus`的版本至`3.5.3.2`（该版本同时支持存储过程）
2. 根据`MyBatis-Plus`提供的接口，实现自动读取脚本文件。各个品牌的脚本路径由`spring.prefix`参数进行控制，该参数默认为`dev`。

```java
    @Override
    public List<String> getSqlFiles() {
        ResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
        Resource[] resources;
        try {
            resources = resolver.getResources(StrUtil.format(SQL_FILE_PATH_RESOURCE, property.getPrefixKey()));
            List<String> fileNames = new ArrayList<>();
            String resourcePath = StrUtil.format(SQL_FILE_PATH, property.getPrefixKey());
            for (Resource resource : resources) {
                if (resource.isReadable()) {
                    fileNames.add(resourcePath + resource.getFilename());
                }
            }
            return fileNames;
        } catch (IOException e) {
            log.error("获取sql文件失败，请手动执行SQL脚本！", e);
        }
        return Collections.emptyList();
    }
```

3. 按照规范编写脚本文件，存放在`back-stage/src/main/resources/db/品牌名/migration`目录下。
4. 初次执行DDL脚本时，会在数据库中自动建立历史表：`ddl_history`。该表结构如下：

| script         | type | version      |
|----------------|------|--------------|
| db/v1-user.sql | sql  | 202503121740 |
| db/v2-user.sql | sql  | 202503121742 |

其中`script`是脚本相对于classpath的相对路径+文件名，作为唯一标识。`version`字段表示了脚本的执行时间。

### 2、集成规范

#### 脚本目录说明

`db/xx/migration/`下主要存放数据库变更脚本，用以替代原来的`/sql`文件夹。

该目录下的sql脚本由`mybatis-plus`的自动DDL功能执行。具体实现可以参考类`MybatisPlusAutoDdl`。

其中xx代表不同品牌子目录（目前分支隔离后不会有多个品牌的脚本在一个分支下）。

#### 脚本命名和编写规范

##### 命名规范

`mybatis-plus`根据`sql`脚本的名称（相对于classpath的路径，如`db/gaga/migration/202503181801_task20_alter_table.sql`
）来执行脚本，因此脚本名称需要具有唯一性，否则`mybatis-plus`会忽略相关的脚本。

脚本名称的格式为：`yyyyMMddHHmm_task10_alter_table.sql`，形如`202503181801_task20_alter_table.sql`，脚本名称分为三部分。

- 第一部分为时间版本号，以年月日时分的形式，避免多人协作时产生相同的版本；
- 第二部分为禅道任务编号，用于追溯当前脚本和任务的关联性，如task20，bug22等，尽可能让所有脚本归属于某个任务，若确实没有关联任务，该部分可省略但是其后的脚本描述不可省略；
- 第三部分为脚本的简单描述；

##### 编写规范

- 目前仅支持sql格式的脚本。

> 从`3.5.3.2`版本开始，支持执行存储过程。在文件名后追加`#$$`，其中`$$`是自定义的完整SQL分隔符。
> 存储过程脚本以`END`结尾，并追加分隔符`END;$$`表示脚本结束。
>
> 但是考虑到不同品牌的数据库兼容性，存储过程几乎用不到。

- 历史脚本的修改，请不要直接修改，而是以`yyyyMMddHHmm_task10_alter_table.sql`的形式创建新的脚本，以补丁的形式执行变更。

- 脚本中不要包含任何业务逻辑，如`if`、`for`、`while`等。

- 针对大数据量的初始化脚本（脚本大小超过100MB），慎用该功能。因为应用在启动时会读取脚本文件执行，文件过大可能导致启动失败。
