
```toc
```

TextInput、TextArea是输入框组件，通常用于响应用户的输入操作，比如评论区的输入、聊天框的输入、表格的输入等，也可以结合其它组件构建功能页面，例如登录注册页面。具体用法请参考[TextInput](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/ts-basic-components-textinput-V5)、[TextArea](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/ts-basic-components-textarea-V5)。

## 创建输入框

TextInput为单行输入框、TextArea为多行输入框。通过以下接口来创建。

```ts
TextInput(value?:{placeholder?: ResourceStr, text?: ResourceStr, controller?: TextInputController})

TextArea(value?:{placeholder?: ResourceStr, text?: ResourceStr, controller?: TextAreaController})
```

- 单行输入框

```ts
TextInput()
```

- 多行输入框

```ts
TextArea()
```

多行输入框文字超出一行时会**自动折行**。

```ts
TextArea({ text: "我是TextArea我是TextArea我是TextArea我是TextArea" }).width(300)
```


## 设置输入框类型

TextInput有9种可选类型，分别为
- Normal基本输入模式
- Password密码输入模式
- Email邮箱地址输入模式
- Number纯数字输入模式
- PhoneNumber电话号码输入模式
- USER_NAME用户名输入模式
- NEW_PASSWORD新密码输入模式
- NUMBER_PASSWORD纯数字密码输入模式
- NUMBER_DECIMAL带小数点的数字输入模式

通过type属性进行设置：

- 基本输入模式（默认）
```ts
TextInput().type(InputType.Normal)
```

- 密码输入模式

```ts
TextInput().type(InputType.Password)
```


## 自定义样式

- 设置无输入时的提示文本

```ts
TextInput({placeholder:'我是提示文本'})
```


- 设置输入框当前的文本内容

```ts
TextInput({placeholder:'我是提示文本', text:'当前的文本内容'})
```


- 添加背景色

```ts
TextInput({ placeholder: '我是提示文本', text: '我是当前文本内容' })
  .backgroundColor(Color.Pink)
```

更丰富的样式可以结合[通用属性](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/ts-universal-attributes-size-V5)实现。


## 添加事件

文本框主要用于获取用户输入的信息，把信息处理成数据进行上传，绑定onChange事件可以获取输入框内改变的内容。用户也可以使用通用事件来进行相应的交互操作。

```ts
TextInput()
  .onChange((value: string) => {
    console.info(value);
  })
  .onFocus(() => {
    console.info('获取焦点');
  })
```


## 键盘避让

键盘抬起后，具有滚动能力的容器组件在横竖屏切换时，才会生效键盘避让，若希望无滚动能力的容器组件也生效键盘避让，建议在组件外嵌套一层具有滚动能力的容器组件，比如[Scroll](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/ts-container-scroll-V5)、[List](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/ts-container-list-V5)、[Grid](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/ts-container-grid-V5)。

```ts
// xxx.ets
@Entry
@Component
struct Index {
  placeHolderArr: string[] = ['1', '2', '3', '4', '5', '6', '7'];

  build() {
    Scroll() {
      Column() {
        ForEach(this.placeHolderArr, (placeholder: string) => {
          TextInput({ placeholder: 'TextInput ' + placeholder })
            .margin(30)
        })
      }
    }
    .height('100%')
    .width('100%')
  }
}
```









