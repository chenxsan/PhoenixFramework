# Recipe 属性开发

在开发用户时，我们曾经分章节完成各个属性。但这里不再细分。

我们来看下 `mix phoenix.gen.html` 命令生成的 `recipe_test.exs` 文件内容：

```elixir
defmodule TvRecipe.RecipeTest do
  use TvRecipe.ModelCase

  alias TvRecipe.Recipe

  @valid_attrs %{content: "some content", episode: 42, name: "some content", season: 42, title: "some content"}
  @invalid_attrs %{}

  test "changeset with valid attributes" do
    changeset = Recipe.changeset(%Recipe{}, @valid_attrs)
    assert changeset.valid?
  end

  test "changeset with invalid attributes" do
    changeset = Recipe.changeset(%Recipe{}, @invalid_attrs)
    refute changeset.valid?
  end
end
```
很显然，默认生成的 `@valid_attrs` 是无效的，因为少了一个 `user_id`。

但我们先处理其它属性，因为比较简单。

我们先增加测试：

```elixir
diff --git a/test/models/recipe_test.exs b/test/models/recipe_test.exs
index a974aad..27f02ea 100644
--- a/test/models/recipe_test.exs
+++ b/test/models/recipe_test.exs
@@ -15,4 +15,29 @@ defmodule TvRecipe.RecipeTest do
     changeset = Recipe.changeset(%Recipe{}, @invalid_attrs)
     refute changeset.valid?
   end
+
+  test "name is required" do
+    attrs = %{@valid_attrs | name: ""}
+    assert {:name, "请填写"} in errors_on(%Recipe{}, attrs)
+  end
+
+  test "title is required" do
+    attrs = %{@valid_attrs | title: ""}
+    assert {:title, "请填写"} in errors_on(%Recipe{}, attrs)
+  end
+
+  test "season is required" do
+    attrs = %{@valid_attrs | season: nil}
+    assert {:season, "请填写"} in errors_on(%Recipe{}, attrs)
+  end
+
+  test "episode is required" do
+    attrs = %{@valid_attrs | episode: nil}
+    assert {:episode, "请填写"} in errors_on(%Recipe{}, attrs)
+  end
+
+  test "content is required" do
+    attrs = %{@valid_attrs | content: ""}
+    assert {:content, "请填写"} in errors_on(%Recipe{}, attrs)
+  end
 end
```
然后修改 `recipe.ex` 文件，自定义验证消息：

```elixir
diff --git a/web/models/recipe.ex b/web/models/recipe.ex
index 946d45c..8d34ed2 100644
--- a/web/models/recipe.ex
+++ b/web/models/recipe.ex
@@ -18,6 +18,6 @@ defmodule TvRecipe.Recipe do
   def changeset(struct, params \\ %{}) do
     struct
     |> cast(params, [:name, :title, :season, :episode, :content])
-    |> validate_required([:name, :title, :season, :episode, :content])
+    |> validate_required([:name, :title, :season, :episode, :content], message: "请填写")
   end
 end
```
测试全部通过了。

## `user_id`

`user_id` 有两条规则：

1. `user_id` 必填
2. `user_id` 对应着 `users` 表中用户的 id，该用户在表中必须存在

### 必填

我们先处理 `user_id` 必填的规则，补充一个测试，如下：

```elixir
diff --git a/test/models/recipe_test.exs b/test/models/recipe_test.exs
index 27f02ea..3a9630b 100644
--- a/test/models/recipe_test.exs
+++ b/test/models/recipe_test.exs
@@ -3,7 +3,7 @@ defmodule TvRecipe.RecipeTest do

   alias TvRecipe.Recipe

-  @valid_attrs %{content: "some content", episode: 42, name: "some content", season: 42, title: "some content"}
+  @valid_attrs %{content: "some content", episode: 42, name: "some content", season: 42, title: "some content", user_id: 1}
   @invalid_attrs %{}

   test "changeset with valid attributes" do
@@ -40,4 +40,9 @@ defmodule TvRecipe.RecipeTest do
     attrs = %{@valid_attrs | content: ""}
     assert {:content, "请填写"} in errors_on(%Recipe{}, attrs)
   end
+
+  test "user_id is required" do
+    attrs = %{@valid_attrs | user_id: nil}
+    assert {:user_id, "请填写"} in errors_on(%Recipe{}, attrs)
+  end
 end
```
然后修改 `recipe.ex` 文件：

```elixir
diff --git a/web/models/recipe.ex b/web/models/recipe.ex
index 8d34ed2..0520582 100644
--- a/web/models/recipe.ex
+++ b/web/models/recipe.ex
@@ -17,7 +17,7 @@ defmodule TvRecipe.Recipe do
   """
   def changeset(struct, params \\ %{}) do
     struct
-    |> cast(params, [:name, :title, :season, :episode, :content])
-    |> validate_required([:name, :title, :season, :episode, :content], message: "请填写")
+    |> cast(params, [:name, :title, :season, :episode, :content, :user_id])
+    |> validate_required([:name, :title, :season, :episode, :content, :user_id], message: "请填写")
   end
 end
```
运行新增的这个测试：

```bash
$ mix test test/models/recipe_test.exs:44
mix test test/models/recipe_test.exs:44
Including tags: [line: "44"]
Excluding tags: [:test]

.

Finished in 0.1 seconds
8 tests, 0 failures, 7 skipped
```
注意，我们只测试前面新增的测试，`:44` 表示执行该文件中第 44 行开始的 `test` 块。

### `user_id` 所指向的用户应存在

我们在 `recipe_test.exs` 文件中再增加一个测试：

```elixir
diff --git a/test/models/recipe_test.exs b/test/models/recipe_test.exs
index 3a9630b..2e1191c 100644
--- a/test/models/recipe_test.exs
+++ b/test/models/recipe_test.exs
@@ -1,7 +1,7 @@
 defmodule TvRecipe.RecipeTest do
   use TvRecipe.ModelCase

-  alias TvRecipe.Recipe
+  alias TvRecipe.{Repo, Recipe}

   @valid_attrs %{content: "some content", episode: 42, name: "some content", season: 42, title: "some content", user_id: 1}
   @invalid_attrs %{}
@@ -45,4 +45,9 @@ defmodule TvRecipe.RecipeTest do
     attrs = %{@valid_attrs | user_id: nil}
     assert {:user_id, "请填写"} in errors_on(%Recipe{}, attrs)
   end
+
+  test "user_id should exist in users table" do
+    {:error, changeset} = Repo.insert Recipe.changeset(%Recipe{}, @valid_attrs)
+    assert {:user_id, "用户不存在"} in errors_on(changeset)
+  end
 end
```
运行新增的测试：

```bash
$ mix test test/models/recipe_test.exs:49
Compiling 13 files (.ex)
Including tags: [line: "49"]
Excluding tags: [:test]



  1) test user_id should exist in users table (TvRecipe.RecipeTest)
     test/models/recipe_test.exs:49
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
       test/models/recipe_test.exs:50: (test)



Finished in 0.2 seconds
9 tests, 1 failure, 8 skipped
```
测试失败，但 Phoenix 给了 [foreign_key_constraint](https://hexdocs.pm/ecto/Ecto.Changeset.html#foreign_key_constraint/3) 的提示：

```elixir
diff --git a/web/models/recipe.ex b/web/models/recipe.ex
index 0520582..a0b42fd 100644
--- a/web/models/recipe.ex
+++ b/web/models/recipe.ex
@@ -19,5 +19,6 @@ defmodule TvRecipe.Recipe do
     struct
     |> cast(params, [:name, :title, :season, :episode, :content, :user_id])
     |> validate_required([:name, :title, :season, :episode, :content, :user_id], message: "请填写")
+    |> foreign_key_constraint(:user_id, message: "用户不存在")
   end
 end
```
再次运行测试：

```bash
$ mix test test/models/recipe_test.exs:49
Compiling 13 files (.ex)
Including tags: [line: "49"]
Excluding tags: [:test]

.

Finished in 0.1 seconds
9 tests, 0 failures, 8 skipped
```
测试通过。

但我们运行所有测试的话：

```bash
$ mix test
..........................................

  1) test creates resource and redirects when data is valid (TvRecipe.RecipeControllerTest)
     test/controllers/recipe_controller_test.exs:18
     ** (RuntimeError) expected redirection with status 302, got: 200
     stacktrace:
       (phoenix) lib/phoenix/test/conn_test.ex:443: Phoenix.ConnTest.redirected_to/2
       test/controllers/recipe_controller_test.exs:20: (test)



  2) test updates chosen resource and redirects when data is valid (TvRecipe.RecipeControllerTest)
     test/controllers/recipe_controller_test.exs:47
     ** (RuntimeError) expected redirection with status 302, got: 200
     stacktrace:
       (phoenix) lib/phoenix/test/conn_test.ex:443: Phoenix.ConnTest.redirected_to/2
       test/controllers/recipe_controller_test.exs:50: (test)

..........

Finished in 0.6 seconds
54 tests, 2 failures
```
`recipe_controller_test.exs` 文件中出现两个错误 - 不过我们留给下一章处理。