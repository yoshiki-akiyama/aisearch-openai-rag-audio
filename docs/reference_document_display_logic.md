# RAG参照ドキュメント（Reference Document）表示ロジックの説明

このドキュメントでは、VoiceRAGアプリケーションにおいて、RAGで利用した参照ドキュメント（reference）を表示するためのロジックフローを詳細に説明します。

## 概要

RAG（Retrieval Augmented Generation）システムでは、AIが回答を生成する際に参照したドキュメントソースを表示することが重要です。このアプリケーションでは、以下のフローで参照ドキュメントを表示しています：

1. バックエンドでAIが参照したソースを特定
2. Azure AI Searchからソースの詳細情報を取得
3. フロントエンドでソース情報を受信し表示

## アーキテクチャフロー

```
[AIがsearchツールで検索]
      ↓
[AIがreport_groundingツールを呼び出し]
      ↓
[_report_grounding_tool関数が実行] (ragtools.py)
      ↓
[Azure AI Searchから参照ドキュメントの詳細を取得]
      ↓
[ToolResultとしてフロントエンドに送信]
      ↓
[App.tsxで受信・解析] (onReceivedExtensionMiddleTierToolResponse)
      ↓
[GroundingFilesコンポーネントで表示]
```

## バックエンドロジック（app/backend/ragtools.py）

### 1. report_groundingツールのスキーマ定義

```python
_grounding_tool_schema = {
    "type": "function",
    "name": "report_grounding",
    "description": "Report use of a source from the knowledge base as part of an answer...",
    "parameters": {
        "type": "object",
        "properties": {
            "sources": {
                "type": "array",
                "items": {"type": "string"},
                "description": "List of source names from last statement actually used..."
            }
        },
        "required": ["sources"]
    }
}
```

このスキーマは、AIモデルに対して「回答に使用したソースを報告する」ツールが利用可能であることを伝えます。

### 2. _report_grounding_tool関数の処理フロー

**入力パラメータ：**
- `search_client`: Azure AI Search クライアント
- `identifier_field`: チャンクIDフィールド名（デフォルト: "chunk_id"）
- `title_field`: タイトルフィールド名（デフォルト: "title"）
- `content_field`: コンテンツフィールド名（デフォルト: "chunk"）
- `args`: AIから渡される引数（sourcesリストを含む）

**処理ステップ：**

```python
async def _report_grounding_tool(search_client: SearchClient, identifier_field: str, title_field: str, content_field: str, args: Any) -> None:
    # ステップ1: ソースIDをフィルタリング（セキュリティチェック）
    sources = [s for s in args["sources"] if KEY_PATTERN.match(s)]
    list = " OR ".join(sources)
    print(f"Grounding source: {list}")
    
    # ステップ2: Azure AI Searchでソースの詳細情報を検索
    search_results = await search_client.search(
        search_text=list, 
        search_fields=[identifier_field], 
        select=[identifier_field, title_field, content_field], 
        top=len(sources), 
        query_type="full"
    )
    
    # ステップ3: 結果をドキュメント形式に変換
    docs = []
    async for r in search_results:
        docs.append({
            "chunk_id": r[identifier_field], 
            "title": r[title_field], 
            "chunk": r[content_field]
        })
    
    # ステップ4: クライアント（フロントエンド）に結果を返す
    return ToolResult({"sources": docs}, ToolResultDirection.TO_CLIENT)
```

**重要なポイント：**

1. **セキュリティチェック**: `KEY_PATTERN.match(s)` により、不正な文字列がソースIDとして渡されないようにしています
2. **検索方式**: `query_type="full"` を使用し、キーワード検索でソースを特定します
3. **データ構造**: 各ドキュメントは `chunk_id`、`title`、`chunk`（コンテンツ）の3つのフィールドを持ちます
4. **送信方向**: `ToolResultDirection.TO_CLIENT` により、結果がフロントエンドに直接送信されます

### 3. Azure AI Searchの検索設定

環境変数で設定可能な検索パラメータ：

- `AZURE_SEARCH_IDENTIFIER_FIELD`: チャンクIDフィールド（デフォルト: "chunk_id"）
- `AZURE_SEARCH_TITLE_FIELD`: タイトルフィールド（デフォルト: "title"）
- `AZURE_SEARCH_CONTENT_FIELD`: コンテンツフィールド（デフォルト: "chunk"）

これらは `app/backend/app.py` の `attach_rag_tools` 呼び出しで設定されます。

## フロントエンドロジック

### 1. データ型定義（app/frontend/src/types.ts）

```typescript
export type GroundingFile = {
    id: string;        // chunk_id
    name: string;      // title
    content: string;   // chunk
};

export type ToolResult = {
    sources: { chunk_id: string; title: string; chunk: string }[];
};
```

### 2. App.tsx での受信・変換処理

```typescript
onReceivedExtensionMiddleTierToolResponse: message => {
    // ステップ1: JSON文字列をToolResultオブジェクトに解析
    const result: ToolResult = JSON.parse(message.tool_result);

    // ステップ2: バックエンドのデータ構造をフロントエンド用に変換
    const files: GroundingFile[] = result.sources.map(x => {
        return { 
            id: x.chunk_id,      // chunk_id → id
            name: x.title,       // title → name
            content: x.chunk     // chunk → content
        };
    });

    // ステップ3: 既存のファイルに新しいファイルを追加
    setGroundingFiles(prev => [...prev, ...files]);
}
```

**重要なポイント：**

1. **データマッピング**: バックエンドの `chunk_id`, `title`, `chunk` をフロントエンドの `id`, `name`, `content` に変換
2. **累積表示**: `[...prev, ...files]` により、会話中に参照されたすべてのドキュメントを保持
3. **リアルタイム更新**: AIが新しいソースを参照するたびに、UIが自動的に更新される

### 3. 表示コンポーネント

#### GroundingFiles コンポーネント (app/frontend/src/components/ui/grounding-files.tsx)

このコンポーネントは参照ドキュメントのリストを表示します：

```tsx
export function GroundingFiles({ files, onSelected }: Properties) {
    // 参照ドキュメントがない場合は何も表示しない
    if (files.length === 0) {
        return null;
    }

    return (
        <Card className="m-4 max-w-full md:max-w-md lg:min-w-96 lg:max-w-2xl">
            <CardHeader>
                <CardTitle>{t("groundingFiles.title")}</CardTitle>
                <CardDescription>{t("groundingFiles.description")}</CardDescription>
            </CardHeader>
            <CardContent>
                {/* アニメーション付きでファイルを表示 */}
                <div className="flex flex-wrap gap-2">
                    {files.map((file, index) => (
                        <GroundingFile 
                            key={index} 
                            value={file} 
                            onClick={() => onSelected(file)} 
                        />
                    ))}
                </div>
            </CardContent>
        </Card>
    );
}
```

**表示の特徴：**

1. **カードUI**: Material Design風のカードで見やすく表示
2. **アニメーション**: Framer Motionを使用し、ファイルが追加される際にスムーズにアニメーション
3. **クリック可能**: 各ファイルをクリックすると詳細が表示される

#### GroundingFile コンポーネント (app/frontend/src/components/ui/grounding-file.tsx)

個別の参照ドキュメントボタンを表示：

```tsx
export default function GroundingFile({ value, onClick }: Properties) {
    return (
        <Button variant="outline" size="sm" className="rounded-full" onClick={onClick}>
            <File className="mr-2 h-4 w-4" />
            {value.name}  {/* ドキュメントのタイトルを表示 */}
        </Button>
    );
}
```

#### GroundingFileView コンポーネント (app/frontend/src/components/ui/grounding-file-view.tsx)

クリックしたドキュメントの全文を表示するモーダル：

```tsx
export default function GroundingFileView({ groundingFile, onClosed }: Properties) {
    return (
        <AnimatePresence>
            {groundingFile && (
                <motion.div className="fixed inset-0 z-50 flex items-center justify-center bg-black bg-opacity-50">
                    <motion.div className="flex max-h-[90vh] w-full max-w-2xl flex-col rounded-lg bg-white p-6">
                        <div className="mb-4 flex items-center justify-between">
                            <h2 className="text-xl font-bold">{groundingFile.name}</h2>
                            <Button onClick={() => onClosed()}>
                                <X className="h-5 w-5" />
                            </Button>
                        </div>
                        <div className="flex-grow overflow-hidden">
                            <pre className="h-[40vh] overflow-auto text-wrap rounded-md bg-gray-100 p-4 text-sm">
                                <code>{groundingFile.content}</code>  {/* 全文を表示 */}
                            </pre>
                        </div>
                    </motion.div>
                </motion.div>
            )}
        </AnimatePresence>
    );
}
```

## データフロー全体像

### 1. 検索フェーズ

```
ユーザーの質問
    ↓
[AIがsearchツールを実行]
    ↓
_search_tool() が Azure AI Searchで検索
    ↓
結果を "[chunk_id]: content" 形式でAIに返す
```

### 2. Grounding（参照報告）フェーズ

```
AIが回答生成時に使用したsourcesを特定
    ↓
[AIがreport_groundingツールを実行]
    ↓
_report_grounding_tool() が詳細情報を取得
    ↓
{sources: [{chunk_id, title, chunk}, ...]} をフロントエンドに送信
```

### 3. 表示フェーズ

```
フロントエンドでToolResult受信
    ↓
GroundingFile型に変換 {id, name, content}
    ↓
groundingFiles状態に追加
    ↓
GroundingFilesコンポーネントで表示
    ↓
ユーザーがクリック
    ↓
GroundingFileViewで全文表示
```

## カスタマイズポイント

### 1. バックエンド側のカスタマイズ

**フィールド名の変更：**

環境変数 `.env` で設定可能：

```bash
AZURE_SEARCH_IDENTIFIER_FIELD=custom_id
AZURE_SEARCH_TITLE_FIELD=doc_title
AZURE_SEARCH_CONTENT_FIELD=text_content
```

**検索方式の変更：**

`ragtools.py` の88-92行目をコメントアウトし、96行目のフィルター方式に切り替え可能：

```python
# フィルター方式を使用する場合（chunk_idがフィルタブルな場合）
search_results = await search_client.search(
    filter=f"search.in(chunk_id, '{list}')", 
    select=["chunk_id", "title", "chunk"]
)
```

### 2. フロントエンド側のカスタマイズ

**表示形式の変更：**

- `grounding-files.tsx`: カードレイアウトやアニメーションを調整
- `grounding-file.tsx`: ボタンのスタイルやアイコンを変更
- `grounding-file-view.tsx`: モーダルのサイズやレイアウトを調整

**多言語対応：**

`i18n` の翻訳ファイルでラベルをカスタマイズ可能：

```json
{
  "groundingFiles.title": "参照ドキュメント",
  "groundingFiles.description": "AIが回答に使用した情報源"
}
```

## トラブルシューティング

### 参照ドキュメントが表示されない場合

1. **バックエンドログを確認**: `Grounding source: ...` が出力されているか
2. **AIのシステムメッセージを確認**: `app.py` で `report_grounding` ツールの使用が指示されているか
3. **Azure AI Searchのインデックス設定を確認**: `chunk_id` フィールドが検索可能か

### 一部のドキュメントしか表示されない場合

1. **KEY_PATTERNのチェック**: 不正な文字が含まれるsource IDはフィルタリングされる
2. **検索結果の上限**: `top=len(sources)` で制限されている

### ドキュメント内容が表示されない場合

1. **フィールドマッピングを確認**: 環境変数が正しく設定されているか
2. **Azure AI Searchの select パラメータ**: 必要なフィールドが含まれているか

## まとめ

VoiceRAGアプリケーションの参照ドキュメント表示ロジックは、以下の3つの主要コンポーネントで構成されています：

1. **バックエンド（ragtools.py）**: AIが使用したソースを特定し、Azure AI Searchから詳細情報を取得
2. **データ転送（WebSocket/RTMiddleTier）**: バックエンドからフロントエンドへToolResultを送信
3. **フロントエンド（App.tsx + UI Components）**: データを受信し、ユーザーフレンドリーなUIで表示

このアーキテクチャにより、AIが回答生成に使用した情報源を透明性高く、リアルタイムでユーザーに提示することが可能になっています。
