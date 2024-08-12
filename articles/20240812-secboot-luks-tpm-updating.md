---
title: "セキュアブート+LUKS+TPM2でカーネルアップデート時の鍵更新を自動化する"
emoji: "🔐"
type: "tech"
topics: ["Linux", "Arch Linux", "LUKS", "tpm", "tpm2"]
published: true
---

## はじめに

LinuxではLUKSを使用することでフルディスク暗号化を実現できます。さらに、TPMに鍵を保存することで起動時に自動で復号することができます[^auto-decryption]。

[^auto-decryption]: ここでは自動で復号するよう設定することの是非については議論しません。

TPMに鍵を保存するとき、同じPCでも別のOS[^limit-decryption]からは復号用の鍵を読み取れないようにしたいと考えるかもしれません。そのような場合にはTPMへの鍵設定時にPCR[4]を設定することで特定のカーネルでない場合は鍵を読み取れないようにします（具体的な話は後述します）。

[^limit-decryption]: マルチブート環境やUSBメモリを使用したライブ環境のようなものを想定しています。

しかしこのような設定をした場合は、カーネルのアップデートのたびにそのカーネルに合わせてTPMの鍵を更新する必要があります。この手順は具体的にはカーネル更新、再起動（ここは自動で復号できないためパスワードを入力）、TPM鍵更新のようになり、煩雑な手動の操作が必要となってしまいます。

そこでこの手順を自動化できないか調べたところ、自分の構成ではうまく設定できました。他の環境では構成に合わせた調整が必要になりますが、考え方だけでも共有できたらと考えこの記事を作成しました。

:::message
この方法を各環境に適用するためには記事の内容を理解して、セキュアブートの設定方法、initramfsの作成方法、ブートローダ、TPMへの鍵設定時のPCR指定などの構成に合わせる必要があります。
:::

## 想定読者

* セキュアブート+LUKS+TPM2の環境でTPMにLUKS復号用の鍵を使用している
* 下の動作環境と自分の環境の構成の差が把握できている
* カーネルアップデートのたびに手動でTPMの鍵を更新するのは面倒だと感じている

セキュアブート、LUKS、TPMなどは設定方法がそれぞれいくつもあるため、この記事ではTPMの鍵更新の自動化に絞って解説します。

## 動作環境

以下の環境で動作確認をしました。はじめにに書いたように、このあたりの構成が変わると設定方法が変わってきます。

| 項目           | 構成                                                                  |
|----------------|-----------------------------------------------------------------------|
| ディストロ     | Arch Linux                                                            |
| initramfs      | [mkinitcpio][mkinitcpio][^initramfs]、[UKI][uki][^uki]使用            |
| セキュアブート | [sbctlを使用して鍵をUEFIに登録][sbctl][^secboot]                      |
| ブートローダー | [systemd-boot][systemd-boot][^bootloader]                             |
| ディスク暗号化 | [LUKSを使用したフルディスク暗号化][luks][^luks]                       |
| TPMへの鍵設定  | [systemd-cryptenroll][systemd-cryptenroll][^tpm]、PCR[4]+PCR[7]を設定 |

[mkinitcpio]: https://wiki.archlinux.org/title/Mkinitcpio
[uki]: https://wiki.archlinux.org/title/Unified_kernel_image
[sbctl]: https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot#Assisted_process_with_sbctl
[systemd-boot]: https://wiki.archlinux.org/title/Systemd-boot
[luks]: https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#LUKS_on_a_partition
[systemd-cryptenroll]: https://wiki.archlinux.org/title/Systemd-cryptenroll#Trusted_Platform_Module

[^initramfs]: Arch Linuxでデフォルトで使用されるinitramfs生成ツールです。Arch Linux以外だと[dracut][dracut]がよく使われているようです。
[^uki]: unified kernel imageの略で、通常は別になっているvmlinuz、initramfs、カーネルパラメーターを1つのUEFIアプリケーション（.efiファイル）にまとめたものです。セキュアブート使用時に起動に必要なもの一式をまとめて署名できるのが利点です。今回の環境ではmkinitcpioを使用して生成していますが、他にも独立したツールを使用して生成する方法もあります。
[^secboot]: Linuxのセキュアブート設定方法のうち、PC内で生成した鍵でカーネルやブートローダーを署名した上で、その鍵をUEFIに登録して検証する方法です。sbctlを使うと簡単に設定できます。他にも一般的なPCにあらかじめ設定されている[Microsoftの鍵で署名されたブートローダー（PreLoaderやshim）を使う方法][presigned-loader]もあります。
[^bootloader]: ブートローダーを使用しても鍵更新が自動化できるのか確認するため、また将来マルチブートしてもそのまま使えるようにsystemd-bootを採用しています。個人的にはシングルブート前提なら[EFISTUB][efistub]が好みです。というかGRUB以外のブートローダーがもっと広まっても良いと思っています。
[^luks]: LUKSを使用してルートファイルシステムを暗号化するフルディスク暗号化をしています。Arch Wikiに[色々な構成がまとまっていて][luks-overview]参考になります。試したことはなくかつこの記事の範囲外ですが、フルディスク暗号化以外のデータ暗号化方法もあります。
[^tpm]: 他にも[Clevis][clevis]を使う方法もあるようです。

[dracut]: https://github.com/dracutdevs/dracut
[presigned-loader]: https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot#Using_a_signed_boot_loader
[efistub]: https://wiki.archlinux.org/title/EFISTUB
[luks-overview]: https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#Overview
[clevis]: https://wiki.archlinux.org/title/Clevis

パーティショニングはこんな感じです。ディスクデバイスはnvme0n1のみでLVMなどは使用せず、EFIシステムパーティション（nvme0n1p1）と暗号化したルートパーティション（nvme0n1p2）のみのシンプルな構成です。この記事の内容には影響しませんがbtrfsを使用しています。

```shell-session
$ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
nvme0n1     259:0    0 931.5G  0 disk
├─nvme0n1p1 259:1    0   512M  0 part  /efi
└─nvme0n1p2 259:2    0   931G  0 part
  └─root    254:0    0   931G  0 crypt /swap
                                       /@
                                       /
```

## 手動での鍵更新

まずは手動でTPMの鍵を設定/更新するコマンドから見ていきます。`--wipe-slot tpm2`オプションでLUKSの鍵スロットから既存のTPMの鍵を削除してから新しい鍵を登録するよう指定します。`--tpm2-device auto`オプションでTPM2にLUKS復号用の鍵を設定するよう指定します。コマンドを実行するとディスクのパスワード入力を要求されます。以下のコマンド実行例では鍵は上書きされていません。

```shell-session
# systemd-cryptenroll --wipe-slot tpm2 --tpm2-device auto --tpm2-pcrs 4+7 /dev/nvme0n1p2
🔐 Please enter current passphrase for disk /dev/nvme0n1p2:
This PCR set is already enrolled, executing no operation.
```

ここでもし`--tpm2-pcrs 4+7`オプションによるPCRの指定がないと、同じPCなら別のOSでもディスクを復号できてしまいます。その対策としてPCRを指定します。

PCRはTPM2が持っているメモリで、UEFIファームウェアのバージョンや設定、ブートローダー、カーネルといった起動プロセスに従い特定の値を持ちます。PCRは意図して特定の値を持たせることができないように設計されています。`--tpm-pcrs 4+7`オプションを指定することで、現在のPCR[4]、PCR[7]の値と同じ値を持つ場合にしかパスワードを読みだせないようにできます。

PCRはPCR[0]からPCR[7]はUEFIで決められたの使用方法、PCR[8]からPCR[15]はブートローダーやOSごとに決められたの使用方法となっています。Linuxでの使い方については[systemd-cryptenrollのマニュアル][man-systemd-cryptenroll]や[Linux TPM PCR Registry][linux-pcr]にまとまっています。

[man-systemd-cryptenroll]: https://www.freedesktop.org/software/systemd/man/latest/systemd-cryptenroll.html
[linux-pcr]: https://uapi-group.org/specifications/specs/linux_tpm_pcr_registry/

今回はPCR[4]で読み込まれたすべてのブートローダーとカーネル、PCR[7]でセキュアブートの設定が変わっていない場合のみ復号できるようにしています。

## 自動化の課題

上記鍵更新コマンドを自動化するには以下が課題になります。

1. パスワードの入力が要求される
1. PCR[4]はカーネル更新後に再起動しないと更新されない
   つまり再起動してからTPMの鍵を更新する必要がある。この再起動時はPCR[4]が新しいカーネルに合わせて変わってしまっているので、ディスクの復号のためにパスワードの入力が必要になる。
1. 意図しないPCR値で設定されてしまうことがある
   メンテナンスのため一時的にセキュアブートを無効化して、USBメモリのライブ環境からUKIを生成するような状況を想定している。`--tpm-pcrs`オプションは現在のPCR値で設定するため、セキュアブートを無効化した状態のPCR[7]などが設定されてしまう。

このうち1と3の対処はそこまで難しくありません。1は[キーファイル][luks-keyfile]を作成して/root.keyなどに置き、systemd-cryptenrollの`--unlock-key-file /root.key`オプションでそのファイルを指定すればパスワードの代わりとして使えます。3はあらかじめ信頼するPCR値を確認しておいて、`--tpm-pcrs 4:sha256=<PCR4>+7:sha256=<PCR7>`のように直接指定すれば良いです（`<PCR4>`と`<PCR7>`はPCR値を16進文字列で指定します）。ついでに使用するハッシュ関数も明示的に指定しています。

[luks-keyfile]: https://wiki.archlinux.org/title/Dm-crypt/Device_encryption#Keyfiles

PCR[4]については計算が必要になるため、次で解説します。

## PCR

PCR[4]の具体的な計算方法について以下の仕様を見るとわかります。1つ目のリンクがTPM2チップの仕様、2つ目のリンクはそのチップをPCでどう使うかについての仕様となっています。

https://trustedcomputinggroup.org/resource/tpm-library-specification/
https://trustedcomputinggroup.org/resource/pc-client-specific-platform-firmware-profile-specification/

現在のPCRの値はArch Linuxだとtpm2-toolsパッケージのtpm2コマンドで確認できます。実行にはroot権限かtssグループへの所属が必要です。

以下が今回の構成で確認したPCR出力例です。全ビットが0や1でなかったデータはXでマスクしています。使用可能なハッシュ関数ごとにPCR値がありますが、ここではSHA256のみが使用可能であることがわかります。

```shell-session
# tpm2 pcrread
  sha1:
  sha256:
    0 : 0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    1 : 0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    2 : 0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    3 : 0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    4 : 0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    5 : 0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    6 : 0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    7 : 0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    8 : 0x0000000000000000000000000000000000000000000000000000000000000000
    9 : 0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    10: 0x0000000000000000000000000000000000000000000000000000000000000000
    11: 0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    12: 0x0000000000000000000000000000000000000000000000000000000000000000
    13: 0x0000000000000000000000000000000000000000000000000000000000000000
    14: 0x0000000000000000000000000000000000000000000000000000000000000000
    15: 0x0000000000000000000000000000000000000000000000000000000000000000
    16: 0x0000000000000000000000000000000000000000000000000000000000000000
    17: 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
    18: 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
    19: 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
    20: 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
    21: 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
    22: 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
    23: 0x0000000000000000000000000000000000000000000000000000000000000000
  sha384:
  sm3_256:
```

PCR[4]はまず起動時に全ビット0か1で初期化されます[^pcr_init]。それ以降はPCR[4]を更新するイベントの発生時に以下のExtend操作で更新されます。$H_{alg}$は使用するハッシュ関数、$||$はバイト列の連結、$digest$はイベントのデータを表します。

$$
PCR_{new} = H_{alg}(PCR_{old} || digest)
$$

[^pcr_init]: ここは少し怪しいです。[TPM 2.0 Library][tpm2]にはdefault initial conditionと表現されていて、0と1のどちらなのか、また初期状態を変更する方法があるのかについては見つけられませんでした。自分の環境では0でした。

[tpm2]: https://trustedcomputinggroup.org/resource/tpm-library-specification/

実際に発生したイベントは/sys/kernel/security/tpm0/binary_bios_measurementsから読み出すことができます。このデータはバイナリで、`tpm2 eventlog`で変換することでYAML形式で表示できます。初期値からPCRIndexが4の全イベントのDigestに対して順にExtend操作を繰り返すことで、最終的なPCR[4]値が計算できます。

以下が今回の構成で確認したイベントログ出力結果例です。PCR[4]ではない部分は省略しています。またデータは一部Xでマスクしています。

```shell-session
# tpm2 eventlog /sys/kernel/security/tpm0/binary_bios_measurements 2>/dev/null
---
version: 1
events:
(省略)
- EventNum: 17
  PCRIndex: 4
  EventType: EV_EFI_ACTION
  DigestCount: 1
  Digests:
  - AlgorithmId: sha256
    Digest: "3d6772b4f84ed47595d72a2c4c5ffd15f5bb72c7507fe26f2aaee2c69d5633ba"
  EventSize: 40
  Event: |-
    Calling EFI Application from Boot Option
(省略)
- EventNum: 22
  PCRIndex: 4
  EventType: EV_SEPARATOR
  DigestCount: 1
  Digests:
  - AlgorithmId: sha256
    Digest: "df3f619804a92fdb4057192dc43dd748ea778adc52bc498ce80524c014b81119"
  EventSize: 4
  Event: "00000000"
(省略)
- EventNum: 29
  PCRIndex: 4
  EventType: EV_EFI_BOOT_SERVICES_APPLICATION
  DigestCount: 1
  Digests:
  - AlgorithmId: sha256
    Digest: "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
  EventSize: XX
  Event:
    ImageLocationInMemory: 0xXXXXXXXX
    ImageLengthInMemory: XXXXXXXX
    ImageLinkTimeAddress: 0xXXXXXXXXX
    LengthOfDevicePath: XX
    DevicePath: 'XXXXXXXX'
(省略)
- EventNum: 32
  PCRIndex: 4
  EventType: EV_EFI_BOOT_SERVICES_APPLICATION
  DigestCount: 1
  Digests:
  - AlgorithmId: sha256
    Digest: "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
  EventSize: XX
  Event:
    ImageLocationInMemory: 0xXXXXXXXX
    ImageLengthInMemory: XXXXXXXX
    ImageLinkTimeAddress: 0xXXXXXXXXX
    LengthOfDevicePath: XX
    DevicePath: 'XXXXXXXX'
(省略)
pcrs:
  sha256:
    0  : 0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    1  : 0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    2  : 0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    3  : 0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    4  : 0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    5  : 0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    6  : 0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    7  : 0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    9  : 0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    11 : 0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

イベントはEV_SEPARATORで区切られていて、最初からEV_SEPARATORまではUEFIファームウェアが記録するため通常の使用時は基本的に固定です。残りのEV_EFI_BOOT_SERVICES_APPLICATIONが起動したブートローダーとカーネルのログです。

EV_EFI_BOOT_SERVICES_APPLICATIONのDigestはWindowsのコード署名で使用されるハッシュ[^authenticode_hash]と同じ方法で計算します。署名データを同一ファイル内に含むため、署名の有無でハッシュ値が変化しないようになっています。この計算を自前で実装するのは大変なので既存のコマンドやライブラリを使うと良いでしょう。下のコードでは[pesign][pesign]を使っています[^pe_hash]。

[^authenticode_hash]: UEFIアプリケーションの形式はWindowsと同じPE形式です。[Windows Authenticode Portable Executable Signature Format][authenticode]のCalculating the PE Image Hashにハッシュ計算方法があります。
[^pe_hash]: 今回使えそうなコマンドを探しましたが署名はできてもハッシュ表示ができないコマンドも多いです。Arch Linuxのパッケージから探したため今回は不採用でしたが、[readpe][readpe]のpehashコマンドも使いやすそうです。

[pesign]: https://github.com/rhboot/pesign
[authenticode]: https://download.microsoft.com/download/9/c/5/9c5b2167-8017-4bae-9fde-d599bac8184a/Authenticode_PE.docx
[readpe]: https://github.com/mentebinaria/readpe

## 実装

### サンプルコード

以上で必要な情報は集まったので実装をします。以下のようなスクリプトを作成しました。ここでは[mkinitcpioのpostフック][mkinitcpio-post-hook]を使用してUKI生成後に自動で鍵更新処理を実行するようにしています。環境ごとのパラメータは上で定義しています。

[mkinitcpio-post-hook]: https://wiki.archlinux.org/title/Mkinitcpio#Post_hooks

:::message
ここでは実装をシンプルにするため、TPMで復号できるのは`kernel`で設定した1つのカーネルだけとしています。
:::

```bash:/etc/initcpio/post/enroll-luks-tpm2
#!/bin/bash

set -eu

uki="$3"

# common PCR configuration
hash=sha256
hash_size=32
hasher() {
    cksum --algorithm "$hash" --untagged | sed 's/ .*//'
}

# PCR[4] configuration
pcr4_init="$(head -c "$hash_size" /dev/zero | sed 's/./00/g')"
base_digests=(
    3d6772b4f84ed47595d72a2c4c5ffd15f5bb72c7507fe26f2aaee2c69d5633ba
    df3f619804a92fdb4057192dc43dd748ea778adc52bc498ce80524c014b81119
)
loaders=(
    /efi/efi/systemd/systemd-bootx64.efi
)
kernel=/efi/efi/arch/linux.efi

# other PCR configuration
other_pcrs=(
    7:$hash=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
)

# storage configuration
disk=/dev/disk/by-uuid/XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
key_file=/root.key

pcr_extend() {
    printf '%s%s' "$1" "$2" | xxd -revert -plain | hasher
}

pe_hash() {
    pesign --hash --digest-type "$hash" --in "$1" | sed 's/ .*//'
}

emulate_pcr4() {
    local pcr4="$pcr4_init"
    local d
    for d in "${base_digests[@]}"; do
        pcr4="$(pcr_extend "$pcr4" "$d")"
    done
    local l
    for l in "${loaders[@]}"; do
        pcr4="$(pcr_extend "$pcr4" "$(pe_hash "$l")")"
    done
    pcr_extend "$pcr4" "$(pe_hash "$kernel")"
}

enroll_tpm2_key() {
    local pcrs="4:$hash=$1"
    pcrs+="$(printf '+%s' "${other_pcrs[@]}")"
    systemd-cryptenroll --unlock-key-file "$key_file" --wipe-slot tpm2 --tpm2-device auto --tpm2-pcrs "$pcrs" "$disk"
}

if [[ "$uki" != "$kernel" ]]; then
    echo 'not target of LUKS tpm2 key'
    exit
fi

pcr4="$(emulate_pcr4)"
enroll_tpm2_key "$pcr4"
```

`pcr_extend`関数がPCRのExtend操作、`emulate_pcr4`関数がPCR[4]をエミュレートして表示する関数です。EV_SEPARATORまでは`base_digests`、ブートローダーは`loaders`、カーネルは`kernel`で設定したものを使用します。`base_digests`は値をそのまま、`loaders`と`kernel`はファイルのハッシュを`pesign`コマンドを使用して取得し、その値でExtendしています。計算したカーネル更新後のPCR[4]を使用して、`enroll_tpm2_key`関数でTPMの鍵を更新します。

### フック動作順

:::message
ここは特に今回の構成固有の話になります。
:::

mkinitcpioの場合、initramfsやUKIの生成後に/usr/lib/initcpio/postと/etc/initcpio/postに配置した実行ファイルが順次実行されます。今回の環境では以下のフックが存在します。

1. /usr/lib/initcpio/post/sbctl
    * 生成したinitramfsまたはUKIをセキュアブート用に署名する
1. /etc/initcpio/post/enroll-luks-tpm2
    * 今回作成したTPMの鍵更新スクリプト

:::message
mkinitcpioではUKIの生成はフックではなくmkinitcpio本体の処理の一部として実行されます。
:::

:::message
フックはファイル名でソートした順に実行されるためTPMの鍵の更新をしてから署名が実行されます。しかしTPMの鍵の更新のために使用されるUKIのハッシュ値は署名の有無で変化しないような方法で計算されるためこの順の処理でも問題ありません。
:::

また、ブートローダー（systemd-bootx64.efi）はパッケージマネージャーのフックを作成して自動で署名と更新をしています[^bootloader-update]。このフックの実行順をmkinitcpioより前に調整する[^pacman-hook]ことでsystemd-bootx64.efi更新後にTPMの鍵を更新させています。

[^bootloader-update]: Arch Linuxのsystemd、sbctlパッケージでは今回の構成ではsystemd-bootの自動的な署名や配置がうまくできなかったので自分でフックを書いています。
[^pacman-hook]: mkinitcpioは90-mkinitcpio-install.hookにより実行されます。このフックもファイル名でソートされるため数字を80などと小さくすることで先に実行されます。

### 環境ごとの調整

上のコードは上記の構成で動作していますが、他の構成では調整が必要です。だいたいこのあたりについて調整が必要になるでしょう。

1. スクリプトの実行方法
    * ここではmkinitcpioのpostフック
    * 自動更新するのがこの記事の目的なので、基本的にinitramfs or UKI生成後のフックで実行する
1. 使用するPCR
    * ここではPCR[4]とPCR[7]
    * 構成やポリシーに合わせて設定
    * 例としてPCR[14]はPCR[7]と同様に固定値で実装できそう
1. ハッシュ関数
    * 最近のPCなら基本的にSHA256で問題ないはず
1. ブートローダー
    * 今回はsystemd-boot（systemd-bootx64.efi）のみ
    * 他のブートローダーを使用している場合は変更する
    * shimやPreLoaderを使用している場合は起動順に列挙する
1. カーネル
    * 今回はlinux.efi
    * UKIでない場合は合わせてPCRの変更も必要になるはず
1. カーネル複数対応
    * このスクリプトは1つのみ対応のため、複数対応のためには追加で実装が必要
    * 試してはいないが以下の実装が必要となるはず
    * sytemd-cryptenrollでカーネルごとに別の`--tpm2-seal-key-handle HANDLE`を指定する？
    * `--wipe-slot tpm2`でまとめて既存の鍵を削除するのではなく更新対象のカーネルの鍵のみ削除する
1. TPMの鍵設定コマンド
    * 今回はsystemd-cryptenroll

## まとめ

LUKSによるフルディスク暗号化をTPMの鍵で復号できるよう設定した状態で、カーネル更新時にTPMの鍵を自動で更新するよう設定することができました。今回実装したスクリプトは構成による影響が多いため実際に使用する際は構成に合わせて調整が必要です。頑張ればより汎用的な実装にもできそうですが今回はここまでとします。もし興味があったら設定してみてください。
