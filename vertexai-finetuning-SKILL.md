# VertexAI ファインチューニング SKILL

## 概要
このSKILLは、Google Cloud VertexAIでGeminiモデルをファインチューニングする際の正しい手順と、初心者が陥りやすいミスを回避するためのガイドです。

## 前提条件チェックリスト

### 1. プロジェクト設定の確認
```bash
# プロジェクトIDの確認（重要：Project NumberではなくProject ID）
gcloud config get-value project

# プロジェクト一覧を表示
gcloud projects list
```

**よくあるミス：Project NumberとProject IDの混同**
- ❌ 間違い: `244775109649` (Project Number - 数字のみ)
- ✅ 正解: `imposing-kite-474908-h0` (Project ID - 文字列)

### 2. API有効化
```bash
# Vertex AI APIを有効化
gcloud services enable aiplatform.googleapis.com --project=YOUR-PROJECT-ID
```

### 3. 権限設定
```bash
# 自分のメールアドレスを確認
gcloud config get-value account

# 必要な権限を付与
gcloud projects add-iam-policy-binding YOUR-PROJECT-ID \
  --member="user:$(gcloud config get-value account)" \
  --role="roles/aiplatform.admin"

gcloud projects add-iam-policy-binding YOUR-PROJECT-ID \
  --member="user:$(gcloud config get-value account)" \
  --role="roles/storage.objectViewer"

# 権限確認（0 itemsでなければOK）
gcloud projects get-iam-policy YOUR-PROJECT-ID \
  --flatten="bindings[].members" \
  --filter="bindings.role:roles/aiplatform.admin"
```

### 4. 認証設定
```bash
# Application Default Credentials (ADC) を設定
# これがないとPythonから実行できない
gcloud auth application-default login
```

**よくあるミス：2種類の認証の混同**
- `gcloud auth login`: gcloudコマンド用（CLIで使う）
- `gcloud auth application-default login`: Python SDK用（**必須**）

## モデル選択の重要ポイント

### チューニング対応モデルの確認方法

**最も確実な方法：GUIで確認**
```bash
# ブラウザでVertex AI Consoleを開く
open "https://console.cloud.google.com/vertex-ai/generative/tuning/create-tuned-model?project=YOUR-PROJECT-ID"
```

または:
1. Google Cloud Console → Vertex AI → Model Garden
2. 「Tune and distill」をクリック
3. ドロップダウンで選べるモデル名を確認

### 2025年1月時点の対応状況

**❌ チューニング非対応（よくある間違い）**
- Gemini 1.5 Pro (`gemini-1.5-pro-002`)
- Gemini 1.5 Flash (`gemini-1.5-flash-002`)

**✅ チューニング対応**
- Gemini 2.5 Flash (`gemini-2.5-flash`)
- Gemma 2シリーズ (`gemma2-2b-it`, `gemma2-9b-it` など)
- 他のGemmaバリエーション
- 「preview」「001」のような細かいモデル指定をすると受け付けない場合あり。最新データを要確認

**重要：モデル情報は頻繁に変わるため、必ずGUIで最新情報を確認すること**

## データセット準備

### JSONL形式の要件
```json
{"contents": [{"role": "user", "parts": [{"text": "ユーザーの質問"}]}, {"role": "model", "parts": [{"text": "モデルの回答"}]}]}
```

### データセットの検証
```bash
# GCSにアップロードされたデータを確認
gsutil cat gs://YOUR-BUCKET/tuning_data.jsonl | head -n 3 | jq .

# ファイルサイズ確認
gsutil ls -l gs://YOUR-BUCKET/tuning_data.jsonl
```

## Pythonコード実装

### 基本的なファインチューニングコード

```python
import vertexai
from vertexai.tuning import sft

# プロジェクト設定
PROJECT_ID = "your-project-id"  # Project IDを使用
LOCATION = "us-central1"
BUCKET = "your-bucket-name"

# 初期化
vertexai.init(project=PROJECT_ID, location=LOCATION)

print("Starting tuning job...")

try:
    # チューニングジョブの作成
    job = sft.train(
        source_model="gemini-2.5-flash",  # 公式サイトで確認した正確なモデル名
        train_dataset=f"gs://{BUCKET}/tuning_data.jsonl",
        tuned_model_display_name="your-tuned-model-v1",
        epochs=3,  # epoch_countではなくepochs
        learning_rate_multiplier=1.0,
    )
    
    print(f"Success! Job name: {job.name}")
    print(f"Resource: {job.resource_name}")
    print(f"\nCheck progress:")
    print(f"gcloud ai tuning-jobs describe {job.name} --location={LOCATION}")
    print(f"\nGUI: https://console.cloud.google.com/vertex-ai/tuning?project={PROJECT_ID}")
    
except Exception as e:
    print(f"Error: {e}")
    import traceback
    traceback.print_exc()
```

### よくあるパラメータミス

| 間違い | 正しい |
|--------|--------|
| `epoch_count=3` | `epochs=3` |
| `training_dataset=` | `train_dataset=` |
| `from vertexai.preview.tuning` | `from vertexai.tuning` (previewは不安定) |

## トラブルシューティング

### エラー1: 認証エラー
```
DefaultCredentialsError: Your default credentials were not found
```

**解決方法：**
```bash
gcloud auth application-default login
```

### エラー2: モデル非対応エラー
```
400 Base model gemini-1.5-pro-002 is not supported
```

**解決方法：**
1. GUIでチューニング対応モデルを確認
2. モデル名を正確にコピー（バージョン番号含む）
3. コードを更新して再実行

### エラー3: 権限エラー
```bash
# 権限確認で「Listed 0 items」が表示される
```

**解決方法：**
```bash
# 権限を付与
gcloud projects add-iam-policy-binding YOUR-PROJECT-ID \
  --member="user:YOUR-EMAIL" \
  --role="roles/aiplatform.admin"
```

### エラー4: 全角文字のSyntaxError
```
SyntaxError: invalid character '：' (U+FF1A)
```

**解決方法：**
```bash
# ファイルを作り直す
cat > tuning.py << 'EOF'
# コードをここに貼り付け
EOF

# キャッシュをクリア
rm -rf __pycache__
find . -name "*.pyc" -delete
```

## ベストプラクティス

### 1. 段階的アプローチ
1. ✅ 権限設定
2. ✅ 認証設定
3. ✅ モデル名確認（GUI）
4. ✅ 最小構成でテスト実行
5. ✅ パラメータ調整

### 2. デバッグ情報の収集
```python
# エラー時に詳細情報を表示
try:
    job = sft.train(...)
except Exception as e:
    print(f"Error: {e}")
    print(f"\nDebug info:")
    print(f"  Project: {PROJECT_ID}")
    print(f"  Location: {LOCATION}")
    print(f"  Dataset: {train_dataset}")
    import traceback
    traceback.print_exc()
```

### 3. 進捗確認
```bash
# CLIで確認
gcloud ai tuning-jobs describe JOB_NAME --location=LOCATION

# または GUIで確認
# https://console.cloud.google.com/vertex-ai/tuning?project=PROJECT_ID
```

## チェックリスト

実行前に以下を確認：

- [ ] Project ID（Numberではない）を確認した
- [ ] Vertex AI APIを有効化した
- [ ] 必要な権限（aiplatform.admin）を付与した
- [ ] ADC認証を設定した（`gcloud auth application-default login`）
- [ ] GUIでチューニング対応モデルを確認した
- [ ] データセットがGCSにアップロードされている
- [ ] JSONLフォーマットが正しい
- [ ] パラメータ名が正しい（`epochs`, `train_dataset`）

## よくある質問

**Q: GrokやGeminiが「Gemini 1.5 Proでできる」と言うのですが？**
A: 2025年1月時点では誤情報です。必ずGUIで最新の対応モデルを確認してください。

**Q: Project NumberとProject IDの違いは？**
A: Project Numberは数字のみ（例：244775109649）、Project IDは文字列（例：imposing-kite-474908-h0）。VertexAIではProject IDを使用します。

**Q: `epoch_count`エラーが出ます**
A: 正しくは`epochs`です（sがつく）。

**Q: ローカルでチューニングできますか？**
A: 非公式な方法（HuggingFace等）はありますが、Google公式ではVertexAI経由のみサポートされています。

## 参考リンク

- Vertex AI Console: `https://console.cloud.google.com/vertex-ai`
- Model Garden: `https://console.cloud.google.com/vertex-ai/model-garden`
- ドキュメント: `https://cloud.google.com/vertex-ai/docs/generative-ai/models/tune-models`
