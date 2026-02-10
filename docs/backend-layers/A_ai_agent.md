## AI Agent層（Agentの実装、ツール定義、型定義 等）

### 必須チェック項目：

- **型定義パターンの遵守**: Tool入力スキーマと内部保存モデルを適切に分離しているか
- **Transform関数の配置**: AI入力から内部形式への変換ロジックを適切に実装しているか
- **FunctionCallResultの設計**: Agent別の結果モデルが適切に定義され、`result.message`の役割が明確か
- **additionalDataの管理**: `ChatMessage.data`に保存すべきデータと保存しないデータが適切に区別されているか
- **RunStepによる監査ログ**: Tool実行の入出力が適切に記録されているか
- **空スキーマ定義ファイル不要**: 中身が空のリクエストDTO等は別ファイルに分離せず使用箇所に直接記述する
- **AIツール戻り値の最小化**: ツール結果は`message`とプログラム的に必要な値（ID等）だけ返す
- **検索ツールの寛容な設計**: 重複登録防止目的などの検索ツールは、入力が不完全でも候補を見つけられる設計にする

---

### 1. 型定義パターン

#### 1.1 Tool入力スキーマ（DTO + Validation）

OpenAI Structured Outputでは全フィールドが存在する形式を求める場合があるため、任意項目の意味はnullableフィールドと明示的なバリデーションで表現する。

```java
// 任意項目は nullable で扱う / Treat optional semantics as nullable
public record OrderDraftItemRequest(
    @NotBlank String itemNum,
    @NotNull BigDecimal quantity,
    String packSize,
    @NotNull Uom uom,
    @NotNull Status status,
    @NotNull Confidence confidence
) {}
```

#### 1.2 内部保存モデル（Tool入力とは別）

AI入力と内部保存でモデルを分離し、nullの扱いなどの変換を明確にする。

```java
// AI入力: packSize nullable / AI input: packSize can be null
// 内部保存: packSize 非null / Internal storage: packSize must be non-null
public record OrderEntryPayloadItem(
    String itemNum,
    BigDecimal quantity,
    String packSize,
    Uom uom,
    String status,
    String confidence
) {}

public record OrderEntryPayload(
    List<OrderEntryPayloadItem> items,
    boolean isSaved
) {}
```

#### 1.3 Transform関数

AI入力から内部保存形式への変換、および外部API用への変換を明示的に関数化する。

```java
// AI入力 -> 内部保存 / AI input -> internal storage
public static OrderEntryPayload transformToOrderEntryPayload(SaveOrUpdateOrderDraftParams params) {
    List<OrderEntryPayloadItem> items = params.orderInfo().stream()
        .map(item -> new OrderEntryPayloadItem(
            item.itemNum(),
            item.quantity(),
            item.packSize() == null ? "" : item.packSize(),
            item.uom(),
            item.status(),
            item.confidence()
        ))
        .toList();

    return new OrderEntryPayload(items, false);
}

// 内部保存 -> 外部API（不要項目を除外） / Internal -> external API (remove unnecessary fields)
public static ToSaveOrderRequestParams transformToSaveOrderRequest(OrderEntryPayload payload) {
    List<ToSaveOrderRequestItem> orderInfo = payload.items().stream()
        .map(item -> new ToSaveOrderRequestItem(
            item.itemNum(),
            item.quantity(),
            item.uom(),
            null
        ))
        .toList();

    return new ToSaveOrderRequestParams(orderInfo);
}
```

---

### 2. FunctionCallResultパターン

#### 2.1 Agent別の結果モデル

基本構造: `status` + `result.message` + Agent固有データ

```java
// Sales Agent用 / For Sales Agent
public record FunctionCallResult(
    String status,
    SalesResult result
) {
    public record SalesResult(
        String message,
        Long savedOrderRequestId,
        Long referencedOrderRequestId
    ) {}
}

// OrderEntryWorkflow用 / For OrderEntryWorkflow
public record OrderEntryWorkflowFunctionCallResult(
    String status,
    OrderEntryWorkflowResult result
) {
    public record OrderEntryWorkflowResult(
        String message,
        OutputReport outputReport
    ) {}
}

// Reviewer Agent用 / For Reviewer Agent
public record ReviewerAgentFunctionCallResult(
    String status,
    ReviewerAgentResult result
) {
    public record ReviewerAgentResult(
        String message,
        ReviewerAgentOutput reviewerAgentOutput
    ) {}
}
```

#### 2.2 `result.message`の役割

`result.message`はモデルが次ターンで参照する情報であり、UIには直接表示しない。

```java
// OpenAIに返却 / Return to OpenAI (function_call_output)
messages.add(Map.of(
    "type", "function_call_output",
    "call_id", callId,
    "output", objectMapper.writeValueAsString(result)
));
```

---

### 3. additionalDataパターン

#### 3.1 `ChatMessage.data`の構造

フロントからの入力と、バックエンド処理で追加されるデータを明確に分離する。

```java
// Input（フロントから受け取る） / Input (from frontend)
public class ChatMessageAdditionalDataInput {
    private Long customerAccountNumber;
    private String customerAccountName;
    private Long branchOrgId;
    private List<UploadedResult> attachments;
    private DomSnapshotInput domSnapshot;
}

// 保存時（バックエンド処理で追加） / Stored (added by backend)
public class ChatMessageAdditionalData extends ChatMessageAdditionalDataInput {
    private Long savedOrderRequestId;
    private Long referencedOrderRequestId;
    private OutputReport outputReport;
    private List<ResponseAnnotation> openaiFileAnnotations;
}
```

#### 3.2 additionalDataに何を保存するか

| データ                   | 保存 | 理由                           |
| ------------------------ | ---- | ------------------------------ |
| savedOrderRequestId      | OK   | UI表示・次ターンのcontext注入  |
| referencedOrderRequestId | OK   | UI表示・次ターンのcontext注入  |
| outputReport             | OK   | UI表示（サマリーカード）       |
| attachments              | OK   | ファイル参照                   |
| Tool実行の途中結果       | NG   | RunStepに記録                  |
| workingMemory            | NG   | ChatTransactionに保存          |

#### 3.3 `additionalData` -> context注入

```java
public String buildContentFromAdditionalData(ChatMessage chatMessage, Long userId) {
    ChatMessageAdditionalData additionalData = chatMessage.getData();
    if (additionalData == null) {
        return chatMessage.getText();
    }

    String context = "\n\n";
    return chatMessage.getText() + "\n\n" + context;
}
```

---

### 4. Tool実行 -> additionalData保存フロー

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. Tool実行                                                      │
├─────────────────────────────────────────────────────────────────┤
│ FunctionCallResult result = executeRequestOrder(...);            │
│ // result = { status: "success",                                │
│ //            result: { message: "...", savedOrderRequestId: 5 }}│
└─────────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ 2. OpenAIに返す                                                  │
├─────────────────────────────────────────────────────────────────┤
│ messages.add({                                                   │
│   "type": "function_call_output",                              │
│   "call_id": callId,                                            │
│   "output": objectMapper.writeValueAsString(result),            │
│ });                                                               │
└─────────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ 3. UI表示用に一時保持                                            │
├─────────────────────────────────────────────────────────────────┤
│ if (result.result().savedOrderRequestId() != null) {             │
│   savedOrderRequestId = result.result().savedOrderRequestId();    │
│ }                                                                 │
└─────────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ 4. 次のChatMessage保存時に書き込む                               │
├─────────────────────────────────────────────────────────────────┤
│ parseMessageAgentService.run(                                    │
│   conversation, chatTx, responseText, responseId,                │
│   Map.of("savedOrderRequestId", savedOrderRequestId,            │
│          "referencedOrderRequestId", referencedOrderRequestId)   │
│ );                                                                │
│ savedOrderRequestId = null; // リセット                          │
└─────────────────────────────────────────────────────────────────┘
```

**Point**: Tool実行結果は次のassistantメッセージの`additionalData`に保存する。

---

### 5. RunStep（監査ログ）

#### 5.1 記録タイミング

Tool実行の前後で記録する。

```java
// 実行前後を記録 / Record before and after execution
RunStep runStep = runStepRepository.save(new RunStep(userMessage.getChatTransactionId(), toolName));
runStep.setInput(toolArgsJson);

FunctionCallResult result = executeFunctionCall(...);

runStep.setOutput(objectMapper.writeValueAsString(result));
runStepRepository.save(runStep);
```

#### 5.2 記録内容

```java
@Entity
public class RunStep {
    @Column(nullable = false)
    private String toolName;

    @Column(columnDefinition = "jsonb")
    private String input; // Tool引数JSON（生 or 正規化後）

    @Column(columnDefinition = "jsonb")
    private String output; // FunctionCallResult JSON
}
```

**Point**: RunStepはデバッグ・監査用。UIには表示しない。

---

### 6. `parseAndValidateToolArgs`ヘルパー

ObjectMapper + Bean Validationで、JSON文字列のパースとバリデーションを統一的に行う。

```java
public static <T> ParseResult<T> parseAndValidateToolArgs(
    String toolArgsJson,
    Class<T> targetClass,
    String toolName,
    ObjectMapper objectMapper,
    Validator validator
) {
    try {
        T parsed = objectMapper.readValue(toolArgsJson, targetClass);
        Set<ConstraintViolation<T>> violations = validator.validate(parsed);
        if (!violations.isEmpty()) {
            return ParseResult.error("Invalid parameters for " + toolName + "...");
        }
        return ParseResult.success(parsed);
    } catch (Exception e) {
        return ParseResult.error("Invalid JSON for " + toolName + "...");
    }
}
```

**使用例:**

```java
case "save_order_draft" -> {
    ParseResult<SaveOrUpdateOrderDraftParams> result = parseAndValidateToolArgs(
        toolArgsJson,
        SaveOrUpdateOrderDraftParams.class,
        "save order draft",
        objectMapper,
        validator
    );
    if (!result.success()) {
        return new FunctionCallResult("error", new FunctionCallResult.SalesResult(result.error(), null, null));
    }
    SaveOrUpdateOrderDraftParams params = result.data();
    // ...
}
```

---

### 7. データフローまとめ

```
┌──────────────────────────────────────────────────────────────────────┐
│ Tool入力                                                              │
│ (DTO + Bean Validation, 必要に応じてnullable許容)                      │
└───────────────────────────────────┬──────────────────────────────────┘
                                    ↓ parseAndValidateToolArgs
┌───────────────────────────────────┴──────────────────────────────────┐
│ 内部処理                                                              │
│ (Transform関数でnull -> 空文字、不要フィールド削除)                    │
└───────────────────────────────────┬──────────────────────────────────┘
                                    ↓
        ┌───────────────────────────┼───────────────────────────┐
        ↓                           ↓                           ↓
┌───────────────┐         ┌─────────────────┐         ┌─────────────────┐
│ RunStep       │         │ Entity更新       │         │ FunctionCall    │
│ (監査ログ)    │         │ (OrderRequest等) │         │ Result          │
└───────────────┘         └─────────────────┘         └────────┬────────┘
                                                               ↓
                                                  ┌────────────┴────────────┐
                                                  ↓                         ↓
                                        ┌─────────────────┐       ┌─────────────────┐
                                        │ OpenAIに返す    │       │ UI表示用に保持   │
                                        │ (message)       │       │ (xxxId, report) │
                                        └─────────────────┘       └────────┬────────┘
                                                                           ↓
                                                              ┌────────────────────────┐
                                                              │ ChatMessage.data       │
                                                              │ (additionalData)       │
                                                              └────────────────────────┘
```

---

### この層で扱わないこと：

- Entity定義（DB層の責務）
- APIエンドポイント定義（Controller層の責務）
- ビジネスロジック（Service層の責務）
