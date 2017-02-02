# 退出登录

这一章里，我们将添加退出登录的功能。

先写测试：

```elixir
diff --git a/test/controllers/session_controller_test.exs b/test/controllers/session_controller_test.exs
index c5f4616..511d0ab 100644
--- a/test/controllers/session_controller_test.exs
+++ b/test/controllers/session_controller_test.exs
@@ -26,6 +26,8 @@ defmodule TvRecipe.SessionControllerTest do
     # 读取用户页，页面上包含已登录用户的用户名
     conn = get conn, user_path(conn, :show, user)
     assert html_response(conn, 200) =~ Map.get(@valid_user_attrs, :username)
+    # 登录后的页面显示“退出”
+    assert html_response(conn, 200) =~ "退出"
   end
```
我们测试用户登录后，页面上是否有“退出”两字。

当然，现在的测试是通不过的：

```bash
$ mix test
Compiling 1 file (.ex)
..............................

  1) test login user and redirect to home page when data is valid (TvRecipe.SessionControllerTest)
     test/controllers/session_controller_test.exs:13
     Assertion with =~ failed
     code:  html_response(conn, 200) =~ "退出"
     left:  "<!DOCTYPE html>\n<html lang=\"en\">\n  <head>\n    <meta charset=\"utf-8\">\n    <meta http-equiv=\"X-UA-Compatible\" content=\"IE=edge\">\n    <meta name=\"viewport\" content=\"width=device-width, initial-scale=1\">\n    <meta name=\"description\" content=\"\">\n    <meta name=\"author\" content=\"\">\n\n    <title>Hello TvRecipe!</title>\n    <link rel=\"stylesheet\" href=\"/css/app.css\">\n  </head>\n\n  <body>\n    <div class=\"container\">\n      <header class=\"header\">\n        <nav role=\"navigation\">\n          <ul class=\"nav nav-pills pull-right\">\n            <li><a href=\"http://www.phoenixframework.org/docs\">Get Started</a></li>\n              <li><a href=\"/users/625\">chenxsan</a></li>\n          </ul>\n        </nav>\n        <span class=\"logo\"></span>\n      </header>\n\n      <p class=\"alert alert-info\" role=\"alert\"></p>\n      <p class=\"alert alert-danger\" role=\"alert\"></p>\n\n      <main role=\"main\">\n<h2>Show user</h2>\n\n<ul>\n\n  <li>\n    <strong>Username:</strong>\nchenxsan  </li>\n\n  <li>\n    <strong>Email:</strong>\nchenxsan@gmail.com  </li>\n\n  <li>\n    <strong>Password:</strong>\n  </li>\n\n</ul>\n\n<a href=\"/users/625/edit\">Edit</a><a href=\"/users\">Back</a>\n      </main>\n\n    </div> <!-- /container -->\n    <script src=\"/js/app.js\"></script>\n  </body>\n</html>\n"
     right: "退出"
     stacktrace:
       test/controllers/session_controller_test.exs:30: (test)

....

Finished in 0.4 seconds
35 tests, 1 failure
```

打开 `app.html.eex` 文件，做如下修改：

```elixir
diff --git a/web/templates/layout/app.html.eex b/web/templates/layout/app.html.eex
index 2d39904..6c87a08 100644
--- a/web/templates/layout/app.html.eex
+++ b/web/templates/layout/app.html.eex
@@ -19,6 +19,7 @@
             <li><a href="http://www.phoenixframework.org/docs">Get Started</a></li>
             <%= if @current_user do %>
               <li><%= link @current_user.username, to: user_path(@conn, :show, @current_user) %></li>
+              <li><%= link "退出", to: session_path(@conn, :delete, @current_user), method: "delete" %></li>
             <% end %>
           </ul>
         </nav>
```
现在运行测试：

```bash
$ mix test
.............................

  1) test creates resource and redirects when data is valid (TvRecipe.UserControllerTest)
     test/controllers/user_controller_test.exs:18
     ** (ArgumentError) No helper clause for TvRecipe.Router.Helpers.session_path/3 defined for action :delete.
     The following session_path actions are defined under your router:

       * :create
       * :new
     stacktrace:
       (phoenix) lib/phoenix/router/helpers.ex:269: Phoenix.Router.Helpers.raise_route_error/5
       (tv_recipe) web/templates/layout/app.html.eex:22: TvRecipe.LayoutView."app.html"/1
       (phoenix) lib/phoenix/view.ex:335: Phoenix.View.render_to_iodata/3
       (phoenix) lib/phoenix/controller.ex:642: Phoenix.Controller.do_render/4
       (tv_recipe) web/controllers/page_controller.ex:1: TvRecipe.PageController.action/2
       (tv_recipe) web/controllers/page_controller.ex:1: TvRecipe.PageController.phoenix_controller_pipeline/2
       (tv_recipe) lib/tv_recipe/endpoint.ex:1: TvRecipe.Endpoint.instrument/4
       (tv_recipe) lib/phoenix/router.ex:261: TvRecipe.Router.dispatch/2
       (tv_recipe) web/router.ex:1: TvRecipe.Router.do_call/2
       (tv_recipe) lib/tv_recipe/endpoint.ex:1: TvRecipe.Endpoint.phoenix_pipeline/1
       (tv_recipe) lib/tv_recipe/endpoint.ex:1: TvRecipe.Endpoint.call/2
       (phoenix) lib/phoenix/test/conn_test.ex:224: Phoenix.ConnTest.dispatch/5
       test/controllers/user_controller_test.exs:23: (test)

...

  2) test login user and redirect to home page when data is valid (TvRecipe.SessionControllerTest)
     test/controllers/session_controller_test.exs:13
     ** (ArgumentError) No helper clause for TvRecipe.Router.Helpers.session_path/3 defined for action :delete.
     The following session_path actions are defined under your router:

       * :create
       * :new
     stacktrace:
       (phoenix) lib/phoenix/router/helpers.ex:269: Phoenix.Router.Helpers.raise_route_error/5
       (tv_recipe) web/templates/layout/app.html.eex:22: TvRecipe.LayoutView."app.html"/1
       (phoenix) lib/phoenix/view.ex:335: Phoenix.View.render_to_iodata/3
       (phoenix) lib/phoenix/controller.ex:642: Phoenix.Controller.do_render/4
       (tv_recipe) web/controllers/page_controller.ex:1: TvRecipe.PageController.action/2
       (tv_recipe) web/controllers/page_controller.ex:1: TvRecipe.PageController.phoenix_controller_pipeline/2
       (tv_recipe) lib/tv_recipe/endpoint.ex:1: TvRecipe.Endpoint.instrument/4
       (tv_recipe) lib/phoenix/router.ex:261: TvRecipe.Router.dispatch/2
       (tv_recipe) web/router.ex:1: TvRecipe.Router.do_call/2
       (tv_recipe) lib/tv_recipe/endpoint.ex:1: TvRecipe.Endpoint.phoenix_pipeline/1
       (tv_recipe) lib/tv_recipe/endpoint.ex:1: TvRecipe.Endpoint.call/2
       (phoenix) lib/phoenix/test/conn_test.ex:224: Phoenix.ConnTest.dispatch/5
       test/controllers/session_controller_test.exs:24: (test)

.

Finished in 0.5 seconds
35 tests, 2 failures
```
前一个测试中的错误消除了，却又多了两个错误。这是因为我们还没有定义 `session_controller.ex` 文件中的 `delete` 动作及相应路由。

我们希望用户登录后点击“退出”，页面跳转到主页，并显示“退出成功”。

我们的测试这么写：

```elixir
diff --git a/test/controllers/session_controller_test.exs b/test/controllers/session_controller_test.exs
index 511d0ab..969662a 100644
--- a/test/controllers/session_controller_test.exs
+++ b/test/controllers/session_controller_test.exs
@@ -50,4 +50,18 @@ defmodule TvRecipe.SessionControllerTest do
     assert html_response(conn, 200) =~ "登录"
   end

+ test "logouts user when logout button clicked", %{conn: conn} do
+   # 在数据库中新建一个用户
+   changeset = User.changeset(%User{}, @valid_user_attrs)
+   user = Repo.insert!(changeset)
+
+    # 登录该用户
+   conn = post conn, session_path(conn, :create), session: Map.delete(@valid_user_attrs, :username)
+
+    # 点击退出
+   conn = delete conn, session_path(conn, :delete, user)
+   assert get_flash(conn, :info) == "退出成功"
+   assert redirected_to(conn) == page_path(conn, :index)
+ end
+
 end
```
接着我们根据测试中的要求调整 `session_controller.ex` 文件及 `router.ex`：

```elixir
diff --git a/web/controllers/session_controller.ex b/web/controllers/session_controller.ex
index b5218f2..2a887ee 100644
--- a/web/controllers/session_controller.ex
+++ b/web/controllers/session_controller.ex
@@ -30,4 +30,11 @@ defmodule TvRecipe.SessionController do
         |> render("new.html")
     end
   end
+
+  def delete(conn, _params) do
+    conn
+    |> delete_session(:user_id)
+    |> put_flash(:info, "退出成功")
+    |> redirect(to: page_path(conn, :index))
+  end
 end

diff --git a/web/router.ex b/web/router.ex
index 1265c86..4c12197 100644
--- a/web/router.ex
+++ b/web/router.ex
@@ -21,6 +21,7 @@ defmodule TvRecipe.Router do
     resources "/users", UserController
     get "/sessions/new", SessionController, :new
     post "/sessions/new", SessionController, :create
+    delete "/sessions/:id", SessionController, :delete
   end

   # Other scopes may use custom stacks.
```
运行测试，全部通过。

我们还可以优化下 `router.ex` 文件：

```elixir
diff --git a/web/router.ex b/web/router.ex
index 4c12197..292aeb8 100644
--- a/web/router.ex
+++ b/web/router.ex
@@ -19,9 +19,7 @@ defmodule TvRecipe.Router do

     get "/", PageController, :index
     resources "/users", UserController
-    get "/sessions/new", SessionController, :new
-    post "/sessions/new", SessionController, :create
-    delete "/sessions/:id", SessionController, :delete
+    resources "/sessions", SessionController, only: [:new, :create, :delete]
   end

   # Other scopes may use custom stacks.
```

下一章，我们给页面加上[登录/注册按钮](04-login-logout-buttons.md)，方便用户操作。