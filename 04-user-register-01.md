# username 必填

[上一章](04-user-register.md)里，我们用 `mix phoenix.gen.html` 命令创建出完整的用户界面，并且具备增加、删除、更改、查询用户的功能。

这一章里，我们将实现 `username` 的第一个规则：`username` 为必填项，如果未填写，提示用户`请填写`。

先来看看，在 `http://localhost:4000/users/new` 页面上，提交空用户名的话，我们会看到什么？

页面会提示我们，`can't be blank`。

很好，**必填**的限制已经莫名有了，那如何将它改成`请填写`呢？

打开 `web/models/user.ex` 文件，其中有一行：

```elixir
|> validate_required([:username, :email, :password])
```
正是 [`validate_required`](https://hexdocs.pm/ecto/Ecto.Changeset.html#validate_required/3) 指定了 `username` 为必填 - 可不是上面说的“莫名有了”。从文档里我们看到，`validate_required` 还接收一个 `message` 参数，用于自定义错误提示。

让我们加上试试：

```elixir
diff --git a/web/models/user.ex b/web/models/user.ex
index 90e7701..a7005fa 100644
--- a/web/models/user.ex
+++ b/web/models/user.ex
@@ -15,7 +15,7 @@ defmodule PhoenixMoment.User do
   def changeset(struct, params \\ %{}) do
     struct
     |> cast(params, [:username, :email, :password])
-    |> validate_required([:username, :email, :password])
+    |> validate_required([:username, :email, :password], message: "请填写")
     |> unique_constraint(:username)
     |> unique_constraint(:email)
   end
```

打开网址，提交空的用户名，页面上已经显示“请填写”了：

![show error when user submit blank username](img/04-users-blank-username.png)

但这是人肉测试。

又或者，我们可以用 Phoenix 生成的测试文件来验证。

打开 `test/models/user_test.exs` 文件，默认内容如下：

```elixir
defmodule PhoenixMoment.UserTest do
  use PhoenixMoment.ModelCase

  alias PhoenixMoment.User

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
文件中有一个 `@valid_attrs` 变量，一个 `@invalid_attrs` 变量，我们且按本章开头确定的规则订正 `@valid_attrs`：

```elixir
diff --git a/test/models/user_test.exs b/test/models/user_test.exs
index 3da2e40..a78967c 100644
--- a/test/models/user_test.exs
+++ b/test/models/user_test.exs
@@ -3,7 +3,7 @@ defmodule PhoenixMoment.UserTest do

   alias PhoenixMoment.User

-  @valid_attrs %{email: "some content", password: "some content", username: "some content"}
+  @valid_attrs %{email: "a@b.com", password: "some content", username: "chenxsan"}
   @invalid_attrs %{}
```

修改后的 `@valid_attrs` 是我们的正常数据。

接着，在 `user_test.exs` 文件中添加一个新测试：

```elixir
diff --git a/test/models/user_test.exs b/test/models/user_test.exs
index a78967c..48c0f6c 100644
--- a/test/models/user_test.exs
+++ b/test/models/user_test.exs
@@ -15,4 +15,9 @@ defmodule PhoenixMoment.UserTest do
     changeset = User.changeset(%User{}, @invalid_attrs)
     refute changeset.valid?
   end
+
+  test "blank username should throw error" do
+    attrs = %{@valid_attrs | username: ""}
+    assert {:username, "请填写"} in errors_on(%User{}, attrs)
+  end
 end
```

这里，`%{@valid_attrs | username: ""}` 是 Elixir 更新对象的方法。

接着在命令行下运行：

```bash
$ mix test test/models
```
因为我们现在只测试模型，所以只针对 `test/models` 目录下的测试文件。

结果如下：

```bash
...

Finished in 0.08 seconds
3 tests, 0 failures
```
测试通过。

## 为什么要写测试？

可能很多人都抱有这个疑问。测试增加了我们的工作量，而它的作用又不那么明显。就实际项目开发来说，绝大多数人也是不写测试的。

那为什么还要写测试？

我只谈一点个人感受：我不喜欢拿自己当人肉测试机。我不喜欢列一个清单，然后一项一项人肉测试过去。等下一次代码变动，还照着清单一项项人肉测试过去。

代码能做的事，就不要人来做。这是提高生活质量的一大要义。

而况 Phoenix 框架下，测试非常容易写。
