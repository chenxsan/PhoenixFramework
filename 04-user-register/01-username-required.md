# username 必填

[上一章](00-prepare.md)里，我们用 `mix phoenix.gen.html` 命令创建出完整用户界面，并且具备增加、删除、更改、查询用户的功能。

这一章，我们将实现 `username` 的第一个规则：`username` 必填，如果未填写，提示用户`请填写`。

先来看看，在 `http://localhost:4000/users/new` 页面上，提交空白用户名的话，我们会看到什么？

页面会提示我们，`can't be blank`。

很好，虽然不知道怎么回事，但**必填**的限制已经有了，那如何将它改成`请填写`呢？

打开 `web/models/user.ex` 文件，其中有一行：

```elixir
|> validate_required([:username, :email, :password])
```
正是 [`validate_required`](https://hexdocs.pm/ecto/Ecto.Changeset.html#validate_required/3) 明确了 `username` 为必填。从文档里我们看到，`validate_required` 还接收一个可选的 `message` 参数，用于自定义错误消息。

让我们加上试试：

```elixir
diff --git a/web/models/user.ex b/web/models/user.ex
index b7713a0..87ce321 100644
--- a/web/models/user.ex
+++ b/web/models/user.ex
@@ -15,7 +15,7 @@ defmodule TvRecipe.User do
   def changeset(struct, params \\ %{}) do
     struct
     |> cast(params, [:username, :email, :password])
-    |> validate_required([:username, :email, :password])
+    |> validate_required([:username, :email, :password], message: "请填写")
     |> unique_constraint(:username)
     |> unique_constraint(:email)
   end
```

打开网址，提交空白用户名，页面上已经显示“请填写”了：

![show error when user submit blank username](../img/04-users-blank-username.png)

很好，但请注意，我们这是**人肉测试**。

又或者，我们可以用 Phoenix 生成的测试文件来验证。

打开 `test/models/user_test.exs` 文件，默认内容如下：

```elixir
defmodule TvRecipe.UserTest do
  use TvRecipe.ModelCase

  alias TvRecipe.User

  @valid_attrs %{email: "some content", password: "some content", username: "some content"}
  @invalid_attrs %{}

  test "changeset with valid attributes" do
    changeset = User.changeset(%User{}, @valid_attrs)
    assert changeset.valid?
  end

  test "changeset with invalid attributes" do
    changeset = User.changeset(%User{}, @invalid_attrs)
    refute changeset.valid?
  end
end
```
文件中有两个变量，`@valid_attrs` 表示有效的 `User` 属性，`@invalid_attrs` 表示无效的 `User` 属性，我们按本章开头拟定的规则修改 `@valid_attrs`：

```elixir
diff --git a/test/models/user_test.exs b/test/models/user_test.exs
index 1d5494f..7c73207 100644
--- a/test/models/user_test.exs
+++ b/test/models/user_test.exs
@@ -3,7 +3,7 @@ defmodule TvRecipe.UserTest do

   alias TvRecipe.User

-  @valid_attrs %{email: "some content", password: "some content", username: "some content"}
+  @valid_attrs %{email: "chenxsan@gmail.com", password: "some content", username: "chenxsan"}
   @invalid_attrs %{}

   test "changeset with valid attributes" do
```

接着，在 `user_test.exs` 文件中添加一个新测试：

```elixir
diff --git a/test/models/user_test.exs b/test/models/user_test.exs
index 7c73207..4c174ab 100644
--- a/test/models/user_test.exs
+++ b/test/models/user_test.exs
@@ -15,4 +15,9 @@ defmodule TvRecipe.UserTest do
     changeset = User.changeset(%User{}, @invalid_attrs)
     refute changeset.valid?
   end
+
+  test "username should not be blank" do
+    attrs = %{@valid_attrs | username: ""}
+    assert {:username, "请填写"} in errors_on(%User{}, attrs)
+  end
 end
```

这里，`%{@valid_attrs | username: ""}` 是 Elixir 更新映射（Map）的一个方法。

至于 `errors_on` 函数，它定义在 `tv_recipe/test/support/model_case.ex` 文件中：

```elixir
def errors_on(struct, data) do
  struct.__struct__.changeset(struct, data)
  |> Ecto.Changeset.traverse_errors(&TvRecipe.ErrorHelpers.translate_error/1)
  |> Enum.flat_map(fn {key, errors} -> for msg <- errors, do: {key, msg} end)
end
```
它检查给定数据中的错误消息，并返回给我们。

现在在命令行下运行：

```bash
$ mix test test/models/user_test.exs
```
结果如下：

```bash
...

Finished in 0.07 seconds
3 tests, 0 failures
```
测试通过。现在我们可以放心地认为，用户提交空白 `username` 时，Phoenix 一定会返回“请填写”的错误消息。

下一章，我们将[验证 username 的唯一性](02-username-unique.md)。

## 为什么要写测试？

可能很多人都抱有这个疑问。测试增加了我们的工作量，而它的作用又不那么明显。更何况，这还只是一个入门教程。

我想谈两点个人感受：

1. 我不喜欢拿自己当人肉测试机。
2. 代码的修改是必然的，而我们在修改时无法保证周全，那时测试就是任务清单，它帮我们指出，哪些地方的代码需要做修改，这样我们才能保证代码的质量。

而况在 Phoenix 框架下，测试非常容易写。
