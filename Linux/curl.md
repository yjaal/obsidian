

```bash
curl -H "Content-Type:application/json" -X POST -d '{"v":"1.0.0","auth":{"appId":"wbb8bcvq","nonce":"ealZaCcuuU6JaCHRfJbBNRsVx4BmqzSV"},"arg":{"site":"eac-android-wework","eacId":"eac-f2gzjmaz1473nndc","width":720,"height":1280}}' -H "DECSHASH:27F48D8448AAC7E4C80652832C91295BA0F998B930FCA22925DBA6349F8EEA77" https://testing-vdi.byodonline.com/iapi/das/describe-eac-url -x 10.107.120.241:9090
```

说明：

`-H`：指定头里面的信息，也就是（`key:value`），如果有多个，可以使用多个进行指定
`-X`：指定请求方式
`-d`：指定要发送的报文
`-x`：指定代理地址
