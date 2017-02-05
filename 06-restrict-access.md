# 安全限制

## 限制未登录用户访问用户页面

目前为止，所有未登录用户都可以访问、操作用户相关页面。我们要加以限制：**未登录用户只允许使用 `:new` 及 `:create` 两个动作，访问其余动作时，全部重定向到登录页。**

首先在 `user_controller_test.exs` 文件中新增一个测试，具体内容看注释：

```elixir
diff --git a/test/controllers/user_controller_test.exs b/test/controllers/user_controller_test.exs
index 26055e3..ac6894e 100644
--- a/test/controllers/user_controller_test.exs
+++ b/test/controllers/user_controller_test.exs
@@ -66,4 +66,18 @@ defmodule TvRecipe.UserControllerTest do
     assert redirected_to(conn) == user_path(conn, :index)
     refute Repo.get(User, user.id)
   end
+
+  test "guest access user action redirected to login page", %{conn: conn} do
+    user = Repo.insert! %User{}
+    Enum.each([
+      get(conn, user_path(conn, :index)),
+      get(conn, user_path(conn, :show, user)),
+      get(conn, user_path(conn, :edit, user)),
+      put(conn, user_path(conn, :update, user), user: %{}),
+      delete(conn, user_path(conn, :delete, user))
+    ], fn conn ->
+      assert redirected_to(conn) == session_path(conn, :new)
+      assert conn.halted
+    end)
+  end
 end
```
接下来修改 `user_controller.ex` 文件中的代码：

```elixir
diff --git a/web/controllers/user_controller.ex b/web/controllers/user_controller.ex
index b9234b1..7bb7dac 100644
--- a/web/controllers/user_controller.ex
+++ b/web/controllers/user_controller.ex
@@ -1,5 +1,6 @@
 defmodule TvRecipe.UserController do
   use TvRecipe.Web, :controller
+  plug :login_require when action in [:index, :show, :edit, :update, :delete]

   alias TvRecipe.User

@@ -63,4 +64,20 @@ defmodule TvRecipe.UserController do
     |> put_flash(:info, "User deleted successfully.")
     |> redirect(to: user_path(conn, :index))
   end
+
+  @doc """
+  检查用户登录状态
+
+  Returns `conn`
+  """
+  def login_require(conn, _opts) do
+    if conn.assigns.current_user do
+      conn
+    else
+      conn
+      |> put_flash(:info, "请先登录")
+      |> redirect(to: session_path(conn, :new))
+      |> halt()
+    end
+  end
 end
```
我们增加了一个函数式 plug `login_require`，并且将它应用在控制器中的动作前。

还记得我们的一个流程图吗？

```elixir
conn
|> router
|> pipelines
|> controller
|> view
|> template
```
这里，我们还能再进一步完善它：

```elixir
conn
|> router
|> pipelines
|> controller
|> plugs
|> action
|> view
|> template
```
我们的控制器在执行动作前，会按指定顺序执行一系列 plug。

现在运行测试：

```bash
$ mix test
.....................

  1) test renders form for editing chosen resource (TvRecipe.UserControllerTest)
     test/controllers/user_controller_test.exs:44
     ** (RuntimeError) expected response with status 200, got: 302, with body:
     <html><body>You are being <a href="/sessions/new">redirected</a>.</body></html>
     stacktrace:
       (phoenix) lib/phoenix/test/conn_test.ex:362: Phoenix.ConnTest.response/2
       (phoenix) lib/phoenix/test/conn_test.ex:376: Phoenix.ConnTest.html_response/2
       test/controllers/user_controller_test.exs:47: (test)



  2) test lists all entries on index (TvRecipe.UserControllerTest)
     test/controllers/user_controller_test.exs:8
     ** (RuntimeError) expected response with status 200, got: 302, with body:
     <html><body>You are being <a href="/sessions/new">redirected</a>.</body></html>
     stacktrace:
       (phoenix) lib/phoenix/test/conn_test.ex:362: Phoenix.ConnTest.response/2
       (phoenix) lib/phoenix/test/conn_test.ex:376: Phoenix.ConnTest.html_response/2
       test/controllers/user_controller_test.exs:10: (test)



  3) test renders page not found when id is nonexistent (TvRecipe.UserControllerTest)
     test/controllers/user_controller_test.exs:38
     expected error to be sent as 404 status, but response sent 302 without error
     stacktrace:
       (phoenix) lib/phoenix/test/conn_test.ex:570: Phoenix.ConnTest.assert_error_sent/2
       test/controllers/user_controller_test.exs:39: (test)

..

  4) test deletes chosen resource (TvRecipe.UserControllerTest)
     test/controllers/user_controller_test.exs:63
     Assertion with == failed
     code:  redirected_to(conn) == user_path(conn, :index)
     left:  "/sessions/new"
     right: "/users"
     stacktrace:
       test/controllers/user_controller_test.exs:66: (test)



  5) test shows chosen resource (TvRecipe.UserControllerTest)
     test/controllers/user_controller_test.exs:32
     ** (RuntimeError) expected response with status 200, got: 302, with body:
     <html><body>You are being <a href="/sessions/new">redirected</a>.</body></html>
     stacktrace:
       (phoenix) lib/phoenix/test/conn_test.ex:362: Phoenix.ConnTest.response/2
       (phoenix) lib/phoenix/test/conn_test.ex:376: Phoenix.ConnTest.html_response/2
       test/controllers/user_controller_test.exs:35: (test)



  6) test updates chosen resource and redirects when data is valid (TvRecipe.UserControllerTest)
     test/controllers/user_controller_test.exs:50
     Assertion with == failed
     code:  redirected_to(conn) == user_path(conn, :show, user)
     left:  "/sessions/new"
     right: "/users/1121"
     stacktrace:
       test/controllers/user_controller_test.exs:53: (test)



  7) test does not update chosen resource and renders errors when data is invalid (TvRecipe.UserControllerTest)
     test/controllers/user_controller_test.exs:57
     ** (RuntimeError) expected response with status 200, got: 302, with body:
     <html><body>You are being <a href="/sessions/new">redirected</a>.</body></html>
     stacktrace:
       (phoenix) lib/phoenix/test/conn_test.ex:362: Phoenix.ConnTest.response/2
       (phoenix) lib/phoenix/test/conn_test.ex:376: Phoenix.ConnTest.html_response/2
       test/controllers/user_controller_test.exs:60: (test)

.......

Finished in 0.4 seconds
37 tests, 7 failures
```
因为我们前面新增了 `login_require` 限制，导致旧的测试有 7 个失败，它们均需要用户登录。

怎么测试用户登录的情况？

我们有一种选择是，在每一个测试前登录用户，比如这样：

```elixir
  test "shows chosen resource", %{conn: conn} do
    user = Repo.insert! User.changeset(%User{}, @valid_attrs)
    conn = post conn, session_path(conn, :create), session: @valid_attrs # <= 这一行，登录用户
    conn = get conn, user_path(conn, :show, user)
    assert html_response(conn, 200) =~ "Show user"
  end
```
只是这样我们会重复很多代码。

我们还可以借助 [`setup`](https://hexdocs.pm/ex_unit/ExUnit.Callbacks.html#setup/2)。在 Elixir 的测试里，`setup` 块的代码会在每一个 `test` 执行以前执行，它们返回的内容合并进 `context`，然后我们就可以在 `test` 中获取到。

但我们在一个测试文件中涉及两种情况，登录与未登录，`setup` 要如何区分它们？

我们可以使用 [`tag`](https://hexdocs.pm/ex_unit/ExUnit.Case.html#module-tags)。通过 `tag`，我们在上下文 `context` 中存储变量，`setup` 读取 `context`，根据需求返回不同的数据。

我们的代码改造如下：

```elixir
diff --git a/test/controllers/user_controller_test.exs b/test/controllers/user_controller_test.exs
index ac6894e..e11df40 100644
--- a/test/controllers/user_controller_test.exs
+++ b/test/controllers/user_controller_test.exs
@@ -1,10 +1,22 @@
 defmodule TvRecipe.UserControllerTest do
   use TvRecipe.ConnCase

-  alias TvRecipe.User
+  alias TvRecipe.{Repo, User}
   @valid_attrs %{email: "chenxsan@gmail.com", password: "some content", username: "chenxsan"}
   @invalid_attrs %{}

+  setup %{conn: conn} = context do
+    if context[:logged_in] == true do
+      # 如果上下文里 :logged_in 值为 true
+      user = Repo.insert! User.changeset(%User{}, @valid_attrs)
+      conn = post conn, session_path(conn, :create), session: @valid_attrs
+      {:ok, [conn: conn, user: user]}
+    else
+      :ok
+    end
+  end
+
+  @tag logged_in: true
   test "lists all entries on index", %{conn: conn} do
     conn = get conn, user_path(conn, :index)
     assert html_response(conn, 200) =~ "Listing users"
@@ -29,24 +41,28 @@ defmodule TvRecipe.UserControllerTest do
     assert html_response(conn, 200) =~ "New user"
   end

+  @tag logged_in: true
   test "shows chosen resource", %{conn: conn} do
     user = Repo.insert! %User{}
     conn = get conn, user_path(conn, :show, user)
     assert html_response(conn, 200) =~ "Show user"
   end

+  @tag logged_in: true
   test "renders page not found when id is nonexistent", %{conn: conn} do
     assert_error_sent 404, fn ->
       get conn, user_path(conn, :show, -1)
     end
   end

+  @tag logged_in: true
   test "renders form for editing chosen resource", %{conn: conn} do
     user = Repo.insert! %User{}
     conn = get conn, user_path(conn, :edit, user)
     assert html_response(conn, 200) =~ "Edit user"
   end

+  @tag logged_in: true
   test "updates chosen resource and redirects when data is valid", %{conn: conn} do
     user = Repo.insert! %User{}
     conn = put conn, user_path(conn, :update, user), user: @valid_attrs
@@ -54,12 +70,14 @@ defmodule TvRecipe.UserControllerTest do
     assert Repo.get_by(User, @valid_attrs |> Map.delete(:password))
   end

+  @tag logged_in: true
   test "does not update chosen resource and renders errors when data is invalid", %{conn: conn} do
     user = Repo.insert! %User{}
     conn = put conn, user_path(conn, :update, user), user: @invalid_attrs
     assert html_response(conn, 200) =~ "Edit user"
   end

+  @tag logged_in: true
   test "deletes chosen resource", %{conn: conn} do
     user = Repo.insert! %User{}
     conn = delete conn, user_path(conn, :delete, user)
```
我们根据 `logged_in` 的值返回不同 `conn`：一个是用户登录的 conn，一个是未登录的 conn。

现在运行测试：

```bash
$ mix test
...........................

  1) test updates chosen resource and redirects when data is valid (TvRecipe.UserControllerTest)
     test/controllers/user_controller_test.exs:66
     ** (RuntimeError) expected redirection with status 302, got: 200
     stacktrace:
       (phoenix) lib/phoenix/test/conn_test.ex:443: Phoenix.ConnTest.redirected_to/2
       test/controllers/user_controller_test.exs:69: (test)

.........

Finished in 0.5 seconds
37 tests, 1 failure
```
我们修复了大部分的错误，但还有一个失败的。

检查测试代码我们可以发现，`setup` 块里创建了一个邮箱为 `chenxsan@gmail.com`、用户名为 `chenxsan` 的用户，而更新时邮箱与用户名重复了。

我们调整一下：

```elixir
diff --git a/test/controllers/user_controller_test.exs b/test/controllers/user_controller_test.exs
index e11df40..c8263c6 100644
--- a/test/controllers/user_controller_test.exs
+++ b/test/controllers/user_controller_test.exs
@@ -65,7 +65,7 @@ defmodule TvRecipe.UserControllerTest do
   @tag logged_in: true
   test "updates chosen resource and redirects when data is valid", %{conn: conn} do
     user = Repo.insert! %User{}
-    conn = put conn, user_path(conn, :update, user), user: @valid_attrs
+    conn = put conn, user_path(conn, :update, user), user: %{@valid_attrs | username: "samchen", email: "chenxsan+1@gmail.com"}
     assert redirected_to(conn) == user_path(conn, :show, user)
     assert Repo.get_by(User, @valid_attrs |> Map.delete(:password))
   end
```
这样，我们就修正了所有测试。

## 限制用户访问管理动作

`user_controller.ex` 文件中，`:index` 与 `:delete` 动作通常是管理员才允许使用的，对普通用户来说，它们应该不可见。

我们直接移除相应的路由与控制器动作：

```elixir
diff --git a/web/controllers/user_controller.ex b/web/controllers/user_controller.ex
index 7bb7dac..c0056fd 100644
--- a/web/controllers/user_controller.ex
--- a/web/controllers/user_controller.ex
+++ b/web/controllers/user_controller.ex
@@ -1,14 +1,9 @@
 defmodule TvRecipe.UserController do
   use TvRecipe.Web, :controller
-  plug :login_require when action in [:index, :show, :edit, :update, :delete]
+  plug :login_require when action in [:show, :edit, :update]

   alias TvRecipe.User

-  def index(conn, _params) do
-    users = Repo.all(User)
-    render(conn, "index.html", users: users)
-  end
-
   def new(conn, _params) do
     changeset = User.changeset(%User{})
     render(conn, "new.html", changeset: changeset)
@@ -53,18 +48,6 @@ defmodule TvRecipe.UserController do
     end
   end

-  def delete(conn, %{"id" => id}) do
-    user = Repo.get!(User, id)
-
-    # Here we use delete! (with a bang) because we expect
-    # it to always work (and if it does not, it will raise).
-    Repo.delete!(user)
-
-    conn
-    |> put_flash(:info, "User deleted successfully.")
-    |> redirect(to: user_path(conn, :index))
-  end
-
   @doc """
   检查用户登录状态
```
然后运行测试。测试会帮我们定位出所有需要移除或修正的代码，我们逐一修改如下：

```elixir
diff --git a/test/controllers/user_controller_test.exs b/test/controllers/user_controller_test.exs
index c8263c6..a2ccee0 100644
--- a/test/controllers/user_controller_test.exs
+++ b/test/controllers/user_controller_test.exs
@@ -16,12 +16,6 @@ defmodule TvRecipe.UserControllerTest do
     end
   end

-  @tag logged_in: true
-  test "lists all entries on index", %{conn: conn} do
-    conn = get conn, user_path(conn, :index)
-    assert html_response(conn, 200) =~ "Listing users"
-  end
     end
   end

-  @tag logged_in: true
-  test "lists all entries on index", %{conn: conn} do
-    conn = get conn, user_path(conn, :index)
-    assert html_response(conn, 200) =~ "Listing users"
-  end
-
   test "renders form for new resources", %{conn: conn} do
     conn = get conn, user_path(conn, :new)
     assert html_response(conn, 200) =~ "New user"
@@ -77,22 +71,12 @@ defmodule TvRecipe.UserControllerTest do
     assert html_response(conn, 200) =~ "Edit user"
   end

-  @tag logged_in: true
-  test "deletes chosen resource", %{conn: conn} do
-    user = Repo.insert! %User{}
-    conn = delete conn, user_path(conn, :delete, user)
-    assert redirected_to(conn) == user_path(conn, :index)
-    refute Repo.get(User, user.id)
-  end
-
   test "guest access user action redirected to login page", %{conn: conn} do
     user = Repo.insert! %User{}
     Enum.each([
-      get(conn, user_path(conn, :index)),
       get(conn, user_path(conn, :show, user)),
       get(conn, user_path(conn, :edit, user)),
       put(conn, user_path(conn, :update, user), user: %{}),
-      delete(conn, user_path(conn, :delete, user))
     ], fn conn ->
       assert redirected_to(conn) == session_path(conn, :new)
       assert conn.halted
     ], fn conn ->
       assert redirected_to(conn) == session_path(conn, :new)
       assert conn.halted
diff --git a/web/templates/user/edit.html.eex b/web/templates/user/edit.html.eex
index 7e08f2b..beae173 100644
--- a/web/templates/user/edit.html.eex
+++ b/web/templates/user/edit.html.eex
@@ -2,5 +2,3 @@

 <%= render "form.html", changeset: @changeset,
                         action: user_path(@conn, :update, @user) %>
-
-<%= link "Back", to: user_path(@conn, :index) %>
diff --git a/web/templates/user/new.html.eex b/web/templates/user/new.html.eex
index e0b494f..adf2399 100644
--- a/web/templates/user/new.html.eex
+++ b/web/templates/user/new.html.eex
@@ -2,5 +2,3 @@

 <%= render "form.html", changeset: @changeset,
                         action: user_path(@conn, :create) %>
-
-<%= link "Back", to: user_path(@conn, :index) %>
diff --git a/web/templates/user/show.html.eex b/web/templates/user/show.html.eex
index d05f88d..4c3f497 100644
--- a/web/templates/user/show.html.eex
+++ b/web/templates/user/show.html.eex
@@ -20,4 +20,3 @@
 </ul>

 <%= link "Edit", to: user_path(@conn, :edit, @user) %>
-<%= link "Back", to: user_path(@conn, :index) %>
```
再次运行测试，全部通过。

## 限制已登录用户访问他人页面

我们还有一个问题，就是登录后的用户，通过修改 url 地址，能够访问他人的用户页面，还可以修改他人的信息。

我们需要加以限制：只有用户自己才可以访问、修改自己的用户页面。

我们有几种解决办法：

1. 不再通过 id 获取用户，直接读取 `conn.assigns.current_user`。
2. 把 id 隐藏起来，改用 `/profile` 这样的路径，用户就无从修改 url 中的 id，不过我们也没办法从 url 中获取 id，只能读取 `conn.assigns.current_user`。
3. 定义一个 `plug`，检查用户访问的 id 与 `conn.assigns.current_user` 的 id 是否一致，不一致则跳转。

这里使用第三种办法。

先定义一个测试：

```elixir
diff --git a/test/controllers/user_controller_test.exs b/test/controllers/user_controller_test.exs
index a2ccee0..fd57531 100644
--- a/test/controllers/user_controller_test.exs
+++ b/test/controllers/user_controller_test.exs
@@ -82,4 +82,19 @@ defmodule TvRecipe.UserControllerTest do
       assert conn.halted
     end)
   end
+
+  @tag logged_in: true
+  test "does not allow access to other user path", %{conn: conn, user: user} do
+    another_user = Repo.insert! %User{}
+    Enum.each([
+      get(conn, user_path(conn, :show, another_user)),
+      get(conn, user_path(conn, :edit, another_user)),
+      put(conn, user_path(conn, :update, another_user), user: %{})
+    ], fn conn ->
+      assert get_flash(conn, :error) == "禁止访问未授权页面"
+      assert redirected_to(conn) == user_path(conn, :show, user)
+      assert conn.halted
+    end)
+  end
+
 end
```
然后修改 `user_controller.ex` 文件：

```elixir
diff --git a/web/controllers/user_controller.ex b/web/controllers/user_controller.ex
index c0056fd..520d986 100644
--- a/web/controllers/user_controller.ex
+++ b/web/controllers/user_controller.ex
@@ -1,6 +1,7 @@
 defmodule TvRecipe.UserController do
   use TvRecipe.Web, :controller
   plug :login_require when action in [:show, :edit, :update]
+  plug :self_require when action in [:show, :edit, :update]

   alias TvRecipe.User

@@ -63,4 +64,21 @@ defmodule TvRecipe.UserController do
       |> halt()
     end
   end
+
+  @doc """
+  检查用户是否授权访问动作
+
+  Returns `conn`
+  """
+  def self_require(conn, _opts) do
+    %{"id" => id} = conn.params
+    if String.to_integer(id) == conn.assigns.current_user.id do
+      conn
+    else
+      conn
+      |> put_flash(:error, "禁止访问未授权页面")
+      |> redirect(to: user_path(conn, :show, conn.assigns.current_user))
+      |> halt()
+    end
+  end
 end
```
我们增加了一个 `self_require` 的 plug，并应用到几个动作上。请注意两个 plug 的顺序，`self_require` 排在 `login_require` 后面。

执行测试：

```bash
$ mix test
....................

  1) test shows chosen resource (TvRecipe.UserControllerTest)
     test/controllers/user_controller_test.exs:39
     ** (RuntimeError) expected response with status 200, got: 302, with body:
     <html><body>You are being <a href="/users/2941">redirected</a>.</body></html>
     stacktrace:
       (phoenix) lib/phoenix/test/conn_test.ex:362: Phoenix.ConnTest.response/2
       (phoenix) lib/phoenix/test/conn_test.ex:376: Phoenix.ConnTest.html_response/2
       test/controllers/user_controller_test.exs:42: (test)



  2) test renders form for editing chosen resource (TvRecipe.UserControllerTest)
     test/controllers/user_controller_test.exs:53
     ** (RuntimeError) expected response with status 200, got: 302, with body:
     <html><body>You are being <a href="/users/2943">redirected</a>.</body></html>
     stacktrace:
       (phoenix) lib/phoenix/test/conn_test.ex:362: Phoenix.ConnTest.response/2
       (phoenix) lib/phoenix/test/conn_test.ex:376: Phoenix.ConnTest.html_response/2
       test/controllers/user_controller_test.exs:56: (test)

....

  3) test updates chosen resource and redirects when data is valid (TvRecipe.UserControllerTest)
     test/controllers/user_controller_test.exs:60
     Assertion with == failed
     code:  redirected_to(conn) == user_path(conn, :show, user)
     left:  "/users/2948"
     right: "/users/2949"
     stacktrace:
       test/controllers/user_controller_test.exs:63: (test)



  4) test renders page not found when id is nonexistent (TvRecipe.UserControllerTest)
     test/controllers/user_controller_test.exs:46
     expected error to be sent as 404 status, but response sent 302 without error
     stacktrace:
       (phoenix) lib/phoenix/test/conn_test.ex:570: Phoenix.ConnTest.assert_error_sent/2
       test/controllers/user_controller_test.exs:47: (test)

.

  5) test does not update chosen resource and renders errors when data is invalid (TvRecipe.UserControllerTest)
     test/controllers/user_controller_test.exs:68
     ** (RuntimeError) expected response with status 200, got: 302, with body:
     <html><body>You are being <a href="/users/2952">redirected</a>.</body></html>
     stacktrace:
       (phoenix) lib/phoenix/test/conn_test.ex:362: Phoenix.ConnTest.response/2
       (phoenix) lib/phoenix/test/conn_test.ex:376: Phoenix.ConnTest.html_response/2
       test/controllers/user_controller_test.exs:71: (test)

......

Finished in 0.5 seconds
36 tests, 5 failures
```
因为代码的改动，我们的测试又有失败的。让我们修正它们：

```elixir
diff --git a/test/controllers/user_controller_test.exs b/test/controllers/user_controller_test.exs
index fd57531..a1b75c6 100644
--- a/test/controllers/user_controller_test.exs
+++ b/test/controllers/user_controller_test.exs
@@ -3,6 +3,7 @@ defmodule TvRecipe.UserControllerTest do

   alias TvRecipe.{Repo, User}
   @valid_attrs %{email: "chenxsan@gmail.com", password: "some content", username: "chenxsan"}
+  @another_valid_attrs %{email: "chenxsan+1@gmail.com", password: "some content", username: "samchen"}
   @invalid_attrs %{}

   setup %{conn: conn} = context do
@@ -36,37 +37,26 @@ defmodule TvRecipe.UserControllerTest do
   end

   @tag logged_in: true
-  test "shows chosen resource", %{conn: conn} do
-    user = Repo.insert! %User{}
+  test "shows chosen resource", %{conn: conn, user: user} do
     conn = get conn, user_path(conn, :show, user)
     assert html_response(conn, 200) =~ "Show user"
   end

   @tag logged_in: true
-  test "renders page not found when id is nonexistent", %{conn: conn} do
-    assert_error_sent 404, fn ->
-      get conn, user_path(conn, :show, -1)
   @tag logged_in: true
-  test "renders page not found when id is nonexistent", %{conn: conn} do
-    assert_error_sent 404, fn ->
-      get conn, user_path(conn, :show, -1)
-    end
-  end
-
-  @tag logged_in: true
-  test "renders form for editing chosen resource", %{conn: conn} do
-    user = Repo.insert! %User{}
+  test "renders form for editing chosen resource", %{conn: conn, user: user} do
     conn = get conn, user_path(conn, :edit, user)
     assert html_response(conn, 200) =~ "Edit user"
   end

   @tag logged_in: true
-  test "updates chosen resource and redirects when data is valid", %{conn: conn} do
-    user = Repo.insert! %User{}
-    conn = put conn, user_path(conn, :update, user), user: %{@valid_attrs | username: "samchen", email: "chenxsan+1@gmail.com"}
+  test "updates chosen resource and redirects when data is valid", %{conn: conn, user: user} do
+    conn = put conn, user_path(conn, :update, user), user: @another_valid_attrs
     assert redirected_to(conn) == user_path(conn, :show, user)
-    assert Repo.get_by(User, @valid_attrs |> Map.delete(:password))
+    assert Repo.get_by(User, @another_valid_attrs |> Map.delete(:password))
   end

   @tag logged_in: true
-  test "does not update chosen resource and renders errors when data is invalid", %{conn: conn} do
-    user = Repo.insert! %User{}
+  test "does not update chosen resource and renders errors when data is invalid", %{conn: conn, user: user} do
     conn = put conn, user_path(conn, :update, user), user: @invalid_attrs
     assert html_response(conn, 200) =~ "Edit user"
   end
```
运行测试：

```bash
$ mix test
...................................

Finished in 0.4 seconds
35 tests, 0 failures
```
全部通过。