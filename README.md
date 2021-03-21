

# Night-surveillance-camera

# 目的

家庭菜園の獣害防止を目的として、夜間に菜園に侵入する動物の動画を撮影し、クラウドプラットフォームに送信する。電池（モバイルバッテリー）駆動で動くよう省電力で動くものにする。

# 使用する機材

- Raspberry Pi Zero
- Raspberry Pi Camera
- USBスティックデータ通信端末（3G対応）
- RTCモジュール
- モバイルバッテリー
- USBケーブル

# Raspberry Pi Zeroの設定

### Raspberry Pi OSの書き込み

以下の操作はMacで行っている。

使い古しのSDカード（IOデータ製 16GB）を使用したので、SD Card Formatterを使用してフォーマットを行う。

Raspberry Pi財団の公式サイトからOSのイメージファイルをダウンロードする。

https://www.raspberrypi.org/software/operating-systems/#raspberry-pi-os-32-bit

ダウンロードしたのは、Raspberry Pi OS Lite。2021年3月13日時点でのKernelバージョンは5.4。

ダウンロードされたファイルは2021-01-11-raspios-buster-armhf-lite.zip。

このファイルをBalenaEtcherを使ってSDカードに書き込む。BalenaEtcherであれば解凍せずにZipのままで書き込みが可能。

セットアップはUSB OTG (On-The-Go) EthernetでホストPCと接続して行う。

（Qiitaの記事、https://qiita.com/Liesegang/items/dcdc669f80d1bf721c21を参考にした）

SSHを有効にするため、SSDのルートディレクトリにSSHという名前の空のファイルを作製する。

touchコマンドでSSHを作成する。

```
$ touch /Volumes/boot/ssh
```

次に、`config.txt`の一番最後に下記の行を追加する。

```
dtoverlay=dwc2
```

さらに、`cmdline.txt`の`rootwait`の後に`modules-load=dwc2,g_ether`を追記する。

```
console=serial0,115200 console=tty1 root=PARTUUID=e8af6eb2-02 rootfstype=ext4 elevator=deadline fsck.repair=yes rootwait modules-load=dwc2,g_ether quiet init=/usr/lib/raspi-config/init_resize.sh
```

USBにデータ通信端末を挿しているとOTGでのSSH接続ができないため、wifiでも接続できるようにしておく。

SSDのルートディレクトリにwpa_supplicant.confという名前でファイルを作製する。

エディターでwpa_supplicant.confを編集する。

```
#wpa_supplicant.conf

country=JP
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

network={
     key_mgmt=WPA-PSK
     ssid="アクセスポイント名"
     psk="パスワード"
}
```

## Raspberry Pi Zeroの起動と設定変更

Raspberry Pi ZeroをUSB（2つある右側、基盤上にPWRと印刷してある方）でモバイルバッテリーと接続する。Bootが終わるのを待って（初回は少し時間がかかる。LEDが点灯状態になるまで待つ）、ターミナルからSSHでログインすると、Raspberry Py Zeroのコンソールに入ることができる。

## raspi-configの設定

タイムゾーンを変更する。

```
pi@raspberrypi:~ $ sudo raspi-config nonint do_change_timezone Asia/Tokyo
```

cameraの使用を許可する。

```
pi@raspberrypi:~ $ sudo raspi-config nonint do_camera 0
```

SD Cardのファイルシステムを拡張する。

```
pi@raspberrypi:~ $ sudo raspi-config nonint do_expand_rootfs

Welcome to fdisk (util-linux 2.33.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): Disk /dev/mmcblk0: 14.4 GiB, 15476981760 bytes, 30228480 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x0eadc62a

Device         Boot  Start      End  Sectors  Size Id Type
/dev/mmcblk0p1        8192   532479   524288  256M  c W95 FAT32 (LBA)
/dev/mmcblk0p2      532480 30228479 29696000 14.2G 83 Linux

Command (m for help): Partition number (1,2, default 2): 
Partition 2 has been deleted.

Command (m for help): Partition type
   p   primary (1 primary, 0 extended, 3 free)
   e   extended (container for logical partitions)
Select (default p): Partition number (2-4, default 2): First sector (2048-30228479, default 2048): Last sector, +/-sectors or +/-size{K,M,G,T,P} (532480-30228479, default 30228479): 
Created a new partition 2 of type 'Linux' and of size 14.2 GiB.
Partition #2 contains a ext4 signature.

Command (m for help): 
Disk /dev/mmcblk0: 14.4 GiB, 15476981760 bytes, 30228480 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x0eadc62a

Device         Boot  Start      End  Sectors  Size Id Type
/dev/mmcblk0p1        8192   532479   524288  256M  c W95 FAT32 (LBA)
/dev/mmcblk0p2      532480 30228479 29696000 14.2G 83 Linux

Command (m for help): The partition table has been altered.
Syncing disks.

```

https://qiita.com/Yoshiyuki-Ono/items/f3d838c62762e9571080に従ってRTCが使える状態にする。

motionをインストールする。

```
pi@raspberrypi:~ $ sudo apt install motion
```

motionの設定ファイル（/etc/motion/motion.conf）を編集する。

daemon off　→　daaemon.on

norm 0　　→　　norm 1

width 320　　→　　width 640

height 240　　→　　height 480

framerate 2　　→　　framerate 20

event_gap 60　　→　　event_gap 10

stream_localhost on　　→　　stream_localhost off

