# deploy-local-AI-teaching

# 硬體
我的硬體
DELL R730XD
CPU:E5 2670V3
GPU:RTX 3090 24G
RAM:DDR4 64G ECC
harddisk:1T HDD
power:2x 750W , 1x700w(GPU)

# 系統需求
Debian 12(13會有版本衝突問題）

# 1.安裝環境
先安裝作業系統套件、NVIDIA 驅動等
```
# 0) 基礎：啟用套件來源與更新
sudo sed -i 's/main/main contrib non-free non-free-firmware/g' /etc/apt/sources.list
sudo apt update && sudo apt -y upgrade
sudo apt install -y curl git vim htop ca-certificates

# 1) 安裝 NVIDIA 驅動（若無 GPU 可略過）
sudo apt install -y linux-headers-$(uname -r) nvidia-driver
sudo reboot

# 重開後確認 GPU
nvidia-smi
# （可選）開啟持久模式，降低首次延遲
sudo nvidia-smi -pm 1

# 2) Python 相關工具（給 Open WebUI 用）
sudo apt install -y python3-venv python3-pip

```

# 2.安裝Ollama
```
# 安裝 Ollama
curl -fsSL https://ollama.com/install.sh | sh

# 確保開機自啟並啟動
sudo systemctl enable --now ollama

# 驗證服務與 API
systemctl status ollama --no-pager
curl http://127.0.0.1:11434/api/tags
```
預設只綁 127.0.0.1:11434，安全且足夠本機 WebUI 使用。
若要讓其他主機連入：在 /etc/ollama.conf 設 OLLAMA_HOST=0.0.0.0:11434 → sudo systemctl daemon-reload && sudo systemctl restart ollama

# 3.安裝Open WebUI（pip+venv）
```
# 1) 建服務帳號與資料目錄
sudo useradd -r -m -d /var/lib/open-webui -s /usr/sbin/nologin openwebui || true
sudo mkdir -p /var/lib/open-webui
sudo chown -R openwebui:openwebui /var/lib/open-webui

# 2) 建 venv 並安裝 Open WebUI
sudo mkdir -p /opt/open-webui
sudo python3 -m venv /opt/open-webui/venv
sudo /opt/open-webui/venv/bin/pip install --upgrade pip open-webui

# 3) 環境設定（讓 WebUI 找到本機 Ollama）
sudo tee /etc/openwebui.env >/dev/null <<'EOF'
OLLAMA_BASE_URL=http://127.0.0.1:11434
# 單人自用、只在內網時可改成關閉登入頁（移除註解）：
# WEBUI_AUTH=False
EOF
```

# 4.(選用)建立 systemd 服務（自動啟用）
```
sudo tee /etc/systemd/system/open-webui.service >/dev/null <<'EOF'
[Unit]
Description=Open WebUI (Ollama Web UI)
After=network-online.target ollama.service
Wants=network-online.target

[Service]
Type=simple
User=openwebui
Group=openwebui
WorkingDirectory=/var/lib/open-webui
EnvironmentFile=/etc/openwebui.env
# 直接指定資料夾避免環境變數相依
ExecStart=/opt/open-webui/venv/bin/open-webui serve --host 0.0.0.0 --port 8080 --data-dir /var/lib/open-webui
Restart=always
RestartSec=5

# 簡易加固
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=full
ProtectHome=true

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now open-webui

# 檢查服務與連線埠
systemctl status open-webui --no-pager
ss -lntp | grep 8080
```

# 5.開啟WebUI
瀏覽器開：http://<你的伺服器IP>:8080
第一次會建立管理者帳號

不知道IP的話，在linux中
```
ip a
```
就會出現了

# 6.抓模型與驗證（Mistral為例）
```
# 抓取模型
ollama pull mistral:latest

# 列出模型，看是否出現 mistral
ollama list | grep -i mistral

# 快速互動測試（離開按 Ctrl+C）
ollama run mistral:latest
```

# 常用維護、升級與排錯
升級 Open WebUI
```
sudo /opt/open-webui/venv/bin/pip install --upgrade open-webui
sudo systemctl restart open-webui
```

升級 Ollama
```
curl -fsSL https://ollama.com/install.sh | sh
sudo systemctl restart ollama
```

# 常見問題
http://IP:8080 開不起來 → systemctl status open-webui、看 journalctl -u open-webui、ss -lntp | grep 8080

WebUI 連不到 Ollama → 檢查 /etc/openwebui.env 的 OLLAMA_BASE_URL、curl 127.0.0.1:11434/api/tags

GPU 沒吃到 → nvidia-smi 要能看到卡；重啟 ollama 後再測推論

模型下載很慢 → 模型檔大，請用有線網路與足夠磁碟空間（SSD/NVMe 更快）
