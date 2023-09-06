
*基本使用*

```bash
curl -H "Content-Type:application/json" -X POST -d '{"v":"1.0.0","auth":{"appId":"wbb8bcvq","nonce":"ealZaCcuuU6JaCHRfJbBNRsVx4BmqzSV"},"arg":{"site":"eac-android-wework","eacId":"eac-f2gzjmaz1473nndc","width":720,"height":1280}}' -H "DECSHASH:27F48D8448AAC7E4C80652832C91295BA0F998B930FCA22925DBA6349F8EEA77" https://testing-vdi.byodonline.com/iapi/das/describe-eac-url -x 10.107.120.241:9090
```

说明：

`-H`：指定头里面的信息，也就是（`key:value`），如果有多个，可以使用多个进行指定
`-X`：指定请求方式
`-d`：指定要发送的报文
`-x`：指定代理地址

可以使用 `-v` 打印详细信息，可以放在最前面，如 `curl -v ...`

*上传文件*

```bash
curl -H "Content-Md5:kNDGWl6s8JtzJTx+tUT2xQ==" -X PUT -H "Content-Type:application/octet-stream" --data-binary "@/data/tomcat/appsystems/files/ewc-ewfront/wxwork/25000000178649732107201154529941000000016917165210723470" "https://testeac-1308262583.cos.ap-guangzhou.myqcloud.com/473474580250392736?q-sign-algorithm=sha1&q-ak=AKIDoRpxoOilX2GEuJRIsBDySfrnTszpOggP&q-sign-time=1691742441%3B1691743641&q-key-time=1691742441%3B1691743641&q-header-list=content-md5%3Bhost&q-url-param-list=&q-signature=cc18542987abd769bfacd8e90d93991a4ebabb8b"
```

这里 `Content-Type:application/octet-stream` 表示无类型的二进制流。`--data-binary` 指定文件，这里 `@` 跟着文件地址。