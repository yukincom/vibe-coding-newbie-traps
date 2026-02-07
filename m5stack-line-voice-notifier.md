# M5Stack音声通知システム構築ガイド

## 概要

LINE通知をトリガーに、M5Stack CoreS3 Liteが音声で家族にメッセージを伝えるシステムの構築ガイド。

### システム構成

```
LINE → Render (Webhook) → Intel Mac (ポーリング) → M4 Mac (音声生成) → M5Stack (再生)
```

### 主要技術スタック

- **M5Stack**: CoreS3 Lite (ESP32-S3, 8MB PSRAM, 内蔵スピーカー)
- **音声合成**: macOS標準音声エンジン (say + afconvert)
- **通知**: LINE Messaging API → Render → ポーリング
- **連携**: Wi-Fi (LAN内通信)
- **開発環境**: Arduino IDE 2.x, Python 3.x, Flask

---

## アーキテクチャ詳細

### 1. 通知フロー

```
1. ユーザーがLINEでメッセージ送信 (例: "帰る")
2. Renderがwebhookを受信
3. Intel MacがRenderを1分ごとにポーリング (10秒に短縮可能)
4. キーワード判定 ("帰る" → "お母さんがそろそろ帰ってくるってー！")
5. M4 Macに音声生成リクエスト
6. M4 Macが44.1kHz WAVを生成
7. Intel Macが通知キューに追加
8. M5Stackが5秒ごとにチェック
9. 音声ダウンロード → 再生
```

### 2. 音声生成プロセス

**重要な発見: macOS say コマンドは22.05kHzで生成**

```bash
# NG: --data-format=LEF32@44100 は動作しない
say -v Kyoko --data-format=LEF32@44100 -o test.aiff "テスト"
# → Opening output file failed: fmt?

# OK: シンプルに生成 → afconvertで高品質アップサンプル
say -v Kyoko -o test.aiff "[[pbas 45]][[rate 160]]テスト"
afconvert -f WAVE -d LEI16@44100 --src-complexity bats -c 1 test.aiff test.wav
```

**afconvert の --src-complexity オプション:**
- `bats`: 最高品質 (推奨)
- `norm`: 標準品質
- `quick`: 高速だが低品質

### 3. ファイルサイズによる品質判定

```
22.05kHz: 約140KB (3秒の音声)
44.1kHz:  約270KB (3秒の音声)

→ 200KB未満なら22.05kHzのまま (アップサンプル失敗)
```

---

## セットアップ手順

### Phase 1: M4 Mac (音声生成サーバー)

#### 1-1. ディレクトリ作成

```bash
mkdir ~/voice_server
cd ~/voice_server
```

#### 1-2. voice_server.py 作成

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""voice_server.py - macOS音声生成サーバー"""

from flask import Flask, jsonify, request, send_file
import hashlib
import os
import subprocess
import time
import uuid

app = Flask(__name__)
VOICE_STORAGE_DIR = os.getenv("VOICE_STORAGE_DIR", "/tmp/voice_gen_store")
os.makedirs(VOICE_STORAGE_DIR, exist_ok=True)

def resolve_voice_name(requested_voice):
    """Enhanced/Premium版を優先して解決"""
    result = subprocess.run(["say", "-v", "?"], capture_output=True, text=True, check=True)
    
    # Premium優先
    for line in result.stdout.split("\n"):
        if line.startswith(requested_voice) and "(Premium)" in line:
            return line.split()[0]
    
    # Enhanced次点
    for line in result.stdout.split("\n"):
        if line.startswith(requested_voice) and "(Enhanced)" in line:
            return line.split()[0]
    
    # 大文字小文字無視で検索
    available = [line.split()[0] for line in result.stdout.split("\n") if line.strip()]
    lower_map = {v.lower(): v for v in available}
    
    if requested_voice in available:
        return requested_voice
    if requested_voice.lower() in lower_map:
        return lower_map[requested_voice.lower()]
    
    return None

@app.route("/health", methods=["GET"])
def health():
    return jsonify({"status": "ok", "service": "voice_generator"})

@app.route("/generate", methods=["POST"])
def generate():
    try:
        data = request.get_json(silent=True) or {}
        text = data.get("text", "").strip()
        requested_voice = data.get("voice", "Kyoko")
        rate = max(100, min(300, int(data.get("rate", 160))))
        pitch = max(10, min(90, int(data.get("pitch", 45))))

        if not text:
            return jsonify({"success": False, "error": "text is required"}), 400

        resolved_voice = resolve_voice_name(requested_voice)
        if not resolved_voice:
            return jsonify({"success": False, "error": f"voice not available: {requested_voice}"}), 400

        voice_id = f"{int(time.time())}_{uuid.uuid4().hex}"
        tmp_dir = "/tmp/voice_gen"
        os.makedirs(tmp_dir, exist_ok=True)

        tmp_aiff = f"{tmp_dir}/{voice_id}.aiff"
        tmp_wav = f"{tmp_dir}/{voice_id}.wav"
        final_wav = os.path.join(VOICE_STORAGE_DIR, f"{voice_id}.wav")

        clipped = text[:120]
        embedded_text = f"[[pbas {pitch}]][[rate {rate}]]{clipped}"

        # Step 1: say で 22.05kHz AIFF 生成
        subprocess.run(
            ["say", "-v", resolved_voice, "-o", tmp_aiff, embedded_text],
            check=True,
            capture_output=True
        )

        # Step 2: afconvert で 44.1kHz 高品質アップサンプル
        subprocess.run(
            [
                "afconvert",
                "-f", "WAVE",
                "-d", "LEI16@44100",
                "--src-complexity", "bats",  # 最高品質
                "-c", "1",
                tmp_aiff,
                tmp_wav
            ],
            check=True,
            capture_output=True,
        )

        with open(tmp_wav, "rb") as f:
            wav_data = f.read()

        sha256_hash = hashlib.sha256(wav_data).hexdigest()
        
        with open(final_wav, "wb") as f:
            f.write(wav_data)

        os.remove(tmp_aiff)
        os.remove(tmp_wav)

        return jsonify({
            "success": True,
            "voice_id": voice_id,
            "size": len(wav_data),
            "sha256": sha256_hash,
            "download_path": f"/voice/{voice_id}",
            "settings": {
                "text": clipped,
                "voice": resolved_voice,
                "requested_voice": requested_voice,
                "rate": rate,
                "pitch": pitch,
                "engine": "m4-voice-44k-bats",
            },
        })

    except Exception as e:
        return jsonify({"success": False, "error": str(e)}), 500

@app.route("/voice/<voice_id>", methods=["GET"])
def get_voice(voice_id):
    wav_path = os.path.join(VOICE_STORAGE_DIR, f"{voice_id}.wav")
    if not os.path.exists(wav_path):
        return jsonify({"success": False, "error": "voice not found"}), 404
    return send_file(wav_path, mimetype="audio/wav", as_attachment=True, download_name="voice.wav")

@app.route("/cleanup", methods=["POST"])
def cleanup():
    payload = request.get_json(silent=True) or {}
    max_age_seconds = int(payload.get("max_age_seconds", 3600))
    keep_latest = bool(payload.get("keep_latest", True))

    files = [(os.path.join(VOICE_STORAGE_DIR, f), os.path.getmtime(os.path.join(VOICE_STORAGE_DIR, f)))
             for f in os.listdir(VOICE_STORAGE_DIR) if f.endswith(".wav")]
    files.sort(key=lambda x: x[1], reverse=True)

    deleted = 0
    for idx, (path, mtime) in enumerate(files):
        if keep_latest and idx == 0:
            continue
        if time.time() - mtime > max_age_seconds:
            os.remove(path)
            deleted += 1

    return jsonify({"success": True, "deleted": deleted})

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5001, debug=False)
```

#### 1-3. Flask インストール & 起動

```bash
pip3 install flask
python3 voice_server.py
```

#### 1-4. ファイアウォール設定

```
システム設定 → ネットワーク → ファイアウォール
→ Python の受信接続を許可
```

---

### Phase 2: Intel Mac (メインサーバー)

#### 2-1. 音声生成関数を修正

```python
# ~/Documents/AI_assistant/server.py

import requests

VOICE_SERVER_URL = os.getenv("VOICE_SERVER_URL", "http://192.168.1.48:5001")
VOICE_DEFAULT_NAME = os.getenv("VOICE_DEFAULT_NAME", "Kyoko")

def generate_voice(text, rate=160, pitch=45, voice=None):
    """M4 Mac の音声生成サーバーを呼び出す"""
    try:
        selected_voice = voice or VOICE_DEFAULT_NAME
        response = requests.post(
            f"{VOICE_SERVER_URL}/generate",
            json={"text": text, "voice": selected_voice, "rate": rate, "pitch": pitch},
            timeout=30
        )
        response.raise_for_status()
        result = response.json()
        
        if not result.get("success"):
            return None
        
        voice_id = result["voice_id"]
        source_url = f"{VOICE_SERVER_URL}/voice/{voice_id}"
        
        return {
            "voice_id": voice_id,
            "source_url": source_url,
            "size": result["size"],
            "sha256": result["sha256"],
            "settings": result["settings"],
        }
    except Exception as e:
        print(f"❌ generate_voice error: {e}")
        return None
```

#### 2-2. voice_state 管理

```python
voice_state = {
    "latest_voice_id": None,
    "latest_voice_url": None,
    "latest_ready": False,
    "latest_sha256": None,
    "latest_settings": None,
}

def process_notification(user_id, message):
    global voice_state
    
    # ... 既存のキーワード判定 ...
    
    # 新規生成開始時に ready を false に
    voice_state["latest_ready"] = False
    
    voice_result = generate_voice(notification_message, rate=160, pitch=45)
    
    if voice_result:
        voice_state["latest_voice_id"] = voice_result["voice_id"]
        voice_state["latest_voice_url"] = voice_result["source_url"]
        voice_state["latest_sha256"] = voice_result["sha256"]
        voice_state["latest_settings"] = voice_result["settings"]
        voice_state["latest_ready"] = True  # 生成完了後に true
    
    pending_notifications.append({
        "sender": sender_name,
        "message": notification_message,
        "voice_id": voice_result["voice_id"] if voice_result else None,
    })
```

#### 2-3. /voice/latest エンドポイント改善

```python
@app.route("/voice/latest", methods=["GET"])
def get_latest_voice():
    global voice_state
    
    # 202: 生成中
    if not voice_state.get("latest_ready"):
        return jsonify({"success": False, "status": "not_ready"}), 202
    
    # 404: ファイルなし
    voice_url = voice_state.get("latest_voice_url")
    if not voice_url:
        return jsonify({"success": False, "status": "missing"}), 404
    
    # M4 Mac から取得してプロキシ
    try:
        remote_response = requests.get(voice_url, timeout=30)
        if remote_response.status_code != 200:
            return jsonify({"success": False, "status": "upstream_error"}), 502
        
        return Response(
            remote_response.content,
            mimetype="audio/wav",
            headers={"Content-Disposition": "attachment; filename=voice.wav"}
        )
    except requests.RequestException:
        return jsonify({"success": False, "status": "upstream_unreachable"}), 502
```

---

### Phase 3: M5Stack CoreS3 Lite

#### 3-1. Arduino IDE セットアップ

```
1. ツール → 設定
   Additional boards manager URLs:
   https://espressif.github.io/arduino-esp32/package_esp32_index.json

2. ツール → ボード → ボードマネージャ
   "esp32" → esp32 by Espressif Systems v2.0.17 インストール

3. ツール → ライブラリを管理
   - M5CoreS3 by M5Stack
   - ArduinoJson by Benoit Blanchon

4. ツール → ボード → esp32 → M5Stack-CoreS3
```

#### 3-2. スケッチ

```cpp
#include <M5CoreS3.h>
#include <WiFi.h>
#include <HTTPClient.h>
#include <ArduinoJson.h>

const char* ssid = "あなたのSSID";
const char* password = "あなたのパスワード";
const char* server_url = "http://192.168.1.46:5000";  // Intel Mac

String last_played_voice_id = "";

void setup() {
    Serial.begin(115200);
    M5.begin();
    M5.Display.setBrightness(255);
    
    // スピーカー初期化（音割れ防止）
    M5.Speaker.begin();
    M5.Speaker.setVolume(200);  // 255だと音割れする
    
    connectWiFi();
    checkAndPlayNotification();
}

void connectWiFi() {
    WiFi.mode(WIFI_STA);
    WiFi.begin(ssid, password);
    
    while (WiFi.status() != WL_CONNECTED && attempts < 20) {
        delay(500);
        attempts++;
    }
    
    if (WiFi.status() == WL_CONNECTED) {
        Serial.println("WiFi Connected!");
        Serial.println(WiFi.localIP());
    }
}

void checkAndPlayNotification() {
    HTTPClient http;
    String pendingUrl = String(server_url) + "/notify/pending";
    
    http.begin(pendingUrl);
    int httpCode = http.GET();
    
    if (httpCode == HTTP_CODE_OK) {
        String payload = http.getString();
        
        DynamicJsonDocument doc(2048);
        deserializeJson(doc, payload);
        
        if (doc["success"] && !doc["notification"].isNull()) {
            const char* voice_id = doc["notification"]["voice_id"];
            const char* message = doc["notification"]["message"];
            
            // 既読チェック
            if (String(voice_id) != last_played_voice_id) {
                if (downloadAndPlayVoice()) {
                    last_played_voice_id = String(voice_id);
                }
            }
        }
    }
    http.end();
}

bool downloadAndPlayVoice() {
    HTTPClient http;
    String voiceUrl = String(server_url) + "/voice/latest";
    
    http.begin(voiceUrl);
    int httpCode = http.GET();
    
    if (httpCode == 202) {
        // 生成中
        return false;
    }
    
    if (httpCode == HTTP_CODE_OK) {
        int len = http.getSize();
        uint8_t* wavData = (uint8_t*)ps_malloc(len);
        
        if (!wavData) return false;
        
        WiFiClient* stream = http.getStreamPtr();
        size_t bytesRead = 0;
        
        while (http.connected() && bytesRead < len) {
            size_t available = stream->available();
            if (available) {
                size_t toRead = min(available, (size_t)(len - bytesRead));
                stream->readBytes(wavData + bytesRead, toRead);
                bytesRead += toRead;
            }
            delay(1);
        }
        
        // 再生
        M5.Speaker.playWav(wavData, len);
        while (M5.Speaker.isPlaying()) {
            M5.update();
            delay(1);
        }
        
        free(wavData);
        http.end();
        return true;
    }
    
    http.end();
    return false;
}

void loop() {
    M5.update();
    
    // 5秒ごとにチェック
    static unsigned long lastCheck = 0;
    if (millis() - lastCheck > 5000) {
        checkAndPlayNotification();
        lastCheck = millis();
    }
    
    delay(100);
}
```

---

## トラブルシューティング

### 1. M5Stackが起動しない

**症状**: 画面が真っ暗、ボタン無反応

**原因**: NVS（不揮発性メモリ）未初期化

**解決**:
```bash
pip3 install esptool
esptool --chip esp32s3 --port /dev/tty.usbmodem21101 erase-flash
```

その後、Arduino IDEでファームウェア書き込み。

---

### 2. 音声が割れる・ノイズが出る

**症状**: パチパチノイズ、音割れ

**原因**: ボリュームが大きすぎる

**解決**:
```cpp
M5.Speaker.setVolume(200);  // 255 → 200
```

---

### 3. 音声が22.05kHzのまま

**症状**: `afinfo` で確認すると 22050 Hz

**原因**: afconvert が -r オプションを無視

**解決**:
```bash
# NG
afconvert -f WAVE -d LEI16 -r 44100 -c 1 input.aiff output.wav

# OK
afconvert -f WAVE -d LEI16@44100 --src-complexity bats -c 1 input.aiff output.wav
```

`@44100` を `-d` オプションに含める。

---

### 4. M4 Macに接続できない (Connection refused)

**チェックリスト**:
1. サーバーが起動しているか: `lsof -i :5001`
2. ファイアウォールで許可しているか
3. `app.run(host="0.0.0.0", ...)` になっているか
4. 正しいIPアドレスか: `ifconfig | grep "inet "`

---

### 5. /tmp ファイルが見つからない

**Finder で開く方法**:
```
Shift + Cmd + G → "/tmp" と入力 → Enter
```

または:
```bash
open /tmp
```

**注意**: `/tmp` は再起動すると消える。永続化したい場合はホームディレクトリに変更:
```python
VOICE_STORAGE_DIR = f"{os.path.expanduser('~')}/voice_storage"
```

---

## ベストプラクティス

### 音声品質

1. **サンプリングレート**: 44.1kHz推奨
2. **リサンプル品質**: `--src-complexity bats`
3. **ピッチ/レート**: 極端な値を避ける (pitch: 30-60, rate: 140-180)

### パフォーマンス

1. **ポーリング間隔**: 10秒（遅延と負荷のバランス）
2. **M5Stackチェック間隔**: 5秒
3. **古いファイル削除**: 1時間ごと（ストレージ節約）

### セキュリティ

1. **LAN内通信**: インターネット公開しない
2. **認証なし**: 信頼できるネットワーク内のみ
3. **ファイアウォール**: 必要なポートのみ開放

---

## カスタマイズ例

### 1. 送信者によって声を変える

```python
def process_notification(user_id, message):
    if user_id == mama_id:
        voice = "Kyoko"  # お母さん → 女性の声
    elif user_id == papa_id:
        voice = "Otoya (Enhanced)"  # お父さん → 男性の声
    else:
        voice = "Kyoko"
    
    voice_result = generate_voice(notification_message, voice=voice)
```

### 2. 時刻によってボリューム調整

```python
import datetime

hour = datetime.datetime.now().hour

if 22 <= hour or hour < 7:
    # 夜間は静かに
    # server.py で M5Stack に volume 情報を送る
    volume = 150
else:
    volume = 200
```

### 3. メッセージに応じて感情表現

```python
if "ありがとう" in message:
    pitch = 50  # 嬉しそうに
    rate = 170
elif "ごめん" in message:
    pitch = 35  # 申し訳なさそうに
    rate = 140
else:
    pitch = 45
    rate = 160
```

---

## 利用可能な音声一覧

### 日本語音声 (macOS標準)

| 音声名 | 特徴 |
|--------|------|
| Kyoko | 女性、標準、クリア |
| Otoya (Enhanced) | 男性、エンハンスド、自然 |
| Eddy | 若い男性 |
| Flo | 若い女性 |
| Grandma | おばあちゃん |
| Grandpa | おじいちゃん |
| Reed | 落ち着いた男性 |
| Rocko | 個性的な男性 |
| Sandy | 個性的な女性 |
| Shelley | 個性的な女性 |

### 確認方法

```bash
say -v ? | grep ja_JP
```

---

## 開発環境

### 必要なもの

- **M5Stack CoreS3 Lite**: ESP32-S3, 8MB PSRAM, 内蔵スピーカー
- **Mac (M4以降推奨)**: 音声生成用
- **Mac (Intel可)**: メインサーバー用
- **Arduino IDE**: 2.3.7以降
- **Python**: 3.x
- **Wi-Fiルーター**: LAN内通信用

### 開発ツール

```bash
# Python
pip3 install flask requests python-dotenv apscheduler

# Arduino
- ESP32 ボードパッケージ v2.0.17
- M5CoreS3 ライブラリ
- ArduinoJson ライブラリ
```

---

## まとめ

このシステムは以下の特徴があります：

✅ **リアルタイム性**: 最大15秒の遅延（実用的）
✅ **高品質音声**: 44.1kHz, 16bit, BATS品質
✅ **拡張性**: 音声・感情表現のカスタマイズ可能
✅ **安定性**: 2-phase publish による競合回避
✅ **デバッグ性**: 詳細なログ出力

家族間のコミュニケーションを音声で豊かにする、実用的なIoTシステムです。

---

## 参考リンク

- [M5Stack CoreS3 公式ドキュメント](https://docs.m5stack.com/en/core/CoreS3)
- [ESP32 Arduino Core](https://github.com/espressif/arduino-esp32)
- [macOS say コマンド](https://ss64.com/osx/say.html)
- [afconvert マニュアル](https://ss64.com/osx/afconvert.html)

---

**作成日**: 2026年2月6日
**バージョン**: 1.0
**対象**: M5Stack CoreS3 Lite + macOS音声合成
