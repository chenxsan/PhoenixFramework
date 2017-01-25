# 优化用户注册界面

截止上一章，我们基本完成用户注册的所有逻辑，但界面上还有些问题需要解决。

1. 错误信息不明显

    ![Phoenix 用户名不为空的错误信息](../img/04-users-blank-username.png)

    截图中可以看到，“请填写”三个字不突出，很多时候用户会视而不见。

2. 密码输入框

    ![密码输入框](../img/04-password-input.png)

    在密码框中输入的内容，现在是明文显示，通常常是用 * 号代替。

第 1 个问题。

Phoenix 生成的 `form.html.eex` 模板里使用了 Bootstrap [样式](https://getbootstrap.com/css/#forms-control-validation)：

```eex
<div class="form-group">
  <%= label f, :username, class: "control-label" %>
  <%= text_input f, :username, class: "form-control" %>
  <%= error_tag f, :username %>
</div>
```
但模板中生成的样式与 Bootstrap 的比，差了 `has-error` 这样的 CSS 状态类。我们可以给它补上：

```eex
diff --git a/web/templates/user/form.html.eex b/web/templates/user/form.html.eex
index 5857c33..b047466 100644
--- a/web/templates/user/form.html.eex
+++ b/web/templates/user/form.html.eex
@@ -5,19 +5,19 @@
     </div>
   <% end %>

-  <div class="form-group">
+  <div class="form-group <%= if f.errors[:username], do: "has-error" %>">
     <%= label f, :username, class: "control-label" %>
     <%= text_input f, :username, class: "form-control" %>
     <%= error_tag f, :username %>
   </div>

-  <div class="form-group">
+  <div class="form-group <%= if f.errors[:email], do: "has-error" %>">
     <%= label f, :email, class: "control-label" %>
     <%= text_input f, :email, class: "form-control" %>
     <%= error_tag f, :email %>
   </div>

-  <div class="form-group">
+  <div class="form-group <%= if f.errors[:password], do: "has-error" %>">
     <%= label f, :password, class: "control-label" %>
     <%= text_input f, :password, class: "form-control" %>
     <%= error_tag f, :password %>
```
这样我们的错误提示界面就会变成：

![用户名不为空](../img/04-username-has-error.png)

非常醒目。至于 Phoenix 生成的模板里为什么不带 `has-error`，可以看 [github 上的一个 issue](https://github.com/phoenixframework/phoenix/issues/1961)。

第 2 个问题就容易解决了，我们来看现有代码：

```eex
<div class="form-group <%= if f.errors[:password], do: "has-error" %>">
  <%= label f, :password, class: "control-label" %>
  <%= text_input f, :password, class: "form-control" %>
  <%= error_tag f, :password %>
</div>
```
生成的模板里现在用了 `text_input`，它本来就是明文显示的，改为 [`password_input`](https://hexdocs.pm/phoenix_html/Phoenix.HTML.Form.html#password_input/3) 后，界面上就会用 * 号代替我们的输入。

这样，我们结束了用户注册模块。接下来，我们将开始开发用户的登录/退出功能。

但是，且慢，还有一个差点被我们遗忘的。

## 控制器的测试

你可能对 `mix test test/models/user_test.exs` 命令已经烂熟于心。但 `mix test test/controllers/user_controller_test.exs` 呢？

我们在生成用户的样板文件时，曾经生成过一个 `user_controller_test.exs` 文件，让我们运行下 `mix test test/controllers/user_controller_test.exs` 看看结果：

```bash
$ mix test test/controllers/user_controller_test.exs
Compiling 1 file (.ex)
....

  1) test updates chosen resource and redirects when data is valid (TvRecipe.UserControllerTest)
     test/controllers/user_controller_test.exs:47
     ** (RuntimeError) expected redirection with status 302, got: 200
     stacktrace:
       (phoenix) lib/phoenix/test/conn_test.ex:443: Phoenix.ConnTest.redirected_to/2
       test/controllers/user_controller_test.exs:50: (test)

....

  2) test creates resource and redirects when data is valid (TvRecipe.UserControllerTest)
     test/controllers/user_controller_test.exs:18
     ** (RuntimeError) expected redirection with status 302, got: 200
     stacktrace:
       (phoenix) lib/phoenix/test/conn_test.ex:443: Phoenix.ConnTest.redirected_to/2
       test/controllers/user_controller_test.exs:20: (test)



Finished in 0.3 seconds
10 tests, 2 failures
```
好消息是，10 个测试，有 8 个通过；坏消息是有 2 个未通过。

显然，从模板文件到现在，我们的代码已经变化，现在测试文件一样需要根据实际情况做调整：

```elixir
diff --git a/test/controllers/user_controller_test.exs b/test/controllers/user_controller_test.exs
index 2e08483..95d3108 100644
--- a/test/controllers/user_controller_test.exs
+++ b/test/controllers/user_controller_test.exs
@@ -2,7 +2,7 @@ defmodule TvRecipe.UserControllerTest do
   use TvRecipe.ConnCase

   alias TvRecipe.User
-  @valid_attrs %{email: "some content", password: "some content", username: "some content"}
+  @valid_attrs %{email: "chenxsan@gmail.com", password: "some content", username: "chenxsan"}
   @invalid_attrs %{}

   test "lists all entries on index", %{conn: conn} do
@@ -18,7 +18,7 @@ defmodule TvRecipe.UserControllerTest do
   test "creates resource and redirects when data is valid", %{conn: conn} do
     conn = post conn, user_path(conn, :create), user: @valid_attrs
     assert redirected_to(conn) == user_path(conn, :index)
-    assert Repo.get_by(User, @valid_attrs)
+    assert Repo.get_by(User, @valid_attrs |> Map.delete(:password))
   end

   test "does not create resource and renders errors when data is invalid", %{conn: conn} do
@@ -48,7 +48,7 @@ defmodule TvRecipe.UserControllerTest do
     user = Repo.insert! %User{}
     conn = put conn, user_path(conn, :update, user), user: @valid_attrs
     assert redirected_to(conn) == user_path(conn, :show, user)
-    assert Repo.get_by(User, @valid_attrs)
+    assert Repo.get_by(User, @valid_attrs |> Map.delete(:password))
   end
```
我们在代码中做了三处修改，一个是订正 `@valid_attrs`，另外两个是修改 `Repo.get`，因为我们的 `User` 不再有 `password` 字段，所以应该从 `@valid_attrs` 中移除它，否则就会报错。

再运行测试，全部通过。

至此，我们完成所有用户注册相关的开发，下一章，开始进入[用户登录环节](../05-session/01-login.md)。