# deploy-local-AI-teaching

> 在 **Proxmox** 上部署本地 AI 服務：**Ollama + Open WebUI**  
> 本指南詳細說明如何在 Debian 環境下安裝、設定與啟動完整的本地 AI 系統，  
> 讓你在沒有外網 API 的情況下，離線使用 Llama3、Mistral 等大型語言模型。

---

## 📚 目錄 (Table of Contents)

- [💡 專案簡介](#-專案簡介)
- [🧰 系統與硬體需求](#-系統與硬體需求)
- [⚙️ 前置條件](#️-前置條件)
- [1️⃣ 安裝 Ollama](#1️⃣-安裝-ollama)
- [2️⃣ 安裝 Open WebUI](#2️⃣-安裝-open-webui)
- [3️⃣ 建立 systemd 自動啟動服務(選用)](#建立-systemd-自動啟動服務(選用))
- [4️⃣ 下載模型與測試](#4️⃣-下載模型與測試)
- [5️⃣ 使用WebUI](#5️⃣-使用WebUI)
- [6️⃣ 常見問題 (FAQ)](#6️⃣-常見問題-faq)
- [7️⃣ 更新與維護](#7️⃣-更新與維護)
- [8️⃣ 在proxmox上直通GPU](#8️⃣-在proxmox上直通GPU)

---

## 💡 專案簡介

本教學示範如何在 **Proxmox VE** 建立一台 Debian VM，  
並在其中安裝 **Ollama**（本地 LLM 執行引擎）與 **Open WebUI**（網頁操作介面）。

完成後，你可以在本地瀏覽器上以 ChatGPT 風格與模型互動，  
支援模型如：`llama3`, `mistral`, `gemma`, `phi3`, `qwen`, 等等。

---

## 🧰 系統與硬體需求

| 項目 | 建議配置 |
|------|------------|
| 作業系統 | Debian 12  |
| CPU | ≥ 4 cores，支援 AVX |
| RAM | ≥ 8GB（推薦 16GB 以上） |
| 儲存空間 | ≥ 30GB（模型會佔空間） |
| GPU（可選） | NVIDIA GPU，支援 CUDA |
| 網路 | 能連外（下載模型）或有內部鏡像源 |

---

| 項目 | 個人配置 |
|------|------------|
|主機|DELL R730XD|
| 作業系統 | Debian 12  |
| CPU | E5 2670V3 |
| RAM | 64G DDR4 ECC |
| 儲存空間 | 1T HDD |
| GPU（可選） | NVIDIA RTX 3090 24G |
| 網路 | 連外固網500M/500M |

## ⚙️ 前置條件

```bash
sudo sed -i 's/main/main contrib non-free non-free-firmware/g' /etc/apt/sources.list
sudo apt update && sudo apt -y upgrade
sudo apt install -y curl git vim htop ca-certificates python3-venv python3-pip
```

啟用 GPU
```
sudo apt install -y linux-headers-$(uname -r) nvidia-driver
sudo reboot
nvidia-smi
```

找不到GPU或是是AMD GPU怎麼辦
->[8️⃣ 在proxmox上直通GPU](#8️⃣-在proxmox上直通GPU)

## 1️⃣ 安裝 Ollama
```
curl -fsSL https://ollama.com/install.sh | sh
sudo systemctl enable --now ollama
```

檢查狀態
```
systemctl status ollama --no-pager
```

測試 API
```
curl http://127.0.0.1:11434/api/tags
```

# 外部連線（選用）
若要讓其他裝置透過網路連入 Ollama
```
sudo nano /etc/ollama.conf
```
加入
```
OLLAMA_HOST=0.0.0.0:11434
```
重啟
```
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

## 2️⃣ 安裝 Open WebUI
1.建立用戶與資料目錄
```
sudo useradd -r -m -d /var/lib/open-webui -s /usr/sbin/nologin openwebui || true
sudo mkdir -p /var/lib/open-webui
sudo chown -R openwebui:openwebui /var/lib/open-webui
```

2.安裝 Python 環境與 WebUI
```
sudo mkdir -p /opt/open-webui
sudo python3 -m venv /opt/open-webui/venv
sudo /opt/open-webui/venv/bin/pip install --upgrade pip open-webui
```

## 3️⃣　建立 systemd 自動啟動服務(選用)
建立 systemd檔案
```
sudo nano /etc/systemd/system/open-webui.service
```

貼上
```
[Unit]
Description=Open WebUI (Ollama Web Interface)
After=network-online.target ollama.service
Wants=network-online.target

[Service]
Type=simple
User=openwebui
Group=openwebui
WorkingDirectory=/var/lib/open-webui
EnvironmentFile=/etc/openwebui.env
ExecStart=/opt/open-webui/venv/bin/open-webui serve --host 0.0.0.0 --port 8080 --data-dir /var/lib/open-webui
Restart=always
RestartSec=5
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=full
ProtectHome=true

[Install]
WantedBy=multi-user.target
```

啟用檢查
```
sudo systemctl daemon-reload
sudo systemctl enable --now open-webui
systemctl status open-webui --no-pager
```

## 4️⃣ 下載模型與測試
Mistral為例
```
# 抓取模型
ollama pull mistral:latest

# 列出模型，看是否出現 mistral
ollama list | grep -i mistral

# 快速互動測試（離開按 Ctrl+C）
ollama run mistral:latest
```
## 5️⃣ 使用WebUI
先找IP
```
ip a
```
找到機器的ip後，開啟瀏覽器，輸入
```
http://(你的ip)/
```

## 6️⃣ 常見問題 (FAQ)
| 問題               | 原因                            | 解決方式                                          |            |
| ---------------- | ----------------------------- | --------------------------------------------- | ---------- |
| WebUI 無法開啟       | 服務沒啟動或 port 被佔用               | `sudo systemctl restart open-webui`、`ss -lntp |  |
| WebUI 連不到 Ollama | `.env` 或 `OLLAMA_BASE_URL` 錯誤 | 改為 `http://127.0.0.1:11434` 或實際 IP            |            |
| 模型下載卡住           | 網速慢或磁碟滿                       | 檢查空間、換網路、或手動下載                                |            |
| GPU 無法使用         | 驅動未裝或 Ollama 未偵測 GPU          | 重新安裝 NVIDIA 驅動並重啟 Ollama                      |  (#8️⃣-在proxmox上直通GPU) |
| 開機不自動啟動          | systemd 未啟用                   | `sudo systemctl enable ollama open-webui`     |            |

## 7️⃣ 更新與維護
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

## 8️⃣ 在proxmox上直通GPU
# 都在PVE上執行
# 一、前置條件
| 項目        | 說明                                       |
| --------- | ---------------------------------------- |
| 主機 BIOS   | 必須支援 **VT-d (Intel)** 或 **SVM (AMD-Vi)** |
| 作業系統      | Proxmox VE 7 / 8                         |
| GPU       | NVIDIA 或 AMD 顯示卡                         |
| VM        | Debian 12 / Ubuntu 22.04 最佳              |
| CPU / 主機板 | 支援 IOMMU 的平台（大多數伺服器皆可）                   |

# 二、啟用 IOMMU 與 VFIO
1️⃣修改 grub 設定
```
sudo nano /etc/default/grub
```

找到
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet"
```

依照 CPU 改成以下之一
Intel
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"
```

AMD
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt"
```

儲存更新
```
update-grub
```

2️⃣啟用 VFIO 模組
```
echo -e "vfio\nvfio_iommu_type1\nvfio_pci\nvfio_virqfd" > /etc/modules-load.d/vfio.conf
```

3️⃣ 屏蔽 GPU 預設驅動
防止 Proxmox Host 佔用 GPU
NVIDIA
```
echo -e "blacklist nouveau\nblacklist nvidia\nblacklist nvidiafb" > /etc/modprobe.d/blacklist-nvidia.conf
```
AMD
```
echo -e "blacklist amdgpu\nblacklist radeon" > /etc/modprobe.d/blacklist-amd.conf
```

4️⃣ 更新 initramfs
```
update-initramfs -u -k all
```

5️⃣ 重啟
```
reboot
```

# 三、確認 IOMMU 啟動成功
開機後
```
dmesg | grep -e IOMMU -e DMAR
```

出現類以下文字似代表成功啟用
```
DMAR: IOMMU enabled
AMD-Vi: Initialized
```

#　四、查詢 GPU PCI ID
```
lspci -nn | grep -E "VGA|3D|Display"
```
例如
```
03:00.0 VGA compatible controller [0300]: NVIDIA Corporation GA102 [GeForce RTX 3090] [10de:2204]
03:00.1 Audio device [0403]: NVIDIA Corporation GP104 High Definition Audio Controller [10de:1b09]
```
`10de:2204` 和 `10de:1b09`（這就是 GPU 的 Vendor:Device ID）

# 五、將 GPU 綁定給 VFIO
編輯檔案
```
nano /etc/modprobe.d/vfio-pci.conf
```
加入
```
options vfio-pci ids=10de:2204,10de:1b09 disable_vga=1
```
換成你剛剛查到的 ID。
如果有多張 GPU，可用逗號分隔

重新更新
```
update-initramfs -u
reboot
```

# 六、確認 GPU 已交由 VFIO 管理
輸入
```
lspci -k | grep -EA3 "VGA|3D|Display"
```

顯示以下代表 GPU 已準備好被直通給 VM 使用
```
Kernel driver in use: vfio-pci
```

# 七、設定 VM（以 101 為例）
1️⃣ 關閉 VM
```
qm stop 101
```
2️⃣ 編輯 VM 設定檔
```
nano /etc/pve/qemu-server/101.conf
```
加入這幾行（依你的 PCI Bus 編號修改）
```
hostpci0: 0000:03:00.0,pcie=1
hostpci1: 0000:03:00.1
machine: q35
args: -cpu host,kvm=on
```

儲存`curl+O`後啟動 VM
```
qm start 101
```

# 八、進入 VM 內安裝 GPU 驅動
NVIDIA 驅動安裝
```
sudo apt install -y linux-headers-$(uname -r) build-essential dkms
sudo apt install -y nvidia-driver
sudo reboot
```
確認
```
nvidia-smi
```
可顯示 GPU代表直通成功！

AMD 驅動安裝
```
sudo apt update
sudo apt install -y firmware-amd-graphics libdrm-amdgpu1 xserver-xorg-video-amdgpu
sudo reboot
```
確認
```
lspci -k | grep amdgpu
```
輸出有以下代表直通成功
```
[drm] Initialized amdgpu ...
```
# 九、驗證 AI 可用 GPU
在 VM 內
```
ollama run mistral
```
支援 GPU，會顯示
```
Using CUDA / ROCm backend
```
或
```
ollama list
```
中顯示
```
GPU: (GPU型號)
```
# 十、常見錯誤排查表
| 問題                                    | 原因                      | 解法                    |
| ------------------------------------- | ----------------------- | --------------------- |
| 開機卡在 Proxmox Logo                     | 顯示卡被主機占用                | 檢查 blacklist 是否生效     |
| VM 啟動報錯 “IOMMU group not found”       | BIOS 未開啟 VT-d / SVM     | 進 BIOS 開啟             |
| VM 開機畫面黑屏                             | 顯卡綁錯組合或沒綁音訊功能           | 加上 GPU + Audio 兩個 PCI |
| VM 開機報錯 `vfio: failed to setup iommu` | 顯卡與 Host 共用 IOMMU Group | 開啟 ACS override       |
| VM 無法安裝驅動                             | Debian kernel 太舊        | 升級到 6.1+              |
| GPU 驅動 OK 但 AI 不使用 GPU                | Ollama 或模型版本不支援         | 重新拉最新版 Ollama、更新模型    |
