# 退出登录

上一章，我们完成了[登录功能](05-session/01-login.md)，这一章里，我们将加上退出登录的功能。

我们的页面在用户登录后，将显示**退出**按钮，用户点击后，即可退出登录。

先写测试：

```elixir
diff --git a/test/controllers/session_controller_test.exs b/test/controllers/session_controller_test.exs
index 31a043e..6141f21 100644
--- a/test/controllers/session_controller_test.exs
+++ b/test/controllers/session_controller_test.exs
@@ -26,6 +26,7 @@ defmodule PhoenixMoment.SessionControllerTest do
       # 再读取首页，flash 中的信息已清空
       conn = get conn, page_path(conn, :index)
       assert html_response(conn, 200) =~ Map.get(@valid_attrs, :username)
+      assert html_response(conn, 200) =~ "退出"
     end
```
我们测试用户登录后，页面上是否有“退出”两字。

当然，现在的测试是通不过的：

```bash
$ mix test test/controllers
............

  1) test create/2 logins user and redirects when data is valid (PhoenixMoment.SessionControllerTest)
     test/controllers/session_controller_test.exs:13
     Assertion with =~ failed
     code: html_response(conn, 200) =~ "退出"
     lhs:  "<!DOCTYPE html>\n<html lang=\"en\">\n  <head>\n    <meta charset=\"utf-8\">\n    <meta http-equiv=\"X-UA-Compatible\" content=\"IE=edge\">\n    <meta name=\"viewport\" content=\"width=device-width, initial-scale=1\">\n    <meta name=\"description\" content=\"\">\n    <meta name=\"author\" content=\"\">\n\n    <title>Hello PhoenixMoment!</title>\n    <link rel=\"stylesheet\" href=\"/css/app.css\">\n  </head>\n\n  <body>\n    <div class=\"container\">\n      <header class=\"header\">\n        <nav role=\"navigation\">\n          <ul class=\"nav nav-pills pull-right\">\n            <li><a href=\"http://www.phoenixframework.org/docs\">Get Started</a></li>\n              <li>chenxsan</li>\n          </ul>\n        </nav>\n        <span class=\"logo\"></span>\n      </header>\n\n      <p class=\"alert alert-info\" role=\"alert\"></p>\n      <p class=\"alert alert-danger\" role=\"alert\"></p>\n\n      <main role=\"main\">\n<div class=\"jumbotron\">\n  <h2>Welcome to Phoenix!</h2>\n  <p class=\"lead\">A productive web framework that<br />does not compromise speed and maintainability.</p>\n</div>\n\n<div class=\"row marketing\">\n  <div class=\"col-lg-6\">\n    <h4>Resources</h4>\n    <ul>\n      <li>\n        <a href=\"http://phoenixframework.org/docs/overview\">Guides</a>\n      </li>\n      <li>\n        <a href=\"https://hexdocs.pm/phoenix\">Docs</a>\n      </li>\n      <li>\n        <a href=\"https://github.com/phoenixframework/phoenix\">Source</a>\n      </li>\n    </ul>\n  </div>\n\n  <div class=\"col-lg-6\">\n    <h4>Help</h4>\n    <ul>\n      <li>\n        <a href=\"http://groups.google.com/group/phoenix-talk\">Mailing list</a>\n      </li>\n      <li>\n        <a href=\"http://webchat.freenode.net/?channels=elixir-lang\">#elixir-lang on freenode IRC</a>\n      </li>\n      <li>\n        <a href=\"https://twitter.com/elixirphoenix\">@elixirphoenix</a>\n      </li>\n    </ul>\n  </div>\n</div>\n      </main>\n\n    </div> <!-- /container -->\n    <script src=\"/js/app.js\"></script>\n  </body>\n</html>\n"
     rhs:  "退出"
     stacktrace:
       test/controllers/session_controller_test.exs:29: (test)

..

Finished in 3.1 seconds
15 tests, 1 failure
```
打开 `app.html.eex` 文件，做如下修改：

```elixir
diff --git a/web/templates/layout/app.html.eex b/web/templates/layout/app.html.eex
index aa4b218..aa95174 100644
--- a/web/templates/layout/app.html.eex
+++ b/web/templates/layout/app.html.eex
@@ -19,6 +19,7 @@
             <li><a href="http://www.phoenixframework.org/docs">Get Started</a></li>
             <%= if @current_user do %>
               <li><%= @current_user.username %></li>
+              <li><%= link "退出", to: session_path(@conn, :delete, @current_user), method: "delete" %></li>
             <% end %>
           </ul>
         </nav>
```
现在运行测试，已经能够通过。

但我们还没有写好 `session_controller.ex` 文件中的 `delete` 函数。

我们期望用户登录后点击“退出”，页面跳转到主页，并且显示“退出成功”。

我们的测试将这么写：

```elixir
diff --git a/test/controllers/session_controller_test.exs b/test/controllers/session_controller_test.exs
index 6141f21..c50466d 100644
--- a/test/controllers/session_controller_test.exs
+++ b/test/controllers/session_controller_test.exs
@@ -51,4 +51,28 @@ defmodule PhoenixMoment.SessionControllerTest do
       assert html_response(conn, 200) =~ "用户名或密码错误"
     end
   end
+
+  describe "delete/2" do
+    test "logouts user when logout button clicked", %{conn: conn} do
+      # 在数据库中新建一个用户
+      changeset = User.changeset(%User{}, @valid_attrs)
+      user = Repo.insert!(changeset)
+
+      # 登录该用户
+      conn = post conn, session_path(conn, :create), session: Map.delete(@valid_attrs, :email)
+      assert redirected_to(conn) == page_path(conn, :index)
+
+      # 读取首页，返回中应该包含“欢迎回来”
+      conn = get conn, page_path(conn, :index)
+      assert html_response(conn, 200) =~ "欢迎回来"
+
+      # 点击退出
+      conn = delete conn, session_path(conn, :delete, user)
+      assert redirected_to(conn) == page_path(conn, :index)
+
+      # 读取首页，返回中应该包含“退出成功”
+      conn = get conn, page_path(conn, :index)
+      assert html_response(conn, 200) =~ "退出成功"
+    end
+  end
 end
```
接着我们根据测试调整 `session_controller.ex` 文件：

```elixir
diff --git a/web/controllers/session_controller.ex b/web/controllers/session_controller.ex
index 60b746c..ea6916a 100644
--- a/web/controllers/session_controller.ex
+++ b/web/controllers/session_controller.ex
@@ -44,6 +44,9 @@ defmodule PhoenixMoment.SessionController do
   end

   def delete(conn, _params) do
-
+    conn
+    |> delete_session(:user_id)
+    |> put_flash(:info, "退出成功")
+    |> redirect(to: page_path(conn, :index))
   end
 end
```
运行测试，全部通过。