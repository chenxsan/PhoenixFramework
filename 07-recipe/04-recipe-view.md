# 菜谱视图

我们执行 [`mix phoenix.gen.html` 命令](https://github.com/phoenixframework/phoenix/blob/master/lib/mix/tasks/phoenix.gen.html.ex#L14)时，它会生成如下文件：

* a schema in web/models
* a view in web/views
* a controller in web/controllers
* a migration file for the repository
* default CRUD templates in web/templates
* test files for generated model and controller

其中有两个测试文件，但是没有视图的测试文件 - 为什么？难道视图不重要？不不不，只要是代码，都会有测试的必要 - 有时没写，只是一个优先级或是投入产出比上的考虑。那如何入手？

查看 `test/views` 目录，现在已经有三个文件。我们可以借鉴其中的 `error_view_test.exs` 文件。

首先在 `test/views` 目录下新建一个 `recipe_view_test.exs` 文件，然后准备好如下内容：

```elixir
defmodule TvRecipe.RecipeViewTest do
  use TvRecipe.ConnCase, async: true

  # Bring render/3 and render_to_string/3 for testing custom views
  import Phoenix.View

end
```
前面章节里我们曾经提到过，templates 文件会被编译成 view 模块下的函数。比如我们的 `web/templates/recipe/index.html.eex` 文件，最终会变成 `TvRecipe.RecipeView` 模块中的一个函数：

```elixir
def render("index.html", assigns) do
  # 返回编译后的 eex 模板
end
```
换句话说，我们其实可以在 `TvRecipe.RecipeView` 中定义 `render("index.html", assigns)`，而不必再写一个 `index.html.eex` 模板。只是模板对我们的开发更为友好，所以才从 View 中分离出来。

因此，测试模板即测试 View 中的函数。

那么，测试什么？测试我们的期望与事实是否相符。

举 `recipe/index.html.eex` 模板说，我们想在模板页面上显示哪些数据？Phoenix 最后的输出是否确保显示了？

来加一个测试：

```elixir
diff --git a/test/views/recipe_view_test.exs b/test/views/recipe_view_test.exs
index be4148a..8174c14 100644
--- a/test/views/recipe_view_test.exs
+++ b/test/views/recipe_view_test.exs
@@ -4,4 +4,28 @@ defmodule TvRecipe.RecipeViewTest do
   # Bring render/3 and render_to_string/3 for testing custom views
   import Phoenix.View

+  alias TvRecipe.Recipe
+
+  test "render index.html", %{conn: conn} do
+    recipes = [%Recipe{id: "1", name: "淘米", title: "侠饭", season: "1", episode: "1", content: "洗掉米表面的淀粉", user_id: "999"},
+      %Recipe{id: "2", name: "煮饭", title: "侠饭", season: "1", episode: "1", content: "浸泡", user_id: "888"}]
+    content = render_to_string(TvRecipe.RecipeView, "index.html", conn: conn, recipes: recipes)
+    # 页面上包含标题 Listing recipes
+    assert String.contains?(content, "Listing recipes")
+    for recipe <- recipes do
+      # 页面上包含菜谱名
+      assert String.contains?(content, recipe.name)
+      # 页面上包含节目名
+      assert String.contains?(content, recipe.title)
+      # 包含 season
+      assert String.contains?(content, recipe.season)
+      # 包含 episode
+      assert String.contains?(content, recipe.episode)
+      # 不包含所有者 id
+      refute String.contains?(content, recipe.user_id)
+      # 因为 content 很长，我们不在 index.html 里显示
+      refute String.contains?(content, recipe.content)
+    end
+  end
+
 end
```
你可能要问，测试里的 `season` 为什么是 "1" 而不是 1，要知道我们给它定义的类型是 `integer`。

是的，我们本应该把它写为 1，但那样的话，我们的测试代码里就要做类型转换：

```elixir
assert String.contains?(content, Integer.to_string(recipe.season))
```
为了省事，我们就直接把 1 写成了 "1" - 这并不会有问题，因为 Phoenix 在编译模板时，也会做[类型转换](https://hexdocs.pm/eex/EEx.html#eval_string/3)：

```bash
iex> EEx.eval_string "foo <%= bar %>", [bar: 123]
"foo 123"
```
运行测试：

```bash
$ mix test
Compiling 1 file (.ex)
...

  1) test render index.html (TvRecipe.RecipeViewTest)
     test/views/recipe_view_test.exs:9
     Expected false or nil, got true
     code: String.contains?(content, recipe.user_id())
     stacktrace:
       test/views/recipe_view_test.exs:25: anonymous fn/3 in TvRecipe.RecipeViewTest.test render index.html/1
       (elixir) lib/enum.ex:1755: Enum."-reduce/3-lists^foldl/2-0-"/3
       test/views/recipe_view_test.exs:15: (test)

....................................................

Finished in 0.8 seconds
58 tests, 1 failure
```
一个错误发生，因为目前的 `index.html.eex` 中包含了 `content` 与 `user_id` 的内容。

我们调整下 `index.html.eex` 文件：

```eex
diff --git a/web/templates/recipe/index.html.eex b/web/templates/recipe/index.html.eex
index b6ff40b..1dc8a3a 100644
--- a/web/templates/recipe/index.html.eex
+++ b/web/templates/recipe/index.html.eex
@@ -7,8 +7,6 @@
       <th>Title</th>
       <th>Season</th>
       <th>Episode</th>
-      <th>Content</th>
-      <th>User</th>

       <th></th>
     </tr>
@@ -20,8 +18,6 @@
       <td><%= recipe.title %></td>
       <td><%= recipe.season %></td>
       <td><%= recipe.episode %></td>
-      <td><%= recipe.content %></td>
-      <td><%= recipe.user_id %></td>

       <td class="text-right">
         <%= link "Show", to: recipe_path(@conn, :show, recipe), class: "btn btn-default btn-xs" %>
```

再运行测试：

```bash
mix test
........................................................

Finished in 0.8 seconds
58 tests, 0 failures
```
悉数通过。

我们知道，`index.html.eex` 页面上有一个 `New recipe` 的按钮，那我们的测试里是否需要体现？`Show`、`Edit`、`Delete` 这些按钮呢？要不要给它们写测试？

测试测什么，测到怎么的粒度，我觉得没有标准答案，更多时候要根据项目情况去权衡。但如果一定要有一个什么做为参考，我会选择设计稿。设计稿上定下来的元素，我们就尽量在测试里体现 - 否则设计人员很容易找上门来，说怎么少了这少了那。

在结束本节之前，别忘了在菜单栏上加上“菜谱”，不然我们就只能通过修改 url 访问菜谱相关页面了：

1. _test/controllers/user_controller_test.exs_

  ```elixir
      conn = get conn, page_path(conn, :index)
      assert html_response(conn, 200) =~ Map.get(@valid_attrs, :username)
  +   assert html_response(conn, 200) =~ "菜谱"
    end
  ```
2. _web/templates/layout/app.html.eex_

  ```eex
              <li><a href="http://www.phoenixframework.org/docs">Get Started</a></li>
              <%= if @current_user do %>
                <li><%= link @current_user.username, to: user_path(@conn, :show, @current_user) %></li>
  +             <li><%= link "菜谱", to: recipe_path(@conn, :index) %></li>
                <li><%= link "退出", to: session_path(@conn, :delete, @current_user), method: "delete" %></li>
              <% else %>
                <li><%= link "登录", to: session_path(@conn, :new) %></li>
  ```

运行测试：

```bash
$ mix test
..........................................................

Finished in 0.8 seconds
58 tests, 0 failures
```