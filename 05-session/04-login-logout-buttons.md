# 登录/注册按钮

我们至今还没有在页面上添加登录/注册按钮，这对普通用户来说非常不友好。

老规则，先写测试，我们不再新增，而是加在旧测试中：

```elixir
diff --git a/test/controllers/session_controller_test.exs b/test/controllers/session_controller_test.exs
index 969662a..98fbb5a 100644
--- a/test/controllers/session_controller_test.exs
+++ b/test/controllers/session_controller_test.exs
@@ -14,6 +14,10 @@ defmodule TvRecipe.SessionControllerTest do
     user_changeset = User.changeset(%User{}, @valid_user_attrs)
     # 插入新用户
     user = Repo.insert! user_changeset
+    # 未登录情况下访问首页，应带有登录/注册文字
+    conn = get conn, page_path(conn, :index)
+    assert html_response(conn, 200) =~ "登录"
+    assert html_response(conn, 200) =~ "注册"
     # 用户登录
     conn = post conn, session_path(conn, :create), session: @valid_user_attrs
     # 显示“欢迎你”的消息
```

打开 `app.html.eex` 文件，添加两个按钮：

```eex
diff --git a/web/templates/layout/app.html.eex b/web/templates/layout/app.html.eex
index 6c87a08..b13f370 100644
--- a/web/templates/layout/app.html.eex
+++ b/web/templates/layout/app.html.eex
@@ -20,6 +20,9 @@
             <%= if @current_user do %>
               <li><%= link @current_user.username, to: user_path(@conn, :show, @current_user) %></li>
               <li><%= link "退出", to: session_path(@conn, :delete, @current_user), method: "delete" %></li>
+            <% else %>
+              <li><%= link "登录", to: session_path(@conn, :new) %></li>
+              <li><%= link "注册", to: user_path(@conn, :new) %></li>
             <% end %>
           </ul>
         </nav>
```
运行测试：

```bash
mix test
....................................

Finished in 0.8 seconds
36 tests, 0 failures
```
全部通过。

在进入下一章前，我们还有个安全相关的问题需要略作修改：

```elixir
diff --git a/web/controllers/session_controller.ex b/web/controllers/session_controller.ex
index 6ac524c..0c2eb0a 100644
--- a/web/controllers/session_controller.ex
+++ b/web/controllers/session_controller.ex
@@ -15,6 +15,7 @@ defmodule TvRecipe.SessionController do
         conn
         |> put_session(:user_id, user.id)
         |> put_flash(:info, "欢迎你")
+        |> configure_session(renew: true)
         |> redirect(to: page_path(conn, :index))
       # 用户存在，但密码错误
       user ->
diff --git a/web/controllers/user_controller.ex b/web/controllers/user_controller.ex
index 8d8a6f5..8b9b38b 100644
--- a/web/controllers/user_controller.ex
+++ b/web/controllers/user_controller.ex
@@ -21,6 +21,7 @@ defmodule TvRecipe.UserController do
         conn
         |> put_flash(:info, "User created successfully.")
         |> put_session(:user_id, user.id)
+        |> configure_session(renew: true)
         |> redirect(to: page_path(conn, :index))
       {:error, changeset} ->
         render(conn, "new.html", changeset: changeset)
```
我们在 `session_controller.ex` 与 `user_controller.ex` 两个文件中加入了 `configure_session(renew: true)`，用于预防 [session fixation 攻击](https://www.owasp.org/index.php/Session_fixation)。

下一章，我们要对[用户相关页面做些限制](../06-restrict-access.md)，以保证数据的安全。