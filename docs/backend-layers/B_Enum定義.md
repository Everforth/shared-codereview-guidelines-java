## Enum定義・運用ガイドライン

### 1. 基本方針

- **DB enumは削除しない**: 過去レコードの保持を優先する
- **仕様から外れた値は「deprecated」として扱う**: コード上で明示的に分離する
- **コードの可読性・型安全性を守る**: 現役仕様が一目で分かる状態を保つ

---

### 2. 二層定義パターン

#### 目的

- コードを見れば「現役仕様」が一目で分かる状態を保つ
- DBに残るlegacy値を仕様から隔離する

#### ルール

- **全集合（legacy含む）** と **現役仕様** を分けて定義する
- アプリの業務ロジック・DTOは **現役仕様のみ** を使う

#### 実装例

```java
// ============================================
// 全集合（DBに残るlegacy値を含む）
// ============================================
public enum ConversationAgentAll {
  SALES_AGENT("salesAgent"),
  ORDER_ENTRY_WORKFLOW("orderEntryWorkflow"),
  ANALYZE_AGENT("analyzeAgent"),
  MARKET_AGENT("marketAgent"),
  // deprecated
  LEGACY_AGENT("legacyAgent");

  private final String dbValue;

  ConversationAgentAll(String dbValue) {
    this.dbValue = dbValue;
  }

  public String getDbValue() {
    return dbValue;
  }
}

// ============================================
// 現役仕様（新規に使ってよい集合）
// ============================================
public final class ConversationAgent {
  private ConversationAgent() {}

  public static final java.util.Set<ConversationAgentAll> ACTIVE =
      java.util.EnumSet.of(
          ConversationAgentAll.SALES_AGENT,
          ConversationAgentAll.ORDER_ENTRY_WORKFLOW,
          ConversationAgentAll.ANALYZE_AGENT,
          ConversationAgentAll.MARKET_AGENT
      );
}
```

#### 使い分け

| 用途                        | 使用する型/構造                           |
| --------------------------- | ----------------------------------------- |
| Entity定義（DBカラムの型）  | `ConversationAgentAll`                     |
| DTO・バリデーション         | 現役仕様を表す構造（`ConversationAgent`） |
| Service層のビジネスロジック | 現役仕様を表す構造（`ConversationAgent`） |
| DBからの読み取り結果の型    | `ConversationAgentAll`                     |

---

### 3. DB / Migration ルール

#### 基本原則

- **enumの追加・リネームは手書きmigration**で行う
- **ORMの自動生成migrationでenumを変更しない**
- **DB enum型へのマッピングは必ず固定**する（Entity/migration側で明示）
- **deprecated値はDB enumに残したまま運用**する

#### 手書きMigrationの書き方

##### 新しいenum値を追加する場合

```sql
-- 新しいenum値を追加
ALTER TYPE conversation_agent
ADD VALUE IF NOT EXISTS 'marketAgentV2';
```

##### enum値をリネームする場合

```sql
-- enum値をリネーム
ALTER TYPE conversation_agent
RENAME VALUE 'analyzeAgent' TO 'analysisAgent';
```

##### 新しいenum型を作成する場合

```sql
-- 新しいenum型を作成
CREATE TYPE order_status AS ENUM ('draft', 'submitted', 'confirmed', 'cancelled');
```

#### JPA/Hibernate Entity側の設定

```java
@Entity
public class Conversation {
  @Enumerated(EnumType.STRING)
  @Column(name = "agent", nullable = false, columnDefinition = "conversation_agent")
  private ConversationAgentAll agent = ConversationAgentAll.SALES_AGENT;
}
```

#### 禁止事項

- ORMのスキーマ自動更新（例: `spring.jpa.hibernate.ddl-auto=update`）でのenum自動変更
- ORMが生成するmigrationでのenum変更（手書きで上書きすること）
- DB enumからの値削除（過去データが参照できなくなる）
