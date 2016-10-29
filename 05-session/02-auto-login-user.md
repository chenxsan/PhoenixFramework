# 注册成功自动登录

我们讲过了注册，也讲过登录，但还有个小问题需要我们注意。

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
代码里，我们在成功创建用户后，发布了一条临时性的提示信息，然后页面重定向到所有用户的列表页。

因为 PhoenixMoment 项目目前没有确认用户邮箱的功能，所以我们可以在用户注册成功后自动登录，省掉登录过程。

我们调整下 `user_controller_test.exs` 文件中的测试：

```elixir
diff --git a/test/controllers/user_controller_test.exs b/test/controllers/user_controller_test.exs
index 301add2..7125bdc 100644
--- a/test/controllers/user_controller_test.exs
+++ b/test/controllers/user_controller_test.exs
@@ -19,6 +19,10 @@ defmodule PhoenixMoment.UserControllerTest do
     conn = post conn, user_path(conn, :create), user: @valid_attrs
     assert redirected_to(conn) == user_path(conn, :index)
     assert Repo.get_by(User, Map.delete(@valid_attrs, :password))
+
+    # 因为注册成功后自动登录了，所以读取首页时，返回中应该包含用户名
+    conn = get conn, page_path(conn, :index)
+    assert html_response(conn, 200) =~ Map.get(@valid_attrs, :username)
   end
```
然后修改 `user_controller.ex` 文件，让用户注册成功后自动登录：

```elixir
diff --git a/web/controllers/user_controller.ex b/web/controllers/user_controller.ex
index 27f760a..f4c534c 100644
--- a/web/controllers/user_controller.ex
+++ b/web/controllers/user_controller.ex
@@ -17,9 +17,11 @@ defmodule PhoenixMoment.UserController do
     changeset = User.changeset(%User{}, user_params)

     case Repo.insert(changeset) do
-      {:ok, _user} ->
+      {:ok, user} ->
         conn
         |> put_flash(:info, "User created successfully.")
+        |> put_session(:user_id, user.id)
+        |> assign(:current_user, user)
         |> redirect(to: user_path(conn, :index))
       {:error, changeset} ->
         render(conn, "new.html", changeset: changeset)
```
接着运行测试 `mix test test/controllers`，全部通过。

下一章里，我们将开发[退出登录](03-logout.md)的功能。