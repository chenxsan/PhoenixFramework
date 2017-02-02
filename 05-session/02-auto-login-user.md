# 注册成功自动登录

我们介绍过了注册，也介绍过登录，但还有个小问题需要我们注意。

来看下 `user_controller.ex` 文件中 `create` 动作的代码：

```elixir
  def create(conn, %{"user" => user_params}) do
    changeset = User.changeset(%User{}, user_params)

    case Repo.insert(changeset) do
      {:ok, _user} ->
        conn
        |> put_flash(:info, "User created successfully.")
        |> redirect(to: user_path(conn, :index))
      {:error, changeset} ->
        render(conn, "new.html", changeset: changeset)
    end
  end
```
代码里，我们在成功创建用户后，发布了一条临时性的提示消息，然后页面重定向到所有用户的列表页。

我们希望用户注册成功后自动登录，并且跳转到站点的主页。

我们调整下 `user_controller_test.exs` 文件中的测试：

```elixir
diff --git a/test/controllers/user_controller_test.exs b/test/controllers/user_controller_test.exs
index 95d3108..26055e3 100644
--- a/test/controllers/user_controller_test.exs
+++ b/test/controllers/user_controller_test.exs
@@ -17,8 +17,11 @@ defmodule TvRecipe.UserControllerTest do

   test "creates resource and redirects when data is valid", %{conn: conn} do
     conn = post conn, user_path(conn, :create), user: @valid_attrs
-    assert redirected_to(conn) == user_path(conn, :index)
+    assert redirected_to(conn) == page_path(conn, :index)
     assert Repo.get_by(User, @valid_attrs |> Map.delete(:password))
+    # 注册后自动登录，检查首页是否包含用户名
+    conn = get conn, page_path(conn, :index)
+    assert html_response(conn, 200) =~ Map.get(@valid_attrs, :username)
   end
```
然后修改 `user_controller.ex` 文件：

```elixir
diff --git a/web/controllers/user_controller.ex b/web/controllers/user_controller.ex
index 7d13c5f..8d8a6f5 100644
--- a/web/controllers/user_controller.ex
+++ b/web/controllers/user_controller.ex
@@ -17,10 +17,11 @@ defmodule TvRecipe.UserController do
     changeset = User.changeset(%User{}, user_params)

     case Repo.insert(changeset) do
-      {:ok, _user} ->
+      {:ok, user} ->
         conn
         |> put_flash(:info, "User created successfully.")
-        |> redirect(to: user_path(conn, :index))
+        |> put_session(:user_id, user.id)
+        |> redirect(to: page_path(conn, :index))
       {:error, changeset} ->
         render(conn, "new.html", changeset: changeset)
     end
```
运行 `mix test`，测试全部通过。

下一章，我们开发[退出登录](03-logout.md)功能。