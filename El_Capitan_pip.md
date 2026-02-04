
### Skill名: 「古いIntel Mac Python 3.8 pip TLS CA証明書エラー完全解決」

**対象環境**
- macOS 10.11 El Capitan（または古いIntel Mac）
- python.orgからインストールしたPython 3.8.x
- pip install で「Could not find a suitable TLS CA certificate bundle, invalid path: /etc/ssl/cert.pem」エラー
- Install Certificates.command が失敗する（鶏と卵問題）

**前提確認**
- `python3.8 -m certifi` → パスが出る（例: /Library/.../certifi/cacert.pem）
- これが出ない場合 → まず certifi を強制インストール（後述の迂回手順）

**解決手順（ステップバイステップ）**

1. **certifiを手動で強制インストール（証明書ループ脱出）**
   ```
   /Library/Frameworks/Python.framework/Versions/3.8/bin/python3.8 -m pip install --trusted-host pypi.org --trusted-host files.pythonhosted.org certifi
   ```
   - これで cacert.pem が site-packages/certifi/ に入る。

2. **/etc/ssl/cert.pem へのシンボリックリンク作成（必須）**
   ```
   sudo mkdir -p /etc/ssl
   sudo rm -f /etc/ssl/cert.pem   # 古いリンクがあれば削除
   sudo ln -sf $(/Library/Frameworks/Python.framework/Versions/3.8/bin/python3.8 -m certifi) /etc/ssl/cert.pem
   ```

   **確認**
   ```
   ls -l /etc/ssl/cert.pem
   # → cert.pem -> /Library/.../certifi/cacert.pem のように表示されればOK
   ```

3. **環境変数を永続的に設定（zshの場合）**
   ```
   echo 'export SSL_CERT_FILE=/etc/ssl/cert.pem' >> ~/.zshrc
   echo 'export REQUESTS_CA_BUNDLE=/etc/ssl/cert.pem' >> ~/.zshrc
   source ~/.zshrc
   ```
   - もし ~/.zshrc が効かない場合、代わりに ~/.zshenv や ~/.zprofile に追加
   - 確認：
     ```
     echo $SSL_CERT_FILE
     echo $REQUESTS_CA_BUNDLE
     # 両方 /etc/ssl/cert.pem と出るはず
     ```

4. **pipをテスト（これでエラー解消するはず）**
   ```
   pip3 install --upgrade pip setuptools wheel
   ```
   - 「Requirement already satisfied」だけ出て、ERRORが出なければ成功。

5. **元のライブラリインストール（成功例）**
   ```
   pip install flask==2.0.3 werkzeug==2.0.3 requests==2.27.1 python-dotenv==0.19.2 google-generativeai==0.3.2
   ```
   - まだエラーが出る場合だけ一時的に迂回：
     ```
     pip install --trusted-host pypi.org --trusted-host files.pythonhosted.org [パッケージ名]
     ```

6. **トラブルシューティング（失敗時用）**
   - Install Certificates.command がまだ失敗する場合 → 手順1〜3を先に実行してから再実行。
   - 環境変数が反映されない → ターミナルを完全にQuit → 再起動。
   - certifiのパスが変わった場合 → リンクを再作成（ステップ2）。
   - 最後の手段 → pip.conf で固定：
     ```
     mkdir -p ~/.pip
     echo '[global]' > ~/.pip/pip.conf
     echo 'cert = /etc/ssl/cert.pem' >> ~/.pip/pip.conf
     ```

**成功の最終確認ポイント**
- `pip3 --version` → 25.x.x 系
- `pip3 install --upgrade pip` → エラーなしで「Requirement already satisfied」
- venv作成 → pip install が普通に通る

**注意・Tips**
- このSkillはPython.org版Python 3.8専用。Homebrew版やpyenv版ではパスが変わる。
- El Capitanはもうサポート終了なので、可能ならHigh Sierra以上にアップデート推奨。
- サーバー用途ならvenvを必ず使い、グローバルは汚さない。
