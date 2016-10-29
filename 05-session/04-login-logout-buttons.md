# 登录/注册按钮

我们至今还没在页面上添加登录/注册按钮，都是直接在浏览器中输入网址访问，这对普通用户来说极度不友好。

老规则，先写测试，我们不再新增，而是加在旧测试中：

```elixir
diff --git a/test/controllers/session_controller_test.exs b/test/controllers/session_controller_test.exs
index c50466d..298d757 100644
--- a/test/controllers/session_controller_test.exs
+++ b/test/controllers/session_controller_test.exs
@@ -11,6 +11,10 @@ defmodule PhoenixMoment.SessionControllerTest do
   describe "create/2" do
     # 用户名/密码组合正确，跳转到站点主页，并显示“欢迎回来”
     test "logins user and redirects when data is valid", %{conn: conn} do
+      # 未登录情况下访问首页，应带有登录/注册文字
+      conn = get conn, page_path(conn, :index)
+      assert html_response(conn, 200) =~ "登录"
+      assert html_response(conn, 200) =~ "注册"
       # 在数据库中新建一个用户
       changeset = User.changeset(%User{}, @valid_attrs)
       Repo.insert!(changeset)
```

打开 `app.html.eex` 文件，添加两个按钮：

```eex
diff --git a/web/templates/layout/app.html.eex b/web/templates/layout/app.html.eex
index aa95174..5badfd7 100644
--- a/web/templates/layout/app.html.eex
+++ b/web/templates/layout/app.html.eex
@@ -20,6 +20,9 @@
             <%= if @current_user do %>
               <li><%= @current_user.username %></li>
               <li><%= link "退出", to: session_path(@conn, :delete, @current_user), method: "delete" %></li>
+            <% else %>
+              <li><%= link "登录", to: session_path(@conn, :new) %></li>
+              <li><%= link "注册", to: user_path(@conn, :new) %></li>
             <% end %>
           </ul>
         </nav>
```
运行测试，通过。

下一章，我们要对[用户列表页面做个限制](05-restrict-access.md)。