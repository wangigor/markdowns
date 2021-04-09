# VIM命令

## 可视块编辑模式VISUAL BLOCK

> 总会有这样的事情发生：需要将下列多行字符进行「包裹」，前后增加双引号、括号等。
>
> ```test
> xxxxxxxxxxxx
> xxxxxxxxxxxx
> xxxxxxxxxxxx
> xxxxxxxxxxxx
> xxxxxxxxxxxx
> xxxxxxxxxxxx
> xxxxxxxxxxxx
> xxxxxxxxxxxx
> xxxxxxxxxxxx
> xxxxxxxxxxxx
> xxxxxxxxxxxx
> xxxxxxxxxxxx
> xxxxxxxxxxxx
> xxxxxxxxxxxx
> xxxxxxxxxxxx
> xxxxxxxxxxxx
> xxxxxxxxxxxx
> xxxxxxxxxxxx
> xxxxxxxxxxxx
> ```

- 跳到第一行 「**gg**」
- 进入列编辑模式 「**control + v**」
- 跳到最后一行 「**shift + g**」
- 进入编辑模式「**shift + i**」
- 输入需要添加的前缀字符，比如 「"」
- 退出「**esc**」

添加后缀的操作也是一样。但是会遇到每一行「数据长度不相等的情况」，可以考虑「替换换行符」。

- **shift + :**
- **%s/\n/",\r/g** 全局把替换每一行的「\n」替换为「",\r」

如果行数太多，需要复制进剪贴板。

- 跳到第一行「**gg**」
- 进入行编辑模式「**shift + v**」
- 跳到最后一行「**shift + g**」
- **shift + :**
- **+y 回车**

