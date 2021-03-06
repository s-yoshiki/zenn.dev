---
title: "JSでサブネットマスク関連の計算とハマったこと"
emoji: 🌐
type: tech
topics: ["javascript","node","アルゴリズム"]
published: true
---

## 概要

JSでIPの計算を行う機会なんて殆どないかもしれませんが、(node.jsは別として)
重要な理論を整理するためにJSで実装を試みてみた際の記録です。

### ※ 注意

ここに記載しているコードは説明するための "最低限の機能" しか実装しておらず、入力値をチェックする機構等は存在しません。例外的な値を入力すると誤作動します。

## サブネットマスク関連の計算

大前提としてサブネットの計算を整理しておきます。

### IPv4アドレス文字列をNumber型に変換する

IPv4アドレスの文字列、例えば `192.168.0.1` といった形式の文字列をbit演算を利用して計算するために Number型 に変換しておきます。

```js
// IPv4 to binary string
const ip2bin = (ip) => ip.split(".").map(e => Number(e).toString(2).padStart(8, '0')).join('')
// IPv4 to Number
const ip2long = (ip) => parseInt(ip2bin(ip), 2)
// Number to IPv4
const long2ip = (num) => {
    let bin = Number(num).toString(2).padStart(32, '0')
    return [
        bin.slice(0, 8),
        bin.slice(8, 16),
        bin.slice(16, 24),
        bin.slice(24, 32),
    ].map(e => parseInt(e, 2)).join('.')
}

console.log(ip2bin("192.0.34.166")) // 11000000000000000010001010100110
console.log(ip2long("192.0.34.166")) // 3221234342
console.log(long2ip(ip2long("192.0.34.166"))) // 192.0.34.166
```

余談ですが、ip2long/long2ipはPHPで同じ名前の関数が存在します。

[ip2long - php.net](https://www.php.net/manual/ja/function.ip2long.php)

これらを参考にしました。

### CIDR と サブネットの相互変換

CIDRは2進数で表示された場合の先頭からの1の数を表しており、最大値が`32`までの整数の値となります。

そしてCIDRとサブネットマスクは次のような関係が成り立ちます。

```shell
26  # CIDR
↓
11111111.11111111.11111111.11000000 # 26個1を並べ2進数表記に変換
↓
255.255.255.192 # 1バイト(=8bit)ずつに区切ってそれぞれを10進数に変換する
```

コードにするとこのようになります。

```js
// CIDR to Number
const cidr2long = (cidr) => parseInt(String("").padStart(cidr, '1').padEnd(32, '0'), 2)
// CIDR to SubnetMask
const cidr2subnetmask = (num) => long2ip(cidr2long(num))
// SubnetMask to CIDR
const subnetmask2cidr = (ip) => ip2bin(ip).split('1').length - 1

console.log(cidr2subnetmask(26)) // 255.255.255.192
console.log(subnetmask2cidr("255.255.255.192")) // 26
```

### ネットワークアドレス と ブロードキャストアドレス

ネットワークアドレス(開始アドレス)は、IPアドレスとサブネットマスクのAND(論理積)で求められます。

また、ブロードキャストアドレス(終了アドレス)は、IPアドレスと反転したサブネットマスクのOR(論理和)で求められます。

イメージだとこんな感じです↓

```js
ip & subnetmask // ネットワークアドレス
ip | ~subnetmask // ブロードキャストアドレス
```

なので、純粋にJSのビット論理積とビット論理和で計算しようとしましたが、、、ここでハマりました。

```js
console.log(64) // 64
console.log(64 | 0) // 64
console.log(ip2long("192.168.0.1")) // 3232235521
console.log(3232235521) // 3232235521
console.log(3232235521 | 0) // -1062731775
```

ここにあるコードの5行目ように、計算中に意図せず負の値となっていました。
これは2の補数(=符号付き32ビットの整数)として表現されています。
符号付き32ビット整数は -2の32乗から2の32乗-1を表現できます。

今回は32ビットで表せる数値を全て符号なしの整数として表現したかったので、変換する方法を探したところ "符号なし右シフト演算子" で符号なしの表現に変換することができました。

```js
console.log((3232235521 | 0) >>> 0) // 3232235521
```

これを利用して、ネットワークアドレスとブロードキャストアドレスは次のように求めます。

```js
// ネットワークアドレス
const getNetworkAddr = (ip, subnetmask) => (ip & subnetmask) >>> 0
// ブロードキャストアドレス
const getBroadcastAddr = (ip, subnetmask) => (ip | ~subnetmask) >>> 0
```

### クラス

IPアドレスは使用するネットワークの規模によって A 〜 C と 特殊用としての「D」や実験用の「E」に分かれています。

 - **クラスA** 0.0.0.0 ～ 127.255.255.255
 - **クラスB** 128.0.0.0 ～ 191.255.255.255
 - **クラスC** 192.0.0.0 ～ 223.255.255.255
 - **クラスD** 224.0.0.0 ～ 239.255.255.255
 - **クラスE** 240.0.0.0 ～ 255.255.255.255

これを次のようにコードに落としました。

```js
const getClass = (ip) => {
    if (ip2long("0.0.0.0") <= ip && ip <= ip2long("127.255.255.255")) {
        return 'A'
    }
    if (ip2long("128.0.0.0") <= ip && ip <= ip2long("191.255.255.255")) {
        return 'B'
    }
    if (ip2long("192.0.0.0") <= ip && ip <= ip2long("223.255.255.255")) {
        return 'C'
    }
    if (ip2long("224.0.0.0") <= ip && ip <= ip2long("239.255.255.255")) {
        return 'D'
    }
    if (ip2long("240.0.0.0") <= ip && ip <= ip2long("255.255.255.255")) {
        return 'E'
    }
    return false;
}
console.log(getClass(ip2long("192.168.0.1"))) // C
```

### IPアドレスが指定した範囲内にあるかどうか判定

IPアドレスのネットワークアドレスが同じかどうかを比較する方法でチェックします。

```js
const inRange = (remoteIp, acceptIp, cidr) => {
    cidr = Number(cidr)
    const remoteIpNetwork = remoteIp >>> (32 - cidr)
    const acceptIpNetwork = acceptIp >>> (32 - cidr)
    return remoteIpNetwork === acceptIpNetwork
}

console.log(inRange(ip2long("192.168.0.1"), ip2long("192.168.0.254"), 24)) 
// true
```

## 改めて計算方法を整理する

改めて整理してサブネットマスク関連の一連の計算を行います。コード全て載せます。

```js
const ip2bin = (ip) => ip.split(".").map(e => Number(e).toString(2).padStart(8, '0')).join('')

const ip2long = (ip) => parseInt(ip2bin(ip), 2)

const long2ip = (num) => {
    let bin = Number(num).toString(2).padStart(32, '0')
    return [
        bin.slice(0, 8),
        bin.slice(8, 16),
        bin.slice(16, 24),
        bin.slice(24, 32),
    ].map(e => parseInt(e, 2)).join('.')
}

const cidr2long = (cidr) => parseInt(String("").padStart(cidr, '1').padEnd(32, '0'), 2)

const cidr2subnetmask = (num) => long2ip(cidr2long(Number(num)))

const subnetmask2cidr = (ip) => ip2bin(ip).split('1').length - 1

const getNetworkAddr = (ip, subnetmask) => (ip & subnetmask) >>> 0

const getBroadcastAddr = (ip, subnetmask) => (ip | ~subnetmask) >>> 0

const inRange = (remoteIp, acceptIp, cidr) => remoteIp >>> (32 - Number(cidr)) === acceptIp >>> (32 - Number(cidr))

const getClass = (ip) => {
    if (ip2long("0.0.0.0") <= ip && ip <= ip2long("127.255.255.255")) {
        return 'A'
    }
    if (ip2long("128.0.0.0") <= ip && ip <= ip2long("191.255.255.255")) {
        return 'B'
    }
    if (ip2long("192.0.0.0") <= ip && ip <= ip2long("223.255.255.255")) {
        return 'C'
    }
    if (ip2long("224.0.0.0") <= ip && ip <= ip2long("239.255.255.255")) {
        return 'D'
    }
    if (ip2long("240.0.0.0") <= ip && ip <= ip2long("255.255.255.255")) {
        return 'E'
    }
    return false;
}

const ipLong = ip2long("192.168.0.1")
const cidr = cidr2long(24)
console.log(`
IPアドレス: ${long2ip(ipLong)}
サブネットマスク: /${subnetmask2cidr("255.255.255.0")} (${cidr2subnetmask(24)})
ネットワークアドレス: ${long2ip(getNetworkAddr(ipLong, cidr))}
使用可能IP: ${long2ip(getNetworkAddr(ipLong, cidr) + 1)} 〜 ${long2ip(getBroadcastAddr(ipLong, cidr) - 1)}
ブロードキャストアドレス: ${long2ip(getBroadcastAddr(ipLong, cidr))}
アドレス数: ${getBroadcastAddr(ipLong, cidr) - getNetworkAddr(ipLong, cidr) + 1}
ホストアドレス数: ${getBroadcastAddr(ipLong, cidr) - getNetworkAddr(ipLong, cidr) - 1}
IPアドレスクラス: ${getClass(ipLong)}
`)
console.log(`192.168.0.1 は 192.168.0.254/24 に含まれ${ inRange(ip2long("192.168.0.1"), ip2long("192.168.0.254"), 24) ? 'ます' : 'ません' }`)
console.log(`192.168.1.0 は 192.168.0.254/24 に含まれ${ inRange(ip2long("192.168.1.0"), ip2long("192.168.0.254"), 24) ? 'ます' : 'ません' }`)
```

そして出力結果がこちらになります。サブネット計算サイトの結果と一致しました。

```
IPアドレス: 192.168.0.1
サブネットマスク: /24 (255.255.255.0)
ネットワークアドレス: 192.168.0.0
使用可能IP: 192.168.0.1 〜 192.168.0.254
ブロードキャストアドレス: 192.168.0.255
アドレス数: 256
ホストアドレス数: 254
IPアドレスクラス: C
192.168.0.1 は 192.168.0.254/24 に含まれます
192.168.1.0 は 192.168.0.254/24 に含まれません
```

## その他

この記事は以下の記事を一つにまとめたものです。

[JSでサブネットマスクの計算 - 404 motivation not found](https://tech-blog.s-yoshiki.com/entry/225)

[JSでIPアドレスがサブネットマスクで指定した範囲内にあるか判定する - 404 motivation not found](https://tech-blog.s-yoshiki.com/entry/228)

[JSで32ビット符号付き整数に対してのビット演算でハマった - 404 motivation not found](https://tech-blog.s-yoshiki.com/entry/229)
