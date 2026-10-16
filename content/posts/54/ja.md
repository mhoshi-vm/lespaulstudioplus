---
title: "Broadwell CPU で ESXi 9.1 の NVMe メモリティアリングが VM で利かない現象"
date: "2026-10-17T19:00:00+09:00"
tags: ["vSphere", "ESXi", "Memory Tiering", "NVMe", "Broadwell"]
thumbnail: img_2.png
---
ESXi 9.1 で NVMe メモリティアリングを有効にしたのに、VM 側では一向にティアリングされない。原因を追いかけたら、CPU 世代とモニターモード (CPL0 / ULM) に行き着きました。
<!--more-->

## 症状

Xeon E5-2697 v4 (Broadwell) のホームラボ ESXi 9.1.1 で、NVMe メモリティアリングを構成しました。
ホストから見ると、ちゃんと DRAM + NVMe の 2 ティアになっています。
ところが、どんだけ Tier 0 が溢れようが Tier 1 が使われません。

![img.png](img.png)

その後、ESXiから VM の `vmware.log` をみたら以下が記録されていました。

```
MemSched: enableTiering: 0
```

`memstats -r vmtier-stats` でも全 VM が `isTiered = n`。vmkernel.log にはこう出ています。

```
VmMemNvme: vm NNN: 473: Sw tiering disabled in VM: IsSwTieringEnabled (0) and NVMe present (1), Cancelling setup
```

この中身だと Memory Tiering ホスト上で有効でも仮想マシンでは有効になっていないようです。


## 原因

そして全 VM のログには `Monitor Mode: CPL0` と出ていました。
これが原因のようです。Memory Tiering を有効にするには、Monitoring Mode がULM になっていないといけないようです。
なんで CPL0 なのかは結局は定かではないのですが、Broadwell という古いCPUを使ったことが原因のようです。

## 直し方

***以下は公式の手順ではないため、自己責任で実行してください。***

ESXi で以下を実行します。

```
echo 'monitor.ulm = "TRUE"' >> /etc/vmware/config
/sbin/auto-backup.sh
```

その後、各VM を完全な停止（restartではだめ）を経て起動し直すと設定が有効になります。

これを行った結果初めて Tier 1 メモリが使われました。

![img_1.png](img_1.png)

正直、これを設定するデメリットはわかっていないのでくれぐれも自己責任でお願いします。

## まとめ

古いハードウェアで、Memory Tiering を設定する場合注意がいりそうです。