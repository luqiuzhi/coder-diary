# Flyway结合Jenkins实现数据库变更自动化

## 一、集成Flyway核心步骤
1. **添加依赖**  
   在`pom.xml`中增加Flyway依赖（推荐与Spring Boot版本匹配）：
```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
    <version>10.0.1</version> <!-- 根据Spring Boot版本选择 -->
</dependency>
```
**注意**：Spring Boot 3.x+需要Flyway 9.0+版本，且PostgreSQL驱动需兼容。

2. **配置参数**  
   在`application.yml`中配置关键参数：
```yaml
spring:
  flyway:
    enabled: true
    baseline-on-migrate: true  # 非空数据库首次集成时必备
    baseline-version: 1        # 基线版本号（根据当前数据库状态设置）
    locations: classpath:db/migration
    validate-on-migrate: true  # 校验脚本是否被篡改
    clean-disabled: true       # 生产环境必须禁用clean操作
  datasource:
    url: jdbc:postgresql://${DB_HOST}:${DB_PORT}/${DB_NAME}
    username: ${DB_USER}
    password: ${DB_PASSWORD}
```

3. **迁移脚本规范**  
   在`src/main/resources/db/migration`目录下按以下规则创建SQL文件：
- 版本化脚本：`V{版本号}__{描述}.sql`（如`V1.2__add_user_table.sql`）
- 可重复脚本：`R__{描述}.sql`（每次修改后重新执行）

## 二、与MyBatis整合的要点
1. **执行顺序控制**  
   确保Flyway在MyBatis初始化前完成迁移：
```java
@Configuration
public class FlywayConfig {
    @Bean
    public FlywayMigrationInitializer flywayInitializer(Flyway flyway) {
        return new FlywayMigrationInitializer(flyway);
    }
    
    @Bean
    @DependsOn("flywayInitializer") // MyBatis依赖此初始化器
    public SqlSessionFactory sqlSessionFactory(...) { 
        // MyBatis初始化逻辑
    }
}
```
通过`@DependsOn`确保数据库结构就绪后才初始化MyBatis。

2. **现有数据库处理**  
   若数据库已有表结构：
- 首次集成时设置`baseline-version`为当前最高版本号+1
- 将现有表结构导出为`V{基线版本}__baseline.sql`文件
- 后续变更使用更高版本号的迁移脚本

## 三、Jenkins自动化集成
1. **构建配置**  
   在Jenkinsfile中添加Flyway验证阶段：
```groovy
stage('Flyway Migration') {
    steps {
        sh './mvnw flyway:migrate -Dflyway.configFiles=src/main/resources/application.yml'
    }
}
```

2. **多环境策略**  
   使用Spring Profiles区分环境配置：
```yaml
# application-prod.yml
spring:
  flyway:
    locations: classpath:db/migration/prod
    placeholders:
      schema: prod_schema

# application-dev.yml  
spring:
  flyway:
    locations: classpath:db/migration/dev
```

## 四、最佳实践建议
1. **版本控制规范**
    - 每个发布版本对应一个迁移脚本包（如`V2.3.0__release_featureX.sql`）
    - 禁止修改已提交的脚本文件（checksum校验机制会阻止启动）

2. **生产环境防护**
    - 启用`validate-on-migrate`防止意外修改
    - 通过`flyway.repair`修复校验失败问题
    - 使用`flyway.info`验证迁移状态

3. **回滚策略**  
   虽然Flyway原生不支持回滚，可通过以下方式实现：
```sql
-- 创建回滚脚本 V2.4__rollback_featureX.sql
ALTER TABLE users DROP COLUMN temp_column;
```

## 五、监控与审计
1. 通过`flyway_schema_history`表监控执行记录：
```sql
SELECT version, script, installed_on 
FROM flyway_schema_history 
ORDER BY installed_rank DESC;
```

2. 集成Prometheus监控：
```java
@Bean
public FlywayMigrationMetrics flywayMetrics(Flyway flyway) {
    return new FlywayMigrationMetrics(flyway);
}
```

## 六、注意事项
1. **测试策略**  
   使用Testcontainers进行集成测试：
```java
@Testcontainers
@SpringBootTest
class MigrationTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine");
    
    @Test
    void testMigration() {
        Flyway flyway = Flyway.configure()
            .dataSource(postgres.getJdbcUrl(), postgres.getUsername(), postgres.getPassword())
            .load();
        MigrationInfoService info = flyway.info();
        assertThat(info.applied().length).isPositive();
    }
}
```

2. **版本冲突处理**  
当多人协作时，建议采用语义化版本控制：
- 主版本号：重大架构变更
- 次版本号：功能迭代
- 修订号：紧急修复

通过以上方案，可实现数据库变更的版本化管理和自动化部署，减少人为操作风险。建议在预发布环境充分测试迁移脚本后再部署到生产环境。

