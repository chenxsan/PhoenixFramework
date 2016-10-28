# password 规则

`password` 的规则与 [`email` 规则类似](06-email-rules.md)，同样有三条：

限制|错误提示
---|---
必填|请填写
密码最短 6 位|密码最短 6 位
密码不能明文存储在数据库中|-

但我们会分为两章完成，这一章里完成前面两条规则，“密码不能明文存储”规则因为比较复杂，放到下一章里。

首先，是添加两个测试：

```elixir
diff --git a/test/models/user_test.exs b/test/models/user_test.exs
index 8d8f3e1..f3a7c44 100644
--- a/test/models/user_test.exs
+++ b/test/models/user_test.exs
@@ -140,4 +140,14 @@ defmodule PhoenixMoment.UserTest do
     # 断言
     assert {:email, "邮箱已被人占用"} in errors
   end
+
+  test "password is required" do
+    attrs = %{@valid_attrs | password: ""}
+    assert {:password, "请填写"} in errors_on(%User{}, attrs)
+  end
+
+  test "password's length should be larger than 6" do
+    attrs = %{@valid_attrs | password: "12345"}
+    assert {:password, "密码最短 6 位"} in errors_on(%User{}, attrs)
+  end
 end
```

一个验证 `password` 必填，一个验证密码长度。

这两个测试，必填的一个会通过，因为在处理 `username` 时一起处理了；验证密码长度的则会失败：

```bash
$ mix test test/models
.............

  1) test password's length should be larger than 6 (PhoenixMoment.UserTest)
     test/models/user_test.exs:149
     Assertion with in failed
     code: {:password, "密码最短 6 位"} in errors_on(%User{}, attrs)
     lhs:  {:password, "密码最短 6 位"}
     rhs:  []
     stacktrace:
       test/models/user_test.exs:151: (test)

....

Finished in 0.2 seconds
18 tests, 1 failure
```
因为我们还没有加上限制条件。

打开 `user.ex` 文件，添加 `validate_length`：

```elixir
diff --git a/web/models/user.ex b/web/models/user.ex
index 19aa630..0796d93 100644
--- a/web/models/user.ex
+++ b/web/models/user.ex
@@ -23,5 +23,6 @@ defmodule PhoenixMoment.User do
     |> validate_exclusion(:username, ~w(admin administrator), message: "系统保留，无法注册，请更换")
     |> validate_format(:email, ~r/@/, message: "邮箱格式错误")
     |> unique_constraint(:email, name: :users_lower_email_index, message: "邮箱已被人占用")
+    |> validate_length(:password, min: 6, message: "密码最短 6 位")
   end
 end
 ```
再运行测试，全部通过。

这样，我们就完成了 `password` 的两条验证规则，[下一章](08-password-storage.md)，我们将探索如何安全存储密码。