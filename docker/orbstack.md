
```
brew install --cask orbstack
```


```
orb start
orb restart docker
```


配置国内镜像源，如果不起作用，可以配置代理
```
{
  "registry-mirrors": [
    "https://5qckqf8j.mirror.aliyuncs.com",
    "https://dockerproxy.com",
    "https://docker.mirrors.ustc.edu.cn",
    "https://docker.nju.edu.cn"
  ],
  "ipv6": true,
  "proxies": {
    "http-proxy": "http://127.0.0.1:1087",
    "no-proxy": "localhost,127.0.0.0/8",
    "https-proxy": "http://127.0.0.1:1087"
  }
}
```