# Phoenix Framework 中文入门教程 - 我的 Phoenix 项目开发历程

前端开发一直都是我的主业。是的，我是在暗示你：Phoenix Framework 非常容易上手。不过也因为我一直在从事前端开发，后端的一些概念我可能理解得不够到位，如有错误，欢迎指正 😁。

教程中使用的软件版本如下：

软件|版本
|---|---|
Elixir|1.5.1
Phoenix Framework|1.3.0

如果你不清楚自己当前系统上 Elixir、Phoenix 的版本，可以执行如下两个命令进行查看：

```sh
➜  ~  elixir -v
Erlang/OTP 20 [erts-9.1] [source] [64-bit] [smp:4:4] [ds:4:4:10] [async-threads:10] [hipe] [kernel-poll:false] [dtrace]

Elixir 1.5.1
➜  ~  mix phx.new --version
Phoenix v1.3.0
```

## 目录

1. [准备工作](00-prepare.md)
2. [创建项目](01-create-project.md)
3. [Phoenix 初探](02-explore-phoenix.md)
4. [TvRecipe 项目规划](03-tv-recipe.md)
5. [注册用户](04-user-register/00-prepare.md)
    1. [username 必填](04-user-register/01-username-required.md)
    2. [username 已被人占用](04-user-register/02-username-unique.md)
    3. [username 只允许使用英文字母、数字及下划线](04-user-register/03-username-format.md)
    4. [username 限定长度值](04-user-register/04-username-length.md)
    5. [username 禁止使用 `admin` 等](04-user-register/05-username-exclude.md)
    6. [email 规则](04-user-register/06-email-rules.md)
    7. [password 规则](04-user-register/07-password-rules.md)
    8. [password 安全存储](04-user-register/08-password-storage.md)
    9. [优化用户注册界面](04-user-register/09-optimize-ui.md)
6. 会话
    1. [登录](05-session/01-login.md)
    2. [注册成功自动登录](05-session/02-auto-login-user.md)
    3. [退出登录](05-session/03-logout.md)
    4. [登录/注册按钮](05-session/04-login-logout-buttons.md)
7. [限制访问](06-restrict-access.md)
8. 菜谱
    1. [生成菜谱样板文件](07-recipe/01-gen-html.md)
    2. [菜谱属性开发](07-recipe/02-recipe-scheme.md)
    3. [菜谱控制器开发](07-recipe/03-recipe-controller.md)
    4. [菜谱视图的测试](07-recipe/04-recipe-view.md)
    5. [添加视频 url](07-recipe/05-recipe-tv-url.md)

## 捐款

如果教程对您有所帮助，欢迎扫描下方的支付宝二维码给我捐款（请备注您的 github 用户名）：

<img src="img/alipay-qr.png" alt="支付宝捐款" width="150" />

## License & Copyright

&copy; 2017 陈三

<a rel="license" href="http://creativecommons.org/licenses/by-nd/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-nd/4.0/88x31.png" /></a><br />本作品采用<a rel="license" href="http://creativecommons.org/licenses/by-nd/4.0/">知识共享署名-禁止演绎 4.0 国际许可协议</a>进行许可。