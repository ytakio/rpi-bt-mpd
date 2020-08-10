# rpi-bt-mpd

RPi zero WHをBTスピーカーを鳴らすサウンドサーバーにする。

- 音源はNAS上
- 再生とかはスマホとかsshとか
- 音はBTスピーカーで出す

ってのを定期的にやって、毎回忘れてるので書く。

# Prerequisite

- RPi Zero WH or RPi 3+以降
    - BTついてればOK
- Bluetoothスピーカー
    - Profile: A2DP
- OS: Raspberry OS 32bit (Debian Buster base)
- Option
    - MPCが使えるスマホとかタブレットとか

# WiFiとか基本設定

```bash
sudo raspi-config
# あとはCUIに従って色々設定
```

- SSIDをブロードキャストしてない場合はこんな感じで`scan_ssid=1`を追加

```bash
$ sudo cat /etc/wpa_supplicant/wpa_supplicant.conf
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=JP

network={
        ssid="xxxxxxx"
        psk="xxxxxxxxxxxxxxxxxxxxx"
        scan_ssid=1
}
```

# Packages

たしかMPDとbluealsa入れときゃOKだったような。あとsshでも再生とかできると楽なのでmpcの`ncmpcc`も入れとく。

```bash
$ sudo apt bluealsa mpd ncmpcc
```

# Bluetoothスピーカー接続

## package

```bash
$ sudo apt install bluealsa
```

`pi`ユーザーに`bluetooth`グループ付与してもできると思うけど、rootの方が自動接続とかしてくれそうなので`sudo`で設定

```
$ sudo bluetoothctl
[bluetooth]# power on       # 念の為
[bluetooth]# agent on       # 全BTデバイスとのペリング
[bluetooth]# default-agent  # これをデフォルトエージェントにする
Default agent request successful
[bluetooth]# scan on        # BTデバイス検索
Discovery started
[NEW] Device XX:XX:XX:XX:XX:XX XX-XX-XX-XX-XX-XX
[NEW] Device XX:XX:XX:XX:XX:XX XX-XX-XX-XX-XX-XX
[NEW] Device XX:XX:XX:XX:XX:XX XX-XX-XX-XX-XX-XX
[NEW] Device XX:XX:XX:XX:XX:XX XX-XX-XX-XX-XX-XX
[NEW] Device ZZ:ZZ:ZZ:ZZ:ZZ:ZZ BT Speaker
[bluetooth]# scan off       # 見つけたので停止
Discovery stopped
[bluetooth]# pair ZZ:ZZ:ZZ:ZZ:ZZ:ZZ     # ペアリング
Attempting to pair with ZZ:ZZ:ZZ:ZZ:ZZ:ZZ
[CHG] Device ZZ:ZZ:ZZ:ZZ:ZZ:ZZ Connected: yes
Request confirmation
[agent] Confirm passkey 606296 (yes/no): yes
Pairing successful
[bluetooth]# connect ZZ:ZZ:ZZ:ZZ:ZZ:ZZ  # BTデバイス接続
Attempting to connect to ZZ:ZZ:ZZ:ZZ:ZZ:ZZ
[CHG] Device ZZ:ZZ:ZZ:ZZ:ZZ:ZZ Connected: yes
Connection successful
[CHG] Device ZZ:ZZ:ZZ:ZZ:ZZ:ZZ ServicesResolved: yes
[TaoTronics TT-SK028]# trust    # 現在のBTデバイスを自動接続設定
[CHG] Device ZZ:ZZ:ZZ:ZZ:ZZ:ZZ Trusted: yes
Changing ZZ:ZZ:ZZ:ZZ:ZZ:ZZ trust succeeded
```

再起動後につながらない場合は以下を実行するとか、BTスピーカーを再起動するとかでなんとかつながるけど、なんかスクリプト入れれば何とかなるとは思う

```
$ sudo bluetoothctl
[bluetooth]# connect ZZ:ZZ:ZZ:ZZ:ZZ:ZZ
```

# ファイル共有クライアント

NASがNFS提供できるので、autofsだけでおｋ

## package

```bash
$ sudo apt install autofs
```

## settings

```bash
$ vi /etc/auto.master   # 下記行追加
/misc   /etc/auto.misc
$ vi /etc/auto.misc     # 下記行追加
media -fstype=nfs4 hoge:/Multimedia
```

# MPD

## package

```bash
$ sudo apt mpd ncmpcc
```

## settings

- 音源へリンク張る & ALSA設定
```bash
$ cd /var/lib/mpd   # ここがMPDの$HOME
$ sudo rm -rf music # デフォルトの格納ディレクトリ
$ sudo ln -s /misc/media music
$ sudo vi .asoundrc && cat .asoundrc
defaults.bluealsa {
    device "ZZ:ZZ:ZZ:ZZ:ZZ:ZZ"
}
$ sudo chown mpd:audio .asuondrc    # 多分不要
```

- MPD設定
```bash
$ sudo /etc/mpd.conf
# music_directory を 変えてもよし
# localhost以外でもアクセス出来るようにbind_to_addressをコメントアウト
audio_output {
    type            "alsa"
    name            "My ALSA Device"
    device          "bluealsa"
    mixer_type      "software"
    format          "44100:16:2"
}
$ sudo systemctl restart mpd    # MPD設定ファイルを有効化
```

- MPD DB更新
```bash
$ mpc update    # 結構かかる
```

