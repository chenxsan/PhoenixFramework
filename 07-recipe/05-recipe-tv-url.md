# 添加视频地址

我们的菜谱现在有对应的节目名、哪一季哪一集，如果能直接附上视频的链接，就更完美了。

我们需要一个数据库迁移（migration）文件：

```bash
$ mix ecto.gen.migration add_url_to_recipe
* creating priv/repo/migrations
* creating priv/repo/migrations/20170211030550_add_url_to_recipe.exs
```
修改新建的迁移文件：

```elixir
diff --git a/priv/repo/migrations/20170211030550_add_url_to_recipe.exs b/priv/repo/migrations/20170211030550_add_url_to_recipe.exs
index 01e5f17..f0918c6 100644
--- a/priv/repo/migrations/20170211030550_add_url_to_recipe.exs
+++ b/priv/repo/migrations/20170211030550_add_url_to_recipe.exs
@@ -2,6 +2,8 @@ defmodule TvRecipe.Repo.Migrations.AddUrlToRecipe do
   use Ecto.Migration

   def change do
-
+    alter table(:recipes) do
+      add :url, :string
+    end
   end
 end
```
接着将 `:url` 加入 schema 中：

```elixir
diff --git a/web/models/recipe.ex b/web/models/recipe.ex
index 230f290..104db50 100644
--- a/web/models/recipe.ex
+++ b/web/models/recipe.ex
@@ -6,6 +6,7 @@ defmodule TvRecipe.Recipe do
     field :title, :string
     field :season, :integer, default: 1
     field :episode, :integer, default: 1
+    field :url, :string
     field :content, :string
     belongs_to :user, TvRecipe.User

@@ -17,7 +18,7 @@ defmodule TvRecipe.Recipe do
   """
   def changeset(struct, params \\ %{}) do
     struct
-    |> cast(params, [:name, :title, :season, :episode, :content])
+    |> cast(params, [:name, :title, :season, :episode, :content, :url])
     |> validate_required([:name, :title, :season, :episode, :content], message: "请填写")
     |> validate_number(:season, greater_than: 0, message: "请输入大于 0 的数字")
     |> validate_number(:episode, greater_than: 0, message: "请输入大于 0 的数字")
```

最后，执行 `mix ecto.migrate`：

```bash
$ mix ecto.migrate
Compiling 13 files (.ex)

11:53:37.646 [info]  == Running TvRecipe.Repo.Migrations.AddUrlToRecipe.change/0 forward

11:53:37.646 [info]  alter table recipes

11:53:37.676 [info]  == Migrated in 0.0s
```
接着新增一个测试，我们需要验证 url 的有效性：

```elixir
diff --git a/test/models/recipe_test.exs b/test/models/recipe_test.exs
index 8b093ed..f1ba3f9 100644
--- a/test/models/recipe_test.exs
+++ b/test/models/recipe_test.exs
@@ -60,4 +60,9 @@ defmodule TvRecipe.RecipeTest do
     assert {:user_id, "does not exist"} in errors_on(changeset)
   end

+  test "url should be valid" do
+    attrs = Map.put(@valid_attrs, :url, "fjsalfa")
+    assert {:url, "url 错误"} in errors_on(%Recipe{}, attrs)
+  end
+
 end
```
运行测试：

```bash
mix test
...............................

  1) test url should be valid (TvRecipe.RecipeTest)
     test/models/recipe_test.exs:63
     Assertion with in failed
     code:  {:url, "url 错误"} in errors_on(%Recipe{}, attrs)
     left:  {:url, "url 错误"}
     right: []
     stacktrace:
       test/models/recipe_test.exs:65: (test)

...........................

Finished in 1.0 seconds
59 tests, 1 failure
```
那么我们要如何在 `recipe.ex` 文件中验证 url 的有效性？

我们可以考虑用正则表达式配合 `validate_format`，但有个更好的办法，是直接引用 Erlang 的方法：

```elixir
diff --git a/web/models/recipe.ex b/web/models/recipe.ex
index 104db50..3b849c8 100644
--- a/web/models/recipe.ex
+++ b/web/models/recipe.ex
@@ -22,6 +22,16 @@ defmodule TvRecipe.Recipe do
     |> validate_required([:name, :title, :season, :episode, :content], message: "请填写")
     |> validate_number(:season, greater_than: 0, message: "请输入大于 0 的数字")
     |> validate_number(:episode, greater_than: 0, message: "请输入大于 0 的数字")
+    |> validate_url(:url)
     |> foreign_key_constraint(:user_id)
   end
+
+  defp validate_url(changeset, field, _options \\ []) do
+    validate_change changeset, field, fn _, url ->
+      case url |> String.to_charlist |> :http_uri.parse do
+        {:ok, _} -> []
+        {:error, _} -> [url: "url 错误"]
+      end
+    end
+  end
 end
```
我们在 `recipe.ex` 文件中新增了一个 `validate_url` 私有方法，并调用 Ecto 提供的 [validate_change](https://hexdocs.pm/ecto/Ecto.Changeset.html#validate_change/3) 函数来验证属性是否有效。[http_uri](http://erlang.org/doc/man/http_uri.html) 是 Erlang 的模块，在 Elixir 中，我们能够以 `:http_uri` 的形式调用。

我们所有的 recipe 模板都需要做调整 - 此时，我想你可能已经意识到测试驱动的好处了，如果我们给各个模板添加过测试，那么有新特性加入时，我们先在测试中体现我们的目的，然后运行测试，就知道需要修改哪些文件来达到我们的目的。

参照前一节的代码，我们来进一步完善 `RecipeViewTest` 模块的代码：

```elixir
diff --git a/test/views/recipe_view_test.exs b/test/views/recipe_view_test.exs
index 8174c14..9695647 100644
--- a/test/views/recipe_view_test.exs
+++ b/test/views/recipe_view_test.exs
@@ -28,4 +28,23 @@ defmodule TvRecipe.RecipeViewTest do
     end
   end

+  test "render new.html", %{conn: conn} do
+    changeset = Recipe.changeset(%Recipe{})
+    content = render_to_string(TvRecipe.RecipeView, "new.html", conn: conn, changeset: changeset)
+    assert String.contains?(content, "url")
+  end
+
+  test "render show.html", %{conn: conn} do
+    recipe = %Recipe{id: "1", name: "淘米", title: "侠饭", season: "1", episode: "1", content: "洗掉米表面的淀粉", user_id: "999", url: "https://github.com/chenxsan/PhoenixFramework"}
+    content = render_to_string(TvRecipe.RecipeView, "show.html", conn: conn, recipe: recipe)
+    assert String.contains?(content, recipe.url)
+  end
+
+  test "render edit.html", %{conn: conn} do
+    recipe = %Recipe{id: "1", name: "淘米", title: "侠饭", season: "1", episode: "1", content: "洗掉米表面的淀粉", user_id: "999", url: "https://github.com/chenxsan/PhoenixFramework"}
+    changeset = Recipe.changeset(recipe)
+    content = render_to_string(TvRecipe.RecipeView, "edit.html", conn: conn, changeset: changeset, recipe: recipe)
+    assert String.contains?(content, recipe.url)
+  end
+
 end
```
然后运行测试：

```bash
mix test
Compiling 1 file (.ex)
...

  1) test render new.html (TvRecipe.RecipeViewTest)
     test/views/recipe_view_test.exs:31
     Expected truthy, got false
     code: String.contains?(content, "url")
     stacktrace:
       test/views/recipe_view_test.exs:34: (test)



  2) test render show.html (TvRecipe.RecipeViewTest)
     test/views/recipe_view_test.exs:37
     Expected truthy, got false
     code: String.contains?(content, recipe.url())
     stacktrace:
       test/views/recipe_view_test.exs:40: (test)

.

  3) test render edit.html (TvRecipe.RecipeViewTest)
     test/views/recipe_view_test.exs:43
     Expected truthy, got false
     code: String.contains?(content, recipe.url())
     stacktrace:
       test/views/recipe_view_test.exs:47: (test)

.......................................................

Finished in 0.9 seconds
62 tests, 3 failures
```
根据测试结果，我们修改文件：

```elixir
diff --git a/web/templates/recipe/form.html.eex b/web/templates/recipe/form.html.eex
index 3bf90ff..ab12be3 100644
--- a/web/templates/recipe/form.html.eex
+++ b/web/templates/recipe/form.html.eex
@@ -30,6 +30,12 @@
   </div>

   <div class="form-group">
+    <%= label f, :url, class: "control-label" %>
+    <%= text_input f, :url, class: "form-control" %>
+    <%= error_tag f, :url %>
+  </div>
+
+  <div class="form-group">
     <%= label f, :content, class: "control-label" %>
     <%= textarea f, :content, class: "form-control" %>
     <%= error_tag f, :content %>
diff --git a/web/templates/recipe/show.html.eex b/web/templates/recipe/show.html.eex
index 3ef437d..f4ea463 100644
--- a/web/templates/recipe/show.html.eex
+++ b/web/templates/recipe/show.html.eex
@@ -23,6 +23,11 @@
   </li>

   <li>
+    <strong>Url:</strong>
+    <%= @recipe.url %>
+  </li>
+
+  <li>
     <strong>Content:</strong>
     <%= @recipe.content %>
   </li>
```
最后再运行一次测试：

```elixir
mix test
Compiling 1 file (.ex)
..............................................................

Finished in 0.9 seconds
62 tests, 0 failures
```
测试全部通过。