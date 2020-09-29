---
title: openwrt设置ddns
date: 2020-09-27 20:10:13
tags:
- openwrt

categories:
- linux
---

随着IPv6的大面积普及，我们普通用户也可以拥有“外网”IP了。有了外网IP，可以在任何支持IPv6的设备上访问我们的“服务”。现在有个问题，分配的IPv6地址不是固定的，可能过段时间就变了，如果能自动把整个地址更新到域名提供商处，即DDNS。我们就可以自由访问我们的服务了。

我的域名是在腾讯云处注册的，其它类似操作。

我从外部找到并修改的脚本,xxxx自动替换成自己的配置。

```
#!/bin/ash


domain='xxxx'
subDomain='xxxx'
recordId='xxxx'

sId='xxxx'
sKey='xxxx'
#
# Get record list first, then find out record id whatever you want to update
#
function get_record_list()
{

        signatureMethod='HmacSHA1'
        timestamp=`date +%s`
        nonce=`head -200 /dev/urandom | gnu-cksum | cut -f2 -d" "`
        region=bj
        url="https://cns.api.qcloud.com/v2/index.php"
        action='RecordList'

        src=`printf "GETcns.api.qcloud.com/v2/index.php?Action=%s&Nonce=%s&Region=%s&SecretId=%s&SignatureMethod=%s&Timestamp=%s&domain=%s" $action $nonce $region $sId $signatureMethod $timestamp $domain`


        signature=`echo -n $src|openssl dgst -sha1 -hmac $sKey -binary |base64`

        params=`printf "Action=%s&domain=%s&Nonce=%s&Region=%s&SecretId=%s&Signature=%s&SignatureMethod=%s&Timestamp=%s" $action $domain $nonce $region $sId "$signature" $signatureMethod $timestamp `


        curl -G -s -d "$params" --data-urlencode "Signature=$signature" "$url"
}
#
# $1 ipv6
# You provide a new ipv6 address to update your subDomain record
#
function modify_record()
{
        signatureMethod='HmacSHA1'

        timestamp=`date +%s`
        nonce=`head -200 /dev/urandom | cksum | cut -f2 -d" "`
        region=bj

        url="https://cns.api.qcloud.com/v2/index.php"
        #设置新ipv6
        action='RecordModify'
        recordType='AAAA'
        recordLine='默认'

        value=$1

        timestamp=`date +%s`
        nonce=`head -200 /dev/urandom | cksum | cut -f2 -d" "`

        src=`printf "GETcns.api.qcloud.com/v2/index.php?Action=%s&Nonce=%s&Region=%s&SecretId=%s&SignatureMethod=%s&Timestamp=%s&domain=%s&recordId=%s&recordLine=%s&recordType=%s&subDomain=%s&value=%s" $action $nonce $region $sId $signatureMethod $timestamp $domain $recordId $recordLine $recordType $subDomain $value`

        signature=`echo -n $src|openssl dgst -sha1 -hmac $sKey -binary |base64`

        params=`printf "Action=%s&Nonce=%s&Region=%s&SecretId=%s&SignatureMethod=%s&Timestamp=%s&domain=%s&recordId=%s&recordLine=%s&recordType=%s&subDomain=%s&value=%s" $action $nonce $region $sId $signatureMethod $timestamp $domain $recordId $recordLine $recordType $subDomain $value`

        result=`curl -G  -s -d "$params" --data-urlencode "Signature=$signature" "$url"`
        echo $result
}

#
#
# we only get the first ipv6 address that owned by interface
#
get_ipv6()
{
        ret=`ip a  | grep 2409 | tail -n1 | awk '{print $2}' | awk -F'/' '{print $1}'`
        echo $ret
}

#
# Main Entry
#
ipv6=$(get_ipv6)
echo "Assign $ipv6 to xxxx"

modify_record "$ipv6"
```

获取IPv6的函数需要自己根据情况修改。

给openwrt添加cron任务，让脚本每隔5分钟自动执行，实现DDNS功能。

```
root@OpenWrt:~# crontab -l
*/5 * * * * /root/ddns.sh
```


