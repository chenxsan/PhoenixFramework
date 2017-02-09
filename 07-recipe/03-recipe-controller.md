# Recipe 控制器

上一章结尾，我们运行 `mix test` 检查出 `recipe_controller_test.exs` 文件中的两个错误。

很显然，它们是因为 `@valid_attrs` 中缺少 `user_id` 导致的。

怎么办，在 `@valid_attrs` 中随意添加个 `user_id` 可不能解决问题 - 用户必须存在。

一个粗暴的解决办法，是在每个测试中新建一个用户，然后把用户 id 传给 `@valid_attrs`，但那样又要重复一堆代码，我们可以把新建用户部分抽取到 `setup` 中：

```elixir
diff --git a/test/controllers/recipe_controller_test.exs b/test/controllers/recipe_controller_test.exs
index 646ebf2..51fdeab 100644
--- a/test/controllers/recipe_controller_test.exs
+++ b/test/controllers/recipe_controller_test.exs
@@ -1,10 +1,16 @@
 defmodule TvRecipe.RecipeControllerTest do
   use TvRecipe.ConnCase

-  alias TvRecipe.Recipe
+  alias TvRecipe.{Repo, User, Recipe}
   @valid_attrs %{content: "some content", episode: 42, name: "some content", season: 42, title: "some content"}
   @invalid_attrs %{}

+  setup do
+    user = Repo.insert! User.changeset(%User{}, %{email: "chenxsan@gmail.com", username: "chenxsan", password: String.duplicate("1", 6)})
+    attrs = Map.put(@valid_attrs, :user_id, user.id)
+    {:ok, [attrs: attrs]}
+  end
+
   test "lists all entries on index", %{conn: conn} do
     conn = get conn, recipe_path(conn, :index)
     assert html_response(conn, 200) =~ "Listing recipes"
@@ -15,10 +21,10 @@ defmodule TvRecipe.RecipeControllerTest do
     assert html_response(conn, 200) =~ "New recipe"
   end
+    user = Repo.insert! User.changeset(%User{}, %{email: "chenxsan@gmail.com", username: "chenxsan", password: String.duplicate("1", 6)})
+    attrs = Map.put(@valid_attrs, :user_id, user.id)
+    {:ok, [attrs: attrs]}
+  end
+
   test "lists all entries on index", %{conn: conn} do
     conn = get conn, recipe_path(conn, :index)
     assert html_response(conn, 200) =~ "Listing recipes"
@@ -15,10 +21,10 @@ defmodule TvRecipe.RecipeControllerTest do
     assert html_response(conn, 200) =~ "New recipe"
   end

-  test "creates resource and redirects when data is valid", %{conn: conn} do
-    conn = post conn, recipe_path(conn, :create), recipe: @valid_attrs
+  test "creates resource and redirects when data is valid", %{conn: conn, attrs: attrs} do
+    conn = post conn, recipe_path(conn, :create), recipe: attrs
     assert redirected_to(conn) == recipe_path(conn, :index)
-    assert Repo.get_by(Recipe, @valid_attrs)
+    assert Repo.get_by(Recipe, attrs)
   end

   test "does not create resource and renders errors when data is invalid", %{conn: conn} do
@@ -44,11 +50,11 @@ defmodule TvRecipe.RecipeControllerTest do
     assert html_response(conn, 200) =~ "Edit recipe"
   end

-  test "updates chosen resource and redirects when data is valid", %{conn: conn} do
+  test "updates chosen resource and redirects when data is valid", %{conn: conn, attrs: attrs} do
     recipe = Repo.insert! %Recipe{}
-    conn = put conn, recipe_path(conn, :update, recipe), recipe: @valid_attrs
+    conn = put conn, recipe_path(conn, :update, recipe), recipe: attrs
     assert redirected_to(conn) == recipe_path(conn, :show, recipe)
-    assert Repo.get_by(Recipe, @valid_attrs)
+    assert Repo.get_by(Recipe, attrs)
   end
```
在 `setup` 块中，我们新建了一个用户，并且重新组合出真正有效的 recipe 属性 `attrs`，然后返回。

现在运行测试：

```bash
$ mix test
......................................................

Finished in 0.8 seconds
56 tests, 0 failures
```
非常好，全部通过了。

接下来，我们处理动作的权限问题。

## Recipe 动作的权限

我们先确认 `RecipeController` 模块中各个动作的权限要求：

动作名|是否需要登录
---|---
index|需要
new|需要
create|需要
show|需要
edit|需要
update|需要
delete|需要

都要登录？难道未登录用户不能查看其它用户创建的菜谱？当然可以，但我们将新建路由来满足这些需求。这一节，我们开发的是 Recipe 相关的管理动作。

前面章节中我们已经尝试过使用 `tag` 来标注用户登录状态下的测试，现在根据上面罗列的需求来修改 `recipe_controller_test.exs` 文件中的测试：

```elixir
diff --git a/test/controllers/recipe_controller_test.exs b/test/controllers/recipe_controller_test.exs
index 51fdeab..5632f8c 100644
--- a/test/controllers/recipe_controller_test.exs
+++ b/test/controllers/recipe_controller_test.exs
@@ -5,51 +5,65 @@ defmodule TvRecipe.RecipeControllerTest do
   @valid_attrs %{content: "some content", episode: 42, name: "some content", season: 42, title: "some content"}
   @invalid_attrs %{}

-  setup do
-    user = Repo.insert! User.changeset(%User{}, %{email: "chenxsan@gmail.com", username: "chenxsan", password: String.duplicate("1", 6)})
-    attrs = Map.put(@valid_attrs, :user_id, user.id)
-    {:ok, [attrs: attrs]}
+  setup %{conn: conn} = context do
+    user_attrs = %{email: "chenxsan@gmail.com", username: "chenxsan", password: String.duplicate("1", 6)}
+    user = Repo.insert! User.changeset(%User{}, user_attrs)
+     attrs = Map.put(@valid_attrs, :user_id, user.id)
+    if context[:logged_in] == true do
+      conn = post conn, session_path(conn, :create), session: user_attrs
+      {:ok, [conn: conn, attrs: attrs]}
+    else
+      {:ok, [attrs: attrs]}
+    end
   end
-
+
+  @tag logged_in: true
   test "lists all entries on index", %{conn: conn} do
     conn = get conn, recipe_path(conn, :index)
     assert html_response(conn, 200) =~ "Listing recipes"
   end

+  @tag logged_in: true
   test "renders form for new resources", %{conn: conn} do
     conn = get conn, recipe_path(conn, :new)
     assert html_response(conn, 200) =~ "New recipe"
   end

+  @tag logged_in: true
   test "creates resource and redirects when data is valid", %{conn: conn, attrs: attrs} do
     conn = post conn, recipe_path(conn, :create), recipe: attrs
     assert redirected_to(conn) == recipe_path(conn, :index)
     assert Repo.get_by(Recipe, attrs)
   end

+  @tag logged_in: true
   test "does not create resource and renders errors when data is invalid", %{conn: conn} do
     conn = post conn, recipe_path(conn, :create), recipe: @invalid_attrs
     assert html_response(conn, 200) =~ "New recipe"
   end

+  @tag logged_in: true
   test "shows chosen resource", %{conn: conn} do
     recipe = Repo.insert! %Recipe{}
     conn = get conn, recipe_path(conn, :show, recipe)
     assert html_response(conn, 200) =~ "Show recipe"
   end

+  @tag logged_in: true
+  @tag logged_in: true
   test "renders page not found when id is nonexistent", %{conn: conn} do
     assert_error_sent 404, fn ->
       get conn, recipe_path(conn, :show, -1)
     end
   end

+  @tag logged_in: true
   test "renders form for editing chosen resource", %{conn: conn} do
     recipe = Repo.insert! %Recipe{}
     conn = get conn, recipe_path(conn, :edit, recipe)
     assert html_response(conn, 200) =~ "Edit recipe"
   end

+  @tag logged_in: true
   test "updates chosen resource and redirects when data is valid", %{conn: conn, attrs: attrs} do
     recipe = Repo.insert! %Recipe{}
     conn = put conn, recipe_path(conn, :update, recipe), recipe: attrs
@@ -57,12 +71,14 @@ defmodule TvRecipe.RecipeControllerTest do
     assert Repo.get_by(Recipe, attrs)
   end

+  @tag logged_in: true
   test "does not update chosen resource and renders errors when data is invalid", %{conn: conn} do
     recipe = Repo.insert! %Recipe{}
     conn = put conn, recipe_path(conn, :update, recipe), recipe: @invalid_attrs
     assert html_response(conn, 200) =~ "Edit recipe"
   end

+  @tag logged_in: true
   test "deletes chosen resource", %{conn: conn} do
     recipe = Repo.insert! %Recipe{}
     conn = delete conn, recipe_path(conn, :delete, recipe)
```

我们给所有测试代码都加上了 `@tag logged_in` 的标签。

接下来我们需要一个验证用户登录状态的 plug，不巧我们在 `user_controller.ex` 文件中已经定义了一个 `login_require` 的 plug，现在是其它地方也要用到它 - 再放在 `user_controller.ex` 中并不合适，我们将它移到 `auth.ex` 文件中：

```elixir
diff --git a/web/controllers/auth.ex b/web/controllers/auth.ex
index e298b68..3dd3e7f 100644
--- a/web/controllers/auth.ex
+++ b/web/controllers/auth.ex
@@ -1,5 +1,7 @@
 defmodule TvRecipe.Auth do
   import Plug.Conn
+  import Phoenix.Controller
+  alias TvRecipe.Router.Helpers

   @doc """
   初始化选项
@@ -21,4 +23,37 @@ defmodule TvRecipe.Auth do
     |> configure_session(renew: true)
   end

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
+      |> redirect(to: Helpers.session_path(conn, :new))
+      |> halt()
+    end
+  end
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
+      |> redirect(to: Helpers.user_path(conn, :show, conn.assigns.current_user))
+      |> halt()
+    end
+  end
+
 end
\ No newline at end of file
diff --git a/web/controllers/user_controller.ex b/web/controllers/user_controller.ex
index 520d986..0f023d3 100644
--- a/web/controllers/user_controller.ex
+++ b/web/controllers/user_controller.ex
@@ -49,36 +49,4 @@ defmodule TvRecipe.UserController do
     end
   end

-  @doc """
-  检查用户登录状态
-
-  Returns `conn`
-  """
-  def login_require(conn, _opts) do
-    if conn.assigns.current_user do
-      conn
-    else
-      conn
-      |> put_flash(:info, "请先登录")
-      |> redirect(to: session_path(conn, :new))
-      |> halt()
-    end
-  end
-
-  @doc """
   end

-  @doc """
-  检查用户登录状态
-
-  Returns `conn`
-  """
-  def login_require(conn, _opts) do
-    if conn.assigns.current_user do
-      conn
-    else
-      conn
-      |> put_flash(:info, "请先登录")
-      |> redirect(to: session_path(conn, :new))
-      |> halt()
-    end
-  end
-
-  @doc """
-  检查用户是否授权访问动作
-
-  Returns `conn`
-  """
-  def self_require(conn, _opts) do
-    %{"id" => id} = conn.params
-    if String.to_integer(id) == conn.assigns.current_user.id do
-      conn
-    else
-      conn
-      |> put_flash(:error, "禁止访问未授权页面")
-      |> redirect(to: user_path(conn, :show, conn.assigns.current_user))
-      |> halt()
-    end
-  end
 end
```
注意，我们并非只是简单的移动文本到 `auth.ex` 文件中。在 `auth.ex` 头部，我们还引入了两行代码，并调整了两个 plug：

```elixir
import Phoenix.Controller
alias TvRecipe.Router.Helpers
```
`import Phoenix.Controller` 导入 `put_flash` 等方法，而 `alias TvRecipe.Router.Helpers` 让我们在 `auth.ex` 中可以快速书写各种路径。

接着在 `web.ex` 文件中 `import` 它：

```elixir
diff --git a/web/web.ex b/web/web.ex
index 50fd62e..9990080 100644
--- a/web/web.ex
+++ b/web/web.ex
@@ -36,6 +36,7 @@ defmodule TvRecipe.Web do

       import TvRecipe.Router.Helpers
       import TvRecipe.Gettext
+      import TvRecipe.Auth, only: [login_require: 2, self_require: 2]
     end
   end
```
注意，目前我们只在 `controller` 中 `import`，后面可能会需要在 `router` 中 `import`。

最后，将 plug 应用到 `recipe_controller.ex` 文件中：

```elixir
diff --git a/web/controllers/recipe_controller.ex b/web/controllers/recipe_controller.ex
index 96a0276..c74b492 100644
--- a/web/controllers/recipe_controller.ex
+++ b/web/controllers/recipe_controller.ex
@@ -1,6 +1,6 @@
 defmodule TvRecipe.RecipeController do
   use TvRecipe.Web, :controller
-
+  plug :login_require
   alias TvRecipe.Recipe

   def index(conn, _params) do
```
我们这次并没有使用 `when action in`，plug 将应用到该文件中所有的动作。

运行测试：

```bash
$ mix test
......................................................

Finished in 0.7 seconds
56 tests, 0 failures
```
一切顺利。

但因为我们现在所有测试都针对登录状态，我们还需要补充下未登录状态的测试：

```elixir
diff --git a/test/controllers/recipe_controller_test.exs b/test/controllers/recipe_controller_test.exs
index 5632f8c..faf67ca 100644
--- a/test/controllers/recipe_controller_test.exs
+++ b/test/controllers/recipe_controller_test.exs
@@ -16,7 +16,7 @@ defmodule TvRecipe.RecipeControllerTest do
       {:ok, [attrs: attrs]}
     end
   end
-
+
   @tag logged_in: true
   test "lists all entries on index", %{conn: conn} do
     conn = get conn, recipe_path(conn, :index)
@@ -85,4 +85,20 @@ defmodule TvRecipe.RecipeControllerTest do
     assert redirected_to(conn) == recipe_path(conn, :index)
     refute Repo.get(Recipe, recipe.id)
   end
+
+  test "guest access user action redirected to login page", %{conn: conn} do
+    recipe = Repo.insert! %Recipe{}
+    Enum.each([
+      get(conn, recipe_path(conn, :index)),
+      get(conn, recipe_path(conn, :new)),
+      get(conn, recipe_path(conn, :create), recipe: %{}),
+      get(conn, recipe_path(conn, :show, recipe)),
+      get(conn, recipe_path(conn, :edit, recipe)),
+      put(conn, recipe_path(conn, :update, recipe), recipe: %{}),
+      put(conn, recipe_path(conn, :delete, recipe)),
+    ], fn conn ->
+      assert redirected_to(conn) == session_path(conn, :new)
+      assert conn.halted
+    end)
+  end
 end
```

## `user_id` 与 `:current_user`

在前面的测试里，我们先创建一个新用户，然后登录新用户，新建 recipe，并将新用户的 id 与新建的 recipe 关联起来。

考虑另一种情况：

1. 新建用户 A
2. 新建用户 B
3. 登录用户 B
4. 关联用户 A 的 id 与 Recipe

这一种情况，我们的测试还没有覆盖到。

我们来新增一个测试，验证一下：

```elixir
diff --git a/test/controllers/recipe_controller_test.exs b/test/controllers/recipe_controller_test.exs
index faf67ca..d8157a2 100644
--- a/test/controllers/recipe_controller_test.exs
+++ b/test/controllers/recipe_controller_test.exs
@@ -101,4 +101,17 @@ defmodule TvRecipe.RecipeControllerTest do
       assert conn.halted
     end)
   end
+
+  @tag logged_in: true
+  test "creates resource and redirects when data is valid but with other user_id", %{conn: conn, attrs: attrs} do
+    # 新建一个用户
+    user = Repo.insert! User.changeset(%User{}, %{email: "chenxsan+1@gmail.com", username: "samchen", password: String.duplicate("1", 6)})
+    # 将新用户的 id 更新入 attrs，尝试替 samchen 创建一个菜谱
+    new_attrs = %{attrs | user_id: user.id}
+    post conn, recipe_path(conn, :create), recipe: new_attrs
+    # 用户 chenxsan 只能创建自己的菜谱，无法替 samchen 创建菜谱
+    assert Repo.get_by(Recipe, attrs)
+    # samchen 不应该有菜谱
+    refute Repo.get_by(Recipe, new_attrs)
+  end
 end
```
运行测试：

```bash
$ mix test
..........................

  1) test creates resource and redirects when data is valid but with other user_id (TvRecipe.RecipeControllerTest)
     test/controllers/recipe_controller_test.exs:106
     Expected truthy, got nil
     code: Repo.get_by(Recipe, attrs)
     stacktrace:
       test/controllers/recipe_controller_test.exs:113: (test)

.............................

Finished in 0.7 seconds
58 tests, 1 failure
```
测试失败：登录状态下的用户 A 给用户 B 创建了一个菜谱。

那么要如何修改我们的控制器代码？

我们可以像测试中一样，把登录用户的 id 传递进去，比如：

```elixir
def create(conn, %{"recipe" => recipe_params}) do
  changeset = Recipe.changeset(%Recipe{}, Map.put(recipe_params, "user_id", conn.assigns.current_user.id))
```
可是，我们为什么要提供给用户传递 `user_id` 的机会呢？

让我们从 `recipe.ex` 文件中把 `user_id` 相关的代码去掉：

```elixir
diff --git a/web/models/recipe.ex b/web/models/recipe.ex
index a0b42fd..8d34ed2 100644
--- a/web/models/recipe.ex
+++ b/web/models/recipe.ex
@@ -17,8 +17,7 @@ defmodule TvRecipe.Recipe do
   """
   def changeset(struct, params \\ %{}) do
     struct
-    |> cast(params, [:name, :title, :season, :episode, :content, :user_id])
-    |> validate_required([:name, :title, :season, :episode, :content, :user_id], message: "请填写")
-    |> foreign_key_constraint(:user_id, message: "用户不存在")
+    |> cast(params, [:name, :title, :season, :episode, :content])
+    |> validate_required([:name, :title, :season, :episode, :content], message: "请填写")
   end
 end
```
`recipe_test.exs` 中 `user_id` 相关的代码也要去掉：

```elixir
diff --git a/test/models/recipe_test.exs b/test/models/recipe_test.exs
index 2e1191c..4e59fb9 100644
--- a/test/models/recipe_test.exs
+++ b/test/models/recipe_test.exs
@@ -3,7 +3,7 @@ defmodule TvRecipe.RecipeTest do

   alias TvRecipe.{Recipe}

-  @valid_attrs %{content: "some content", episode: 42, name: "some content", season: 42, title: "some content", user_id: 1}
+  @valid_attrs %{content: "some content", episode: 42, name: "some content", season: 42, title: "some content"}
   @invalid_attrs %{}

   test "changeset with valid attributes" do
@@ -40,14 +40,5 @@ defmodule TvRecipe.RecipeTest do
     attrs = %{@valid_attrs | content: ""}
     assert {:content, "请填写"} in errors_on(%Recipe{}, attrs)
   end
-
-  test "user_id is required" do
-    attrs = %{@valid_attrs | user_id: nil}
-    assert {:user_id, "请填写"} in errors_on(%Recipe{}, attrs)
-  end

-  test "user_id should exist in users table" do
-    {:error, changeset} = Repo.insert Recipe.changeset(%Recipe{}, @valid_attrs)
-    assert {:user_id, "用户不存在"} in errors_on(changeset)
-  end
 end
```
不要忘了还有 `recipe_controller_test.exs` 文件中的代码：

```elixir
diff --git a/test/controllers/recipe_controller_test.exs b/test/controllers/recipe_controller_test.exs
index d8157a2..d953315 100644
--- a/test/controllers/recipe_controller_test.exs
+++ b/test/controllers/recipe_controller_test.exs
@@ -7,13 +7,12 @@ defmodule TvRecipe.RecipeControllerTest do

   setup %{conn: conn} = context do
     user_attrs = %{email: "chenxsan@gmail.com", username: "chenxsan", password: String.duplicate("1", 6)}
-    user = Repo.insert! User.changeset(%User{}, user_attrs)
-     attrs = Map.put(@valid_attrs, :user_id, user.id)
+    Repo.insert! User.changeset(%User{}, user_attrs)
     if context[:logged_in] == true do
       conn = post conn, session_path(conn, :create), session: user_attrs
-      {:ok, [conn: conn, attrs: attrs]}
+      {:ok, [conn: conn]}
     else
-      {:ok, [attrs: attrs]}
+      :ok
     end
   end

@@ -30,10 +29,10 @@ defmodule TvRecipe.RecipeControllerTest do
   end

   @tag logged_in: true
-  test "creates resource and redirects when data is valid", %{conn: conn, attrs: attrs} do
-    conn = post conn, recipe_path(conn, :create), recipe: attrs
+  test "creates resource and redirects when data is valid", %{conn: conn} do
+    conn = post conn, recipe_path(conn, :create), recipe: @valid_attrs
     assert redirected_to(conn) == recipe_path(conn, :index)
-    assert Repo.get_by(Recipe, attrs)
+    assert Repo.get_by(Recipe, @valid_attrs)
   end

   @tag logged_in: true
@@ -64,11 +63,11 @@ defmodule TvRecipe.RecipeControllerTest do
@@ -64,11 +63,11 @@ defmodule TvRecipe.RecipeControllerTest do
   end

   @tag logged_in: true
-  test "updates chosen resource and redirects when data is valid", %{conn: conn, attrs: attrs} do
+  test "updates chosen resource and redirects when data is valid", %{conn: conn} do
     recipe = Repo.insert! %Recipe{}
-    conn = put conn, recipe_path(conn, :update, recipe), recipe: attrs
+    conn = put conn, recipe_path(conn, :update, recipe), recipe: @valid_attrs
     assert redirected_to(conn) == recipe_path(conn, :show, recipe)
-    assert Repo.get_by(Recipe, attrs)
+    assert Repo.get_by(Recipe, @valid_attrs)
   end

   @tag logged_in: true
@@ -102,16 +101,4 @@ defmodule TvRecipe.RecipeControllerTest do
     end)
   end

-  @tag logged_in: true
-  test "creates resource and redirects when data is valid but with other user_id", %{conn: conn, attrs: attrs} do
-    # 新建一个用户
-    user = Repo.insert! User.changeset(%User{}, %{email: "chenxsan+1@gmail.com", username: "samchen", password: String.duplicate("1", 6)})
-    # 将新用户的 id 更新入 attrs，尝试替 samchen 创建一个菜谱
-    new_attrs = %{attrs | user_id: user.id}
-    post conn, recipe_path(conn, :create), recipe: new_attrs
-    # 用户 chenxsan 只能创建自己的菜谱，无法替 samchen 创建菜谱
-    assert Repo.get_by(Recipe, attrs)
-    # samchen 不应该有菜谱
-    refute Repo.get_by(Recipe, new_attrs)
-  end
 end
```
是的，绕了一圈，我们把前面新增的那个测试给删除了，因为 `Recipe.changeset` 已经不再接收 `user_id`，那个测试已经失去意义。

那么，我们要怎样将当前登录的用户 id 置入 recipe 中？

Ecto 提供了 [build_assoc](https://hexdocs.pm/ecto/Ecto.html#build_assoc/3) 方法，用于处理这类“关联”，我们来改造下 `recipe_controller.ex` 文件：

```elixir
diff --git a/web/controllers/recipe_controller.ex b/web/controllers/recipe_controller.ex
index c74b492..967b7bc 100644
--- a/web/controllers/recipe_controller.ex
+++ b/web/controllers/recipe_controller.ex
@@ -9,12 +9,18 @@ defmodule TvRecipe.RecipeController do
   end

   def new(conn, _params) do
-    changeset = Recipe.changeset(%Recipe{})
+    changeset =
+      conn.assigns.current_user
+      |> build_assoc(:recipes)
+      |> Recipe.changeset()
     render(conn, "new.html", changeset: changeset)
   end

   def create(conn, %{"recipe" => recipe_params}) do
-    changeset = Recipe.changeset(%Recipe{}, recipe_params)
+    changeset =
+      conn.assigns.current_user
+      |> build_assoc(:recipes)
+      |> Recipe.changeset(recipe_params)

     case Repo.insert(changeset) do
       {:ok, _recipe} ->
```
你可能会好奇此时的 `changeset`，我们可以使用 `IO.inspect(changeset)` 在终端窗口中打印出来，它大致是这个样子：

```bash
#Ecto.Changeset<action: nil,
 changes: %{content: "Phoenix Framework is awesome", episode: 2, name: "2", season: 2, title: "2"},
 errors: [], data: #TvRecipe.Recipe<>, valid?: true>
```
可是其中并没有看到 `user_id` - 那么 `build_assoc` 构建的数据存放在哪？在 `changeset.data` 下，大致是这样：

```bash
%TvRecipe.Recipe{__meta__: #Ecto.Schema.Metadata<:built, "recipes">,
 content: nil, episode: nil, id: nil, inserted_at: nil, name: nil, season: nil,
 title: nil, updated_at: nil,
 user: #Ecto.Association.NotLoaded<association :user is not loaded>, user_id: 1}
```
咦，上面看起来有点像魔法。

但我们仔细阅读 `Repo.insert` 的[说明](https://hexdocs.pm/ecto/Ecto.Repo.html#c:insert/2)，可以看到如下一段：

> In case a changeset is given, the changes in the changeset are merged with the struct fields, and all of them are sent to the database.

其中有个关键词 **merged**，是的，存放在 `changeset.data` 下的数据，会被合并进去，这解释了我们 `user_id` 不在 `changeset.changes` 下，却最终在数据库中出现的魔法。

最后，我们还要调整些代码，来检验新建的 recipe 中是否包含了当前登录用户的 id：

```elixir
diff --git a/test/controllers/recipe_controller_test.exs b/test/controllers/recipe_controller_test.exs
index d953315..b901b61 100644
--- a/test/controllers/recipe_controller_test.exs
+++ b/test/controllers/recipe_controller_test.exs
@@ -7,10 +7,10 @@ defmodule TvRecipe.RecipeControllerTest do

   setup %{conn: conn} = context do
     user_attrs = %{email: "chenxsan@gmail.com", username: "chenxsan", password: String.duplicate("1", 6)}
-    Repo.insert! User.changeset(%User{}, user_attrs)
+    user = Repo.insert! User.changeset(%User{}, user_attrs)
     if context[:logged_in] == true do
       conn = post conn, session_path(conn, :create), session: user_attrs
-      {:ok, [conn: conn]}
+      {:ok, [conn: conn, user: user]}
     else
       :ok
     end
@@ -29,10 +29,10 @@ defmodule TvRecipe.RecipeControllerTest do
   end

   @tag logged_in: true
-  test "creates resource and redirects when data is valid", %{conn: conn} do
+  test "creates resource and redirects when data is valid", %{conn: conn, user: user} do
     conn = post conn, recipe_path(conn, :create), recipe: @valid_attrs
     assert redirected_to(conn) == recipe_path(conn, :index)
-    assert Repo.get_by(Recipe, @valid_attrs)
+    assert Repo.get_by(Recipe, Map.put(@valid_attrs, :user_id, user.id))
   end

   @tag logged_in: true
```
运行测试：

```bash
mix test
.....................................................

Finished in 0.7 seconds
55 tests, 0 failures
```
全部通过。

## 我的 recipes

我们还有一个情况未处理，用户 A 登录后现在可以查看、编辑、更新用户 B 的菜谱的，我们要禁止这些动作。

写个测试验证一下：

```elixir
diff --git a/test/controllers/recipe_controller_test.exs b/test/controllers/recipe_controller_test.exs
index b901b61..cdbc420 100644
--- a/test/controllers/recipe_controller_test.exs
+++ b/test/controllers/recipe_controller_test.exs
@@ -101,4 +101,19 @@ defmodule TvRecipe.RecipeControllerTest do
     end)
   end

+  @tag logged_in: true
+  test "user should not allowed to show recipe of other people", %{conn: conn, user: user} do
+    # 当前登录用户创建了一个菜谱
+    conn = post conn, recipe_path(conn, :create), recipe: @valid_attrs
+    recipe = Repo.get_by(Recipe, Map.put(@valid_attrs, :user_id, user.id))
+    # 新建一个用户
+    new_user_attrs = %{email: "chenxsan+1@gmail.com", "username": "samchen", password: String.duplicate("1", 6)}
+    Repo.insert! User.changeset(%User{}, new_user_attrs)
+    # 登录新建的用户
+    conn = post conn, session_path(conn, :create), session: new_user_attrs
+    # 读取前头的 recipe 失败，因为它不属于新用户所有
+    assert_error_sent 404, fn ->
+      get conn, recipe_path(conn, :show, recipe)
+    end
+  end
 end
```
运行测试：

```bash
mix test
Compiling 1 file (.ex)
................................

  1) test user should not allowed to show recipe of other people (TvRecipe.RecipeControllerTest)
     test/controllers/recipe_controller_test.exs:105
     expected error to be sent as 404 status, but response sent 200 without error
     stacktrace:
       (phoenix) lib/phoenix/test/conn_test.ex:570: Phoenix.ConnTest.assert_error_sent/2
       test/controllers/recipe_controller_test.exs:115: (test)

.....................

Finished in 0.8 seconds
56 tests, 1 failure
```
Oops，报错了，我们期望响应是 404，却得到 200。

那么该如何取得当前登录用户自有的菜谱？

既然我们前面已经定义过用户与菜谱的关联关系，那么一切应该很容易才是。

是的，Ecto 提供了一个 [`assoc` 方法](https://hexdocs.pm/ecto/Ecto.html#assoc/2)，它能帮我们取得用户关联的所有菜谱。

我们调整下 `recipe_controller.ex` 文件：

```elixir
diff --git a/web/controllers/recipe_controller.ex b/web/controllers/recipe_controller.ex
index 967b7bc..22554ea 100644
--- a/web/controllers/recipe_controller.ex
+++ b/web/controllers/recipe_controller.ex
@@ -33,7 +33,7 @@ defmodule TvRecipe.RecipeController do
   end

   def show(conn, %{"id" => id}) do
-    recipe = Repo.get!(Recipe, id)
+    recipe = Repo.get!(assoc(conn.assigns.current_user, :recipes), id)
     render(conn, "show.html", recipe: recipe)
   end
```
再运行测试：

```bash
$ mix test
..........................

  1) test shows chosen resource (TvRecipe.RecipeControllerTest)
     test/controllers/recipe_controller_test.exs:45
     ** (Ecto.NoResultsError) expected at least one result but got none in query:

     from r in TvRecipe.Recipe,
       where: r.user_id == ^2469,
       where: r.id == ^"645"

     stacktrace:
       (ecto) lib/ecto/repo/queryable.ex:78: Ecto.Repo.Queryable.one!/4
       (tv_recipe) web/controllers/recipe_controller.ex:36: TvRecipe.RecipeController.show/2
       (tv_recipe) web/controllers/recipe_controller.ex:1: TvRecipe.RecipeController.action/2
       (tv_recipe) web/controllers/recipe_controller.ex:1: TvRecipe.RecipeController.phoenix_controller_pipeline/2
       (tv_recipe) lib/tv_recipe/endpoint.ex:1: TvRecipe.Endpoint.instrument/4
       (tv_recipe) lib/phoenix/router.ex:261: TvRecipe.Router.dispatch/2
       (tv_recipe) web/router.ex:1: TvRecipe.Router.do_call/2
       (tv_recipe) lib/tv_recipe/endpoint.ex:1: TvRecipe.Endpoint.phoenix_pipeline/1
       (tv_recipe) lib/tv_recipe/endpoint.ex:1: TvRecipe.Endpoint.call/2
       (phoenix) lib/phoenix/test/conn_test.ex:224: Phoenix.ConnTest.dispatch/5
       test/controllers/recipe_controller_test.exs:47: (test)

...........................

Finished in 0.8 seconds
56 tests, 1 failure
```
我们修复了前面一个错误，但因为我们的修复代码导致了另一个新错误。

检查测试代码，可以发现，旧的测试代码已经不适用了，因为它们新建的 recipe 不包含 `user_id`。我们调整一下：

```elixir
diff --git a/test/controllers/recipe_controller_test.exs b/test/controllers/recipe_controller_test.exs
index cdbc420..d93bbd1 100644
--- a/test/controllers/recipe_controller_test.exs
+++ b/test/controllers/recipe_controller_test.exs
@@ -42,8 +42,8 @@ defmodule TvRecipe.RecipeControllerTest do
   end

   @tag logged_in: true
-  test "shows chosen resource", %{conn: conn} do
-    recipe = Repo.insert! %Recipe{}
+  test "shows chosen resource", %{conn: conn, user: user} do
+    recipe = Repo.insert! %Recipe{user_id: user.id}
     conn = get conn, recipe_path(conn, :show, recipe)
     assert html_response(conn, 200) =~ "Show recipe"
   end
```
再运行测试：

```elixir
$ mix test
......................................................

Finished in 0.8 seconds
56 tests, 0 failures
```
通过了。

但上面我们只修改了 `show` 这个动作，其它几个动作同样需要修改：

```elixir
diff --git a/web/controllers/recipe_controller.ex b/web/controllers/recipe_controller.ex
index 22554ea..f317b59 100644
--- a/web/controllers/recipe_controller.ex
+++ b/web/controllers/recipe_controller.ex
@@ -4,7 +4,7 @@ defmodule TvRecipe.RecipeController do
   alias TvRecipe.Recipe

   def index(conn, _params) do
-    recipes = Repo.all(Recipe)
+    recipes = Repo.all(assoc(conn.assigns.current_user, :recipes))
     render(conn, "index.html", recipes: recipes)
   end

@@ -38,13 +38,13 @@ defmodule TvRecipe.RecipeController do
   end

   def edit(conn, %{"id" => id}) do
-    recipe = Repo.get!(Recipe, id)
+    recipe = Repo.get!(assoc(conn.assigns.current_user, :recipes), id)
     changeset = Recipe.changeset(recipe)
     render(conn, "edit.html", recipe: recipe, changeset: changeset)
   end

   def update(conn, %{"id" => id, "recipe" => recipe_params}) do
-    recipe = Repo.get!(Recipe, id)
+    recipe = Repo.get!(assoc(conn.assigns.current_user, :recipes), id)
     changeset = Recipe.changeset(recipe, recipe_params)

     case Repo.update(changeset) do
@@ -58,7 +58,7 @@ defmodule TvRecipe.RecipeController do
   end

   def delete(conn, %{"id" => id}) do
-    recipe = Repo.get!(Recipe, id)
+    recipe = Repo.get!(assoc(conn.assigns.current_user, :recipes), id)

     # Here we use delete! (with a bang) because we expect
     # it to always work (and if it does not, it will raise).
```
别忘了修复测试：

```elixir
diff --git a/test/controllers/recipe_controller_test.exs b/test/controllers/recipe_controller_test.exs
index d93bbd1..190ede9 100644
--- a/test/controllers/recipe_controller_test.exs
+++ b/test/controllers/recipe_controller_test.exs
@@ -56,30 +56,30 @@ defmodule TvRecipe.RecipeControllerTest do
   end

   @tag logged_in: true
-  test "renders form for editing chosen resource", %{conn: conn} do
-    recipe = Repo.insert! %Recipe{}
   end

   @tag logged_in: true
-  test "renders form for editing chosen resource", %{conn: conn} do
-    recipe = Repo.insert! %Recipe{}
+  test "renders form for editing chosen resource", %{conn: conn, user: user} do
+    recipe = Repo.insert! %Recipe{user_id: user.id}
     conn = get conn, recipe_path(conn, :edit, recipe)
     assert html_response(conn, 200) =~ "Edit recipe"
   end

   @tag logged_in: true
-  test "updates chosen resource and redirects when data is valid", %{conn: conn} do
-    recipe = Repo.insert! %Recipe{}
+  test "updates chosen resource and redirects when data is valid", %{conn: conn, user: user} do
+    recipe = Repo.insert! %Recipe{user_id: user.id}
     conn = put conn, recipe_path(conn, :update, recipe), recipe: @valid_attrs
     assert redirected_to(conn) == recipe_path(conn, :show, recipe)
     assert Repo.get_by(Recipe, @valid_attrs)
   end

   @tag logged_in: true
-  test "does not update chosen resource and renders errors when data is invalid", %{conn: conn} do
-    recipe = Repo.insert! %Recipe{}
+  test "does not update chosen resource and renders errors when data is invalid", %{conn: conn, user: user} do
+    recipe = Repo.insert! %Recipe{user_id: user.id}
     conn = put conn, recipe_path(conn, :update, recipe), recipe: @invalid_attrs
     assert html_response(conn, 200) =~ "Edit recipe"
   end

   @tag logged_in: true
-  test "deletes chosen resource", %{conn: conn} do
-    recipe = Repo.insert! %Recipe{}
+  test "deletes chosen resource", %{conn: conn, user: user} do
+    recipe = Repo.insert! %Recipe{user_id: user.id}
     conn = delete conn, recipe_path(conn, :delete, recipe)
     assert redirected_to(conn) == recipe_path(conn, :index)
     refute Repo.get(Recipe, recipe.id)
```

## 数据的完整性

前面我们在处理 `user_id` 时，顺手删除了 `foreign_key_constraint`，那么，我们要如何处理这样一种情况：用户提交创建菜谱的请求，但用户突然被管理员删除。这时我们的数据库里就会出现无主的菜谱。我们希望避免这种情况。

我们在 `recipe_test.exs` 文件中重新增加一个测试：

```elixir
diff --git a/test/models/recipe_test.exs b/test/models/recipe_test.exs
index 4dbc961..2e2b518 100644
--- a/test/models/recipe_test.exs
+++ b/test/models/recipe_test.exs
@@ -1,7 +1,7 @@
 defmodule TvRecipe.RecipeTest do
   use TvRecipe.ModelCase

-  alias TvRecipe.{Recipe}
+  alias TvRecipe.{Repo, User, Recipe}

   @valid_attrs %{content: "some content", episode: 42, name: "some content", season: 42, title: "some content"}
   @invalid_attrs %{}
@@ -41,4 +41,13 @@ defmodule TvRecipe.RecipeTest do
     assert {:content, "请填写"} in errors_on(%Recipe{}, attrs)
   end

+  test "user must exist" do
+    changeset =
+      %User{id: -1}
+      |> Ecto.build_assoc(:recipes)
+      |> Recipe.changeset(@valid_attrs)
+    {:error, changeset} = Repo.insert changeset
+    assert {:user_id, "does not exist"} in errors_on(changeset)
+  end
+
 end
```
运行测试：

```elixir
mix test
Compiling 13 files (.ex)
.............................................

  1) test user must exist (TvRecipe.RecipeTest)
     test/models/recipe_test.exs:44
     ** (Ecto.ConstraintError) constraint error when attempting to insert struct:

         * foreign_key: recipes_user_id_fkey

     If you would like to convert this constraint into an error, please
     call foreign_key_constraint/3 in your changeset and define the proper
     constraint name. The changeset has not defined any constraint.

     stacktrace:
       (ecto) lib/ecto/repo/schema.ex:493: anonymous fn/4 in Ecto.Repo.Schema.constraints_to_errors/3
       (elixir) lib/enum.ex:1229: Enum."-map/2-lists^map/1-0-"/2
       (ecto) lib/ecto/repo/schema.ex:479: Ecto.Repo.Schema.constraints_to_errors/3
       (ecto) lib/ecto/repo/schema.ex:213: anonymous fn/13 in Ecto.Repo.Schema.do_insert/4
       (ecto) lib/ecto/repo/schema.ex:684: anonymous fn/3 in Ecto.Repo.Schema.wrap_in_transaction/6
       (ecto) lib/ecto/adapters/sql.ex:615: anonymous fn/3 in Ecto.Adapters.SQL.do_transaction/3
       (db_connection) lib/db_connection.ex:1274: DBConnection.transaction_run/4
       (db_connection) lib/db_connection.ex:1198: DBConnection.run_begin/3
       (db_connection) lib/db_connection.ex:789: DBConnection.transaction/3
       test/models/recipe_test.exs:49: (test)

.........

Finished in 0.7 seconds
57 tests, 1 failure
```
很好，`foreign_key_constraint` 的提示又出来了。修改 `recipe.ex` 文件：

```elixr
diff --git a/web/models/recipe.ex b/web/models/recipe.ex
index 8d34ed2..fcc97ad 100644
--- a/web/models/recipe.ex
+++ b/web/models/recipe.ex
@@ -19,5 +19,6 @@ defmodule TvRecipe.Recipe do
     struct
     |> cast(params, [:name, :title, :season, :episode, :content])
     |> validate_required([:name, :title, :season, :episode, :content], message: "请填写")
+    |> foreign_key_constraint(:user_id)
   end
 end
```
再次运行测试：

```bash
mix test
Compiling 13 files (.ex)
.......................................................

Finished in 0.7 seconds
57 tests, 0 failures
```

悉数通过。至此，我们完成了 `RecipeController` 的测试。下一章，我们将接触 View 的测试。