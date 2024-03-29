```toc

```

## markdown 语法

https://mermaid.js.org/intro/

当我需要画图时插入如下这样的一个代码块：

````
```mermaid
    流程图/时序图代码
```
复制代码
````

## 流程图

### 流程图布局

流程图代码以「graph 《布局方向》」开头，布局方向主要有如下几种：

-   TB，从上到下
-   TD，从上到下
-   BT，从下到上
-   RL，从右到左
-   LR，从左到右

例：

````
```mermaid
graph TB
```
````

### 流程常用符号

```mermaid
graph LR
A(起止框)
```

用法如下：

```mermaid
graph LR
B[处理框]
```

可以使用 HTML 中的实体字符。

```mermaid
graph LR
A["双引号-#quot;"]
```

```mermaid
graph LR
C{判断框}
```

```mermaid
graph LR
D((连接))
```

### 流程图连接样式

-   实线，箭头，文字

```mermaid
graph LR
A --文字描述---> B
B --> |文字| C
```

-   实线，箭头，无文字

```mermaid
graph LR
A-->B
```

-   实线，无箭头，无文字

```mermaid
graph LR
    A---B
```

-   实线，无箭头，文字

```mermaid
graph LR
    A-- 文字描述 ---B
```

-   虚线，箭头，无文字

```mermaid
graph LR;
    A-.->B;
```

-   虚线，箭头，文字

```mermaid
graph LR;
A-. 文字 .->B;
```

综合示例：

```mermaid
graph TB
    A(接口请求) --> B[参数校验]
    B[参数校验] --> C{校验通过?}
    C{校验通过?} -- 通过 --> d[处理业务逻辑]
    C{校验不通过} -- 不通过 --> e[结束]
    d[处理业务逻辑] --> e(结束)
```

## 时序图

时序图代码以「sequenceDiagram」开头。 时序图中包括如下常见元素：

### 参与者


```mermaid
sequenceDiagram
participant A as 别名
```

语句次序即为参与者横向排列次序。

### 消息

交互时一方对另一方的操作（比如接口调用）或传递出的信息。

-   用单向箭头来表示——实线代表主动发出消息；
-   虚线代表响应；
-   末尾带「X」代表异步消息，无需等待回应。

消息语句格式为：

```
<参与者> <箭头> <参与者> : <描述文本>。
复制代码
```

其中 <箭头> 的写法有：

```
->> 显示为实线箭头（主动发出消息）
-->> 显示为虚线箭头（响应）
-x 显示为末尾带「X」的实线箭头（异步消息）
复制代码
```

示例：

```mermaid
sequenceDiagram
    participant A as lilei
    participant B as hanmeimei
    A ->> B: How are you.
    Note left of A: 对象A的描述(提示)
    B -->> A: Fine, Thank you.
    Note right of B: 对象B的描述
    A -x B: 我走了
```


### 激活框

从消息接收方的时间线上标记一小段时间，表示对消息进行处理的时间间隔。

```
<参与者> <箭头> [+/-]: <描述文本>。
```

示例：

```mermaid
sequenceDiagram
    老板 ->> + 员工:  今年年终奖翻倍
    员工 -->> - 老板: 持续鼓掌
```



### 注解

```mermaid
sequenceDiagram
    Note left of 老板A : 我脸盲
    Note right of 老板B : 我对钱没兴趣
    Note over 老板A,老板B : 996走起来
```


### 循环

相当于编程代码中的 while 循环 循环格式为：

```
loop 循环的描述
    消息
end
复制代码
```

示例：

```mermaid
sequenceDiagram
    用户 ->> + 网站 : 账号实名认证
    网站 -->> - 用户 : 资料提交成功，等待审核

    loop 一天七次
        用户 ->> + 网站 : 查看审核进度
        网站 -->> - 用户 : 审核中
    end
```



### 选择(alt)

相当于 if 及 else if 语句。或者理解为 switch 也可以

```mermaid
sequenceDiagram
    学生 ->> 学校 : 查询成绩
    学校 -->> 学生 : 成绩

    alt 成绩 > 550
        学生 ->> 学校 : 可以上一本了
    else 400 < 成绩 < 550
        学生 ->> 学校 : 上个二本吧
    else 余额 > 300
        学生 ->> 学校 : 考虑下专科吧
    end
```



### 可选

相当于单个分支的 if 语句。

```mermaid
sequenceDiagram
    美女 ->> 帅哥 : 我想找个高富帅

    opt 我就是
        帅哥 -->> 美女 : 加个微信吧
    end
```


### 并行

```mermaid
sequenceDiagram
    老板 ->> 员工 : 开始实行996

    par 开始摸鱼
        员工 ->> 员工 : 刷微博
    and
        员工 ->> 员工 : 听音乐
    end

    员工 -->> 老板 : 9点下班
```


## 大小调整

```mermaid
%%{init:{'theme':'base', 'fontSize': '10px'}}%%
sequenceDiagram
	participant A as PC
	participant B as CTCORE
	participant C as PSCORE
	A ->> B: 创建朋友圈
	B ->> C: 创建朋友圈
	C -->> C: 查询当前userid当天是否已经创建过？
	C -->> B: 当天已创建，不能再次创建
	B -->> A: 返回失败，当天已创建，不能再次创建
	C -->> C: 当天未创建，则创建朋友圈任务
	C -->> B: 返回成功
	B -->> A: 返回成功
```


