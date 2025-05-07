kickstart結構

PXE 基礎設施
  1.DHCP Server 可提供 IP 並設定：
    filename "pxelinux.0";
    next-server 指向 PXE/TFTP Server 的 IP

  2.TFTP Server 正常運作
    pxelinux.0 位於 /srv/tftp/
    pxelinux.cfg/default 設定檔存在(開機作業系統要吃到的檔案)
      有放置 linux 和 initrd.gz（解開自 Debian ISO）
      ＝＝＝＝＝＝＝＝
      TFTP 伺服器設定正確
          在 next-server 指定的機器上（也就是你的 Kickstart server），你需要：
          TFTP 服務運作中（可用 tftpd-hpa 或 xinetd+tftpd）
          /srv/tftp/pxelinux.0 存在
          /srv/tftp/pxelinux.cfg/default 設定選單正確
          指向的 linux 和 initrd.gz 存在於 /srv/tftp/debian12/
      
  3.HTTP Server 正常
     確認 preseed 檔案是否存在正確路徑
       preseed.cfg 放在 /var/www/html/preseed/debian12.cfg(開機流程跟要做的事情)
       ls -l /var/www/html/preseed/debian12.cfg
      確認 Apache2（或其他 HTTP Server）是否有啟動
       systemctl status apache2
         你的 Kickstart Server 上要有：
          Apache2 運作中，port 80 開放
            檔案 /var/www/html/preseed/debian12.cfg 可透過 http://10.168.10.98/preseed/debian12.cfg 讀取

  4.PXE Boot 設定內容正確
      pxelinux.cfg/default 裡的 KERNEL 與 APPEND 指向正確檔案與 Preseed URL
      
  5.實體機器支援並已啟用 PXE Boot
     BIOS設定為 網路開機優先 與 PXE Server 位於同一網段
     DHCP 有對該機器提供正確 IP 和 pxelinux.0
      
======================================
tftp目錄結構
/srv/tftp/
├── pxelinux.0
├── pxelinux.cfg/
│   └── default
└── debian12/
    ├── initrd.gz
    └── linux ← 是檔案，不是資料夾
＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝
TS 步驟

如要驗證 PXE Client 有沒有連到你的 Kickstart Server
tail -f /var/log/syslog | grep dhcp
tcpdump -i eth0 port 67 or port 69

問題集:
我看到有給IP 但是畫面 還是卡在
>> Start PXE over IPv4
沒有下一步

1.要確認tftp 服務有沒有開
sudo ss -lun | grep :69
應該看到類似：
udp   UNCONN  0 0 0.0.0.0:69  0.0.0.0:*

2.防火牆也要檢查

3.檢查tftp設定
cat tftpd-hpa
# /etc/default/tftpd-hpa

TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/srv/tftp"
TFTP_ADDRESS="0.0.0.0:69"
TFTP_OPTIONS="--secure"

4.再次確認 PXE 設定檔 /srv/tftp/pxelinux.cfg/default

5.檢查pxelinux.0 是不是空的
是空的話要跑下面流程
安裝 PXELINUX 工具包
sudo apt update
sudo apt install pxelinux syslinux-common -y
複製檔案到 TFTP 根目錄
sudo cp /usr/lib/PXELINUX/pxelinux.0 /srv/tftp/
sudo cp /usr/lib/syslinux/modules/bios/ldlinux.c32 /srv/tftp/
檢查檔案是否完整
ls -lh /srv/tftp/pxelinux.0 /srv/tftp/ldlinux.c32

6.檢查你的 PXE 結構最小應為：
/srv/tftp/
├── pxelinux.0         ← bootloader
├── ldlinux.c32        ← 必要模組
├── pxelinux.cfg/
│   └── default        ← menu 設定
└── debian12/
    ├── linux          ← Debian 核心（vmlinuz）
    └── initrd.gz      ← initrd
PS 光碟內的linux 跟initrd.gz不能用，要去這裡下載
https://deb.debian.org/debian/dists/bookworm/main/installer-amd64/current/images/netboot/debian-installer/amd64/

7.檔案權限
sudo chown -R tftp:tftp /srv/tftp
sudo chmod -R 755 /srv/tftp

8.DHCP 設定正確嗎
# 定義網段與參數
subnet 10.168.10.0 netmask 255.255.255.0 {
  range 10.168.10.15 10.168.10.20;
  option routers 10.168.10.254;
  option subnet-mask 255.255.255.0;
  option domain-name-servers 8.8.8.8, 1.1.1.1;
  option domain-name "adasbox.local";
  default-lease-time 600;
  max-lease-time 7200;

  # 下面這兩行是 PXE 關鍵設定
  filename "pxelinux.0";
  next-server 10.168.10.98;
}
PS: range 可以改，網段要從ipam找，不要重複即可。

9.確認你的 /srv/tftp/ 裡有這些檔案：
ls -lh /srv/tftp/
-rw-r--r-- 1 root root  26K pxelinux.0
-rw-r--r-- 1 root root  86K ldlinux.c32
drwxr-xr-x 2 root root 4.0K pxelinux.cfg
drwxr-xr-x 2 root root 4.0K debian12

10. 抓封包看一下問題，有看到抓到pxelinux.0 就代表有吃到了
10:08:45.169550 IP 10.168.10.16.1174 > kickstart98.tftp: TFTP, length 32, RRQ "pxelinux.0" octet blksize 1468
11. BIOS模式，預設要用BIOS開啟，這樣才能吃到PXE的DHCP
12. 上述做完重開機應該就會自動跑安裝畫面了
