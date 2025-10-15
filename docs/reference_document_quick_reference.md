# RAG参照ドキュメント表示ロジック - クイックリファレンス

## 📋 概要

このアプリケーションでは、AIが回答生成時に参照したドキュメント（グラウンディングソース）を自動的に表示します。

## 🔄 データフロー（簡易版）

```
質問 → AI検索 → AI回答生成 → report_grounding → Azure Search → フロントエンド → UI表示
```

## 📁 主要ファイル

### バックエンド
- **`app/backend/ragtools.py`** - `_report_grounding_tool()` 関数
  - 役割: AIが使用したソースの詳細情報を取得
  - 出力: `{chunk_id, title, chunk}` のリスト

### フロントエンド
- **`app/frontend/src/App.tsx`** - データ受信・変換
  - 役割: バックエンドからのデータを受信しUI用に変換
  - 変換: `{chunk_id, title, chunk}` → `{id, name, content}`

- **`app/frontend/src/components/ui/grounding-files.tsx`** - リスト表示
  - 役割: 参照ドキュメントのボタンリストを表示

- **`app/frontend/src/components/ui/grounding-file-view.tsx`** - 詳細表示
  - 役割: クリックされたドキュメントの全文をモーダルで表示

## 🔧 設定方法

環境変数（`.env` または Azure App Settings）:

```bash
AZURE_SEARCH_IDENTIFIER_FIELD=chunk_id  # ドキュメントID
AZURE_SEARCH_TITLE_FIELD=title          # タイトル
AZURE_SEARCH_CONTENT_FIELD=chunk        # 内容
```

## 🔍 コードの場所

### バックエンド処理
```python
# app/backend/ragtools.py (82-127行)
async def _report_grounding_tool(...):
    # Step 1: ソースIDをフィルタリング
    sources = [s for s in args["sources"] if KEY_PATTERN.match(s)]
    
    # Step 2: Azure AI Searchで詳細情報を取得
    search_results = await search_client.search(...)
    
    # Step 3: 結果をパッケージング
    docs = [{"chunk_id": ..., "title": ..., "chunk": ...}]
    
    # Step 4: フロントエンドに送信
    return ToolResult({"sources": docs}, TO_CLIENT)
```

### フロントエンド処理
```typescript
// app/frontend/src/App.tsx (34-42行)
onReceivedExtensionMiddleTierToolResponse: message => {
    // ① JSONをパース
    const result: ToolResult = JSON.parse(message.tool_result);
    
    // ② データ変換
    const files: GroundingFile[] = result.sources.map(x => ({
        id: x.chunk_id,
        name: x.title,
        content: x.chunk
    }));
    
    // ③ 状態を更新（累積）
    setGroundingFiles(prev => [...prev, ...files]);
}
```

## 📊 データ構造

### バックエンド → フロントエンド

| バックエンド | フロントエンド | 説明 |
|------------|--------------|------|
| `chunk_id` | `id` | ドキュメント識別子 |
| `title` | `name` | ドキュメントタイトル |
| `chunk` | `content` | ドキュメント内容 |

## 🎨 UI表示

1. **GroundingFiles** - カード内にボタンリスト表示
   - ファイルがない場合は非表示
   - アニメーション付きで表示

2. **GroundingFile** - 個別ボタン
   - ファイルアイコン + タイトル
   - クリック可能

3. **GroundingFileView** - モーダル表示
   - タイトル
   - 全文（スクロール可能）
   - 閉じるボタン

## 🔐 セキュリティ

- **ソースID検証**: 正規表現で英数字のみ許可
  ```python
  KEY_PATTERN = re.compile(r'^[a-zA-Z0-9_=\-]+$')
  ```

- **検索API使用**: フィルタではなく検索を使用（インジェクション対策）

## 🐛 トラブルシューティング

### 参照ドキュメントが表示されない
1. バックエンドログで "Grounding source: ..." を確認
2. AIのシステムメッセージに `report_grounding` 使用指示があるか確認
3. Azure AI Searchのインデックス設定を確認

### 一部のドキュメントしか表示されない
1. ソースIDに不正な文字が含まれていないか確認
2. 検索結果の上限 `top=len(sources)` を確認

## 📚 詳細ドキュメント

- **詳細な日本語ドキュメント**: `docs/reference_document_display_logic.md`
- **英語リファレンス**: `docs/README_reference_docs.md`
- **フロー図**: `docs/reference_document_flow_diagram.md`

## 💡 カスタマイズのヒント

### フィールド名の変更
環境変数を設定するだけ：
```bash
AZURE_SEARCH_TITLE_FIELD=my_custom_title_field
```

### 検索方式の変更
`ragtools.py` の96行目のコメントを参照してフィルター方式に切り替え可能。

### UI表示のカスタマイズ
- `grounding-files.tsx` - カードレイアウト
- `grounding-file.tsx` - ボタンスタイル
- `grounding-file-view.tsx` - モーダルレイアウト

## ✅ チェックリスト

開発時の確認事項：

- [ ] バックエンドで `_report_grounding_tool` が呼ばれているか？
- [ ] Azure AI Searchから正しいフィールドが取得できているか？
- [ ] フロントエンドでToolResultが正しくパースされているか？
- [ ] GroundingFileの型が正しくマッピングされているか？
- [ ] UIコンポーネントが正しくレンダリングされているか？

---

**最終更新**: このクイックリファレンスはドキュメント化作業の一環として作成されました。
