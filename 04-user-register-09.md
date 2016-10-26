# 优化用户注册界面

截止上一章，我们基本完成了用户注册的所有逻辑，但界面上还有些问题需要解决。

1. 错误消息不明显

    ![Phoenix 用户名不为空的错误消息](img/04-users-blank-username.png)

    “请填写”三个字不够突出。

2. 密码输入框明文显示

    ![密码输入框](img/04-password-input.png)

第 1 个问题。Phoenix 默认生成的 `form.html.eex` 模板里使用了 Bootstrap [样式](https://getbootstrap.com/css/#forms-control-validation)：

```eex
<div class="form-group">
  <%= label f, :username, class: "control-label" %>
  <%= text_input f, :username, class: "form-control" %>
  <%= error_tag f, :username %>
</div>
```
但模板中生成的与 Bootstrap 的比，差了 `has-error` 这样的 CSS 状态类。我们可以加上：

```eex
diff --git a/web/templates/user/form.html.eex b/web/templates/user/form.html.eex
index 5857c33..8b50f25 100644
--- a/web/templates/user/form.html.eex
+++ b/web/templates/user/form.html.eex
@@ -5,7 +5,7 @@
     </div>
   <% end %>

-  <div class="form-group">
+  <div class="form-group <%= if f.errors[:username], do: "has-error" %>">
     <%= label f, :username, class: "control-label" %>
     <%= text_input f, :username, class: "form-control" %>
     <%= error_tag f, :username %>
```
这时我们的错误提示界面就会变成：

![用户名不为空](img/04-username-has-error.png)

一目了然。至于为什么生成的模板里不带 `has-error`，可以看 [github 上的一个 issue](https://github.com/phoenixframework/phoenix/issues/1961)。

第 2 个问题就好解决了，我们看现有代码：

```eex
<div class="form-group">
  <%= label f, :password, class: "control-label" %>
  <%= text_input f, :password, class: "form-control" %>
  <%= error_tag f, :password %>
</div>
```
因为生成的模板里用了 `text_input`，它是明文显示的，改为 [`password_input`](https://hexdocs.pm/phoenix_html/Phoenix.HTML.Form.html#password_input/3) 就好。

这样，我们就结束了用户注册模块。接下来，我们开始开发用户的登录/退出功能。