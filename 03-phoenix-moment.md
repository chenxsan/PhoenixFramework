# PhoenixMoment 项目规划

在前几章，我们创建了 PhoenixMoment 项目用于演示。这一章里，我们要对 PhoenixMoment 项目做一个规划。我们的目标是，在教程结束时，你基本掌握 Phoenix 开发的基础，同时完成 [PhoenixMoment 项目的开发与部署](https://github.com/chenxsan/PhoenixMoment)。

简单说，PhoenixMoment 项目在我的设计里，类似轻博客，用户可以用邮箱注册，登录后则可以发布内容，还可以评论、收藏别人的内容。

我们的模块大致划分如下：

1. 用户模块
    1. 注册
2. 会话模块
    1. 登录
    2. 退出登录
3. moment 模块
    1. 发布 moment
    2. 删除 moment
4. 评论模块
    1. 发布评论
    2. 删除评论
5. 收藏模块
    1. 收藏 moment
    2. 取消收藏

下一章，我们就开始[开发用户注册的功能](04-user-register-00.md)。

但在项目开始前，有几个约定需要事先说明。

1. 代码块

    你会在后面的教程里看到大量代码块：

    ```elixir
    diff --git a/test/models/user_test.exs b/test/models/user_test.exs
    index accc8ec..a81988a 100644
    --- a/test/models/user_test.exs
    +++ b/test/models/user_test.exs
    @@ -44,4 +44,9 @@ defmodule PhoenixMoment.UserTest do
        changeset = User.changeset(%User{}, attrs)
        refute changeset.valid?
    end
    +
    +  test "changeset with invalid username should throw errors" do
    +    attrs = %{@valid_attrs | username: "陈三"}
    +    assert {:username, "用户名只允许使用英文字母、数字及下划线"} in errors_on(%User{}, attrs)
    +  end
    end
    ```
    如果你用过 [Git](https://github.com/git/git)，你可能很熟悉，这是 git diff 的结果。
    
    如果你没用过 git，你只需要关注带有 + - 号的代码行，+ 号表示在代码块里增加该行，- 号表示从代码中删除该行。