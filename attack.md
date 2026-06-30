### Permission 相關（非 root 帳號下的工具安裝/執行）
大部分問題其實是「目錄寫入權限」問題，不是工具本身需要 root。常見解法：
```
# Go 工具（subfinder, httpx, katana, dalfox, gospider 等）
# 預設會寫到 $GOPATH/bin，不需要 root
go env -w GOBIN=$HOME/go/bin
export PATH=$PATH:$HOME/go/bin
echo 'export PATH=$PATH:$HOME/go/bin' >> ~/.bashrc

go install github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest

# pip 套件（arjun, droopescan, XSStrike 等）
# 用 --user 裝到使用者目錄，不寫系統路徑
pip install --user arjun

# 如果遇到「externally-managed-environment」錯誤（新版 Debian/Kali 常見）
pip install --user --break-system-packages arjun
# 或更乾淨的做法：用虛擬環境
python3 -m venv ~/venvs/pentest
source ~/venvs/pentest/bin/activate
pip install arjun XSStrike

# git clone 的工具（XSStrike, commix, tplmap, LinkFinder...）
# clone 到家目錄即可，不需要 /opt 或系統目錄
mkdir -p ~/tools && cd ~/tools
git clone https://github.com/s0md3v/XSStrike.git

# apt 安裝會需要 sudo（如 seclists, gobuster, nikto 沒裝的話）
# 如果完全沒有 sudo 權限，這些工具沒裝就裝不了，
# 只能改用 pip/go/git 有對應版本的替代品，或請管理員協助安裝
sudo apt install seclists gobuster nikto
```

### 唯一例外是如果你的 wordlist 或工具目錄被裝到 /usr/share 之類受保護的路徑、且你帳號沒有讀取權限——那就用 cp 複製一份到家目錄。

### DVWA/WebGoat 裡驗證能不能讀到資料庫內容
這個我可以給你方法論層面的指引：在 SQLi 找到注入點之後，要驗證「影響範圍」（也就是滲透測試報告裡會寫的「可讀取的機敏資料範圍」），標準流程是：

sqlmap -u "..." --dbs 列出資料庫，先確認你能列舉到什麼，而不是直接 dump
--current-db / --current-user 確認目前連線權限等級
-D dbname --tables 找出像 users、accounts 這類表名
只 dump 結構先看欄位（--columns），再決定要不要實際 dump 資料——這一步在正式測試報告裡通常只需要證明「可以讀到」，不需要把整張表撈出來
