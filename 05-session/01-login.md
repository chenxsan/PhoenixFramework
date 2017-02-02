# 登录

这一次，我们没有 `mix phoenix.gen.html` 可以用，所以要一步一步写了。

它的过程，跟[添加帮助文件一章](../02-explore-phoenix.md)一样。

但这里，我们要从测试写起，运行它，看着它抛出错误，之后才填补代码，保证每个测试通过。

Don't panic，错误是指引我们成功的路灯。

## 添加路由

首先在 `test/controllers` 目录下新建一个 `session_controller_test.exs` 文件：

```elixir
defmodule TvRecipe.SessionControllerTest do
  use TvRecipe.ConnCase
end
```

我们希望在用户访问 `/sessions/new` 网址时，返回一个登录页面。虽然目前我们还不清楚 Phoenix 下的测试代码究竟是什么原理，但没关系，我们可以参考 `user_controller_test.exs` 测试文件照猫画虎：

```elixir
  test "renders form for new sessions", %{conn: conn} do
    conn = get conn, session_path(conn, :new)
    # 200 响应，页面上带有“登录”
    assert html_response(conn, 200) =~ "登录"
  end
```
运行测试，结果如下：

```bash
$ mix test test/controllers/session_controller_test.exs
** (CompileError) test/controllers/session_controller_test.exs:5: undefined function session_path/2
    (stdlib) lists.erl:1338: :lists.foreach/2
    (stdlib) erl_eval.erl:670: :erl_eval.do_apply/6
    (elixir) lib/code.ex:370: Code.require_file/2
    (elixir) lib/kernel/parallel_require.ex:57: anonymous fn/2 in Kernel.ParallelRequire.spawn_requires/5
```

`session_path` 函数未定义。要怎么定义，在哪定义？

实际上，在前面的章节里，我们已经遭遇过 `user_path`，但还没有解释过它从哪里来。

我们来看 [Phoenix.Router](https://hexdocs.pm/phoenix/Phoenix.Router.html) 的文档，其中 **Helpers** 一节有说明如下：

> Phoenix automatically generates a module Helpers inside your router which contains named helpers to help developers generate and keep their routes up to date.

> Helpers are automatically generated based on the controller name. 

我们在 `router.ex` 文件中定义 `TvRecipe.Router` 模块，而 Phoenix 会在该模块下生成一个 `TvRecipe.Router.Helpers` 模块，用于管理我们的路由。`Helpers` 下的内容，基于控制器的名称生成。

比如我们有一个路由：

```elixir
get "/", PageController, :index
```
则 Phoenix 会自动生成 `TvRecipe.Router.Helpers.page_path`。

那么，前面章节里 `user_path` 出现时，是在控制器与模板中，并且是光秃秃的 `user_path`，而不是 `TvRecipe.Router.Helpers.user_path` 这样冗长写法，它们究竟是怎样引用的？

我们回头去看控制器的代码，会在开头处看到这么一行：

```elixir
use TvRecipe.Web, :controller
```

而 `TvRecipe.Web` 是定义在 `web/web.ex` 文件，其中会有这样的内容：

```elixir
  def controller do
    quote do
      use Phoenix.Controller

      alias TvRecipe.Repo
      import Ecto
      import Ecto.Query

      import TvRecipe.Router.Helpers
      import TvRecipe.Gettext
    end
  end
  ```
我们看到了 `import TvRecipe.Router.Helpers` 一行，这正是我们在控制器中可以直接使用 `user_path` 等函数的原因 - `use TvRecipe.Web, :controller` 做了准备工作。

现在，我们知道要怎么定义 `session_path` 了。

打开 `router.ex` 文件，添加一个新路由：

```elixir
diff --git a/web/router.ex b/web/router.ex
index 4ddc1cc..aac327c 100644
--- a/web/router.ex
+++ b/web/router.ex
@@ -18,6 +18,7 @@ defmodule TvRecipe.Router do

     get "/", PageController, :index
     resources "/users", UserController
+    get "/sessions/new", SessionController, :new
   end
```
运行测试：

```bash
mix test test/controllers/session_controller_test.exs
Compiling 8 files (.ex)


  1) test renders form for new sessions (TvRecipe.SessionControllerTest)
     test/controllers/session_controller_test.exs:4
     ** (UndefinedFunctionError) function TvRecipe.SessionController.init/1 is undefined (module TvRecipe.SessionController
is not available)
     stacktrace:
       TvRecipe.SessionController.init(:new)
       (tv_recipe) web/router.ex:1: anonymous fn/1 in TvRecipe.Router.match_route/4
       (tv_recipe) lib/phoenix/router.ex:261: TvRecipe.Router.dispatch/2
       (tv_recipe) web/router.ex:1: TvRecipe.Router.do_call/2
       (tv_recipe) lib/tv_recipe/endpoint.ex:1: TvRecipe.Endpoint.phoenix_pipeline/1
       (tv_recipe) lib/tv_recipe/endpoint.ex:1: TvRecipe.Endpoint.call/2
       (phoenix) lib/phoenix/test/conn_test.ex:224: Phoenix.ConnTest.dispatch/5
       test/controllers/session_controller_test.exs:5: (test)



Finished in 0.08 seconds
1 test, 1 failure
```

`SessionController` 未定义。

## 创建 `SessionController` 模块

在 `web/controllers` 目录下新建一个 `session_controller.ex` 文件，内容如下：

```elixir
defmodule TvRecipe.SessionController do
  use TvRecipe.Web, :controller

  def new(conn, _params) do
    render conn, "new.html"
  end
end
```
你可能在想，`_params` 是什么意思。在 Elixir 下，如果一个参数没被用到，编译时就会有提示，我们给这个未用到的参数加个 `_` 前缀，就能消除编译时的提示。

现在运行测试：

```bash
mix test test/controllers/session_controller_test.exs
Compiling 1 file (.ex)
Generated tv_recipe app


  1) test renders form for new sessions (TvRecipe.SessionControllerTest)
     test/controllers/session_controller_test.exs:4
     ** (UndefinedFunctionError) function TvRecipe.SessionView.render/2 is undefined (module TvRecipe.SessionView is not ava
ilable)
     stacktrace:
       TvRecipe.SessionView.render("new.html", %{conn: %Plug.Conn{adapter: {Plug.Adapters.Test.Conn, :...}, assigns: %{layou
t: {TvRecipe.LayoutView, "app.html"}}, before_send: [#Function<0.101282891/1 in Plug.CSRFProtection.call/2>, #Function<4.111
648917/1 in Phoenix.Controller.fetch_flash/2>, #Function<0.61377594/1 in Plug.Session.before_send/2>, #Function<1.115972179/
1 in Plug.Logger.call/2>], body_params: %{}, cookies: %{}, halted: false, host: "www.example.com", method: "GET", owner: #PI
D<0.302.0>, params: %{}, path_info: ["sessions", "new"], path_params: %{}, peer: {{127, 0, 0, 1}, 111317}, port: 80, private
: %{TvRecipe.Router => {[], %{}}, :phoenix_action => :new, :phoenix_controller => TvRecipe.SessionController, :phoenix_endpo
int => TvRecipe.Endpoint, :phoenix_flash => %{}, :phoenix_format => "html", :phoenix_layout => {TvRecipe.LayoutView, :app},
:phoenix_pipelines => [:browser], :phoenix_recycled => true, :phoenix_route => #Function<12.75217690/1 in TvRecipe.Router.ma
tch_route/4>, :phoenix_router => TvRecipe.Router, :phoenix_template => "new.html", :phoenix_view => TvRecipe.SessionView, :p
lug_session => %{}, :plug_session_fetch => :done, :plug_skip_csrf_protection => true}, query_params: %{}, query_string: "",
remote_ip: {127, 0, 0, 1}, req_cookies: %{}, req_headers: [], request_path: "/sessions/new", resp_body: nil, resp_cookies: %
{}, resp_headers: [{"cache-control", "max-age=0, private, must-revalidate"}, {"x-request-id", "eedn739jkdct1hr8r3nod6nst95b2
qvu"}, {"x-frame-options", "SAMEORIGIN"}, {"x-xss-protection", "1; mode=block"}, {"x-content-type-options", "nosniff"}], sch
eme: :http, script_name: [], secret_key_base: "XfacEiZ/QVO87L4qirM0thXcedgcx5zYhLPAsmVPnL8AVu6qB/Et84yvJ6712aSn", state: :un
set, status: nil}, view_module: TvRecipe.SessionView, view_template: "new.html"})
       (tv_recipe) web/templates/layout/app.html.eex:29: TvRecipe.LayoutView."app.html"/1
       (phoenix) lib/phoenix/view.ex:335: Phoenix.View.render_to_iodata/3
       (phoenix) lib/phoenix/controller.ex:642: Phoenix.Controller.do_render/4
       (tv_recipe) web/controllers/session_controller.ex:1: TvRecipe.SessionController.action/2
       (tv_recipe) web/controllers/session_controller.ex:1: TvRecipe.SessionController.phoenix_controller_pipeline/2
       (tv_recipe) lib/tv_recipe/endpoint.ex:1: TvRecipe.Endpoint.instrument/4
       (tv_recipe) lib/phoenix/router.ex:261: TvRecipe.Router.dispatch/2
       (tv_recipe) web/router.ex:1: TvRecipe.Router.do_call/2
       (tv_recipe) lib/tv_recipe/endpoint.ex:1: TvRecipe.Endpoint.phoenix_pipeline/1
       (tv_recipe) lib/tv_recipe/endpoint.ex:1: TvRecipe.Endpoint.call/2
       (phoenix) lib/phoenix/test/conn_test.ex:224: Phoenix.ConnTest.dispatch/5
       test/controllers/session_controller_test.exs:5: (test)



Finished in 0.1 seconds
1 test, 1 failure
```
测试失败，因为 `TvRecipe.SessionView` 未定义。

## 创建 `SessionView` 模块

在 `web/views` 目录下新建一个 `session_view.ex` 文件，内容如下：

```elixir
defmodule TvRecipe.SessionView do
  use TvRecipe.Web, :view
end
```
在 Phoenix 下，View 与 templates 是分开的，其中 View 是模块（module），而 templates 在编译后，会变成 View 模块中的函数。这也是为什么我们在定义模板之前，要先定义视图的原因。

此时运行测试：

```bash
mix test test/controllers/session_controller_test.exs
Compiling 1 file (.ex)
Generated tv_recipe app


  1) test renders form for new sessions (TvRecipe.SessionControllerTest)
     test/controllers/session_controller_test.exs:4
     ** (Phoenix.Template.UndefinedError) Could not render "new.html" for TvRecipe.SessionView, please define a matching cla
use for render/2 or define a template at "web/templates/session". No templates were compiled for this module.
     Assigns:

     %{conn: %Plug.Conn{adapter: {Plug.Adapters.Test.Conn, :...}, assigns: %{layout: {TvRecipe.LayoutView, "app.html"}}, bef
ore_send: [#Function<0.101282891/1 in Plug.CSRFProtection.call/2>, #Function<4.111648917/1 in Phoenix.Controller.fetch_flash
/2>, #Function<0.61377594/1 in Plug.Session.before_send/2>, #Function<1.115972179/1 in Plug.Logger.call/2>], body_params: %{
}, cookies: %{}, halted: false, host: "www.example.com", method: "GET", owner: #PID<0.300.0>, params: %{}, path_info: ["sess
ions", "new"], path_params: %{}, peer: {{127, 0, 0, 1}, 111317}, port: 80, private: %{TvRecipe.Router => {[], %{}}, :phoenix
_action => :new, :phoenix_controller => TvRecipe.SessionController, :phoenix_endpoint => TvRecipe.Endpoint, :phoenix_flash =
> %{}, :phoenix_format => "html", :phoenix_layout => {TvRecipe.LayoutView, :app}, :phoenix_pipelines => [:browser], :phoenix
_recycled => true, :phoenix_route => #Function<12.75217690/1 in TvRecipe.Router.match_route/4>, :phoenix_router => TvRecipe.
Router, :phoenix_template => "new.html", :phoenix_view => TvRecipe.SessionView, :plug_session => %{}, :plug_session_fetch =>
 :done, :plug_skip_csrf_protection => true}, query_params: %{}, query_string: "", remote_ip: {127, 0, 0, 1}, req_cookies: %{
}, req_headers: [], request_path: "/sessions/new", resp_body: nil, resp_cookies: %{}, resp_headers: [{"cache-control", "max-
age=0, private, must-revalidate"}, {"x-request-id", "vi7asqkbb9153m6ku8btf8r50p38rsqn"}, {"x-frame-options", "SAMEORIGIN"},
{"x-xss-protection", "1; mode=block"}, {"x-content-type-options", "nosniff"}], scheme: :http, script_name: [], secret_key_ba
se: "XfacEiZ/QVO87L4qirM0thXcedgcx5zYhLPAsmVPnL8AVu6qB/Et84yvJ6712aSn", state: :unset, status: nil}, template_not_found: TvR
ecipe.SessionView, view_module: TvRecipe.SessionView, view_template: "new.html"}

     stacktrace:
       (phoenix) lib/phoenix/template.ex:364: Phoenix.Template.raise_template_not_found/3
       (tv_recipe) web/templates/layout/app.html.eex:29: TvRecipe.LayoutView."app.html"/1
       (phoenix) lib/phoenix/view.ex:335: Phoenix.View.render_to_iodata/3
       (phoenix) lib/phoenix/controller.ex:642: Phoenix.Controller.do_render/4
       (tv_recipe) web/controllers/session_controller.ex:1: TvRecipe.SessionController.action/2
       (tv_recipe) web/controllers/session_controller.ex:1: TvRecipe.SessionController.phoenix_controller_pipeline/2
       (tv_recipe) lib/tv_recipe/endpoint.ex:1: TvRecipe.Endpoint.instrument/4
       (tv_recipe) lib/phoenix/router.ex:261: TvRecipe.Router.dispatch/2
       (tv_recipe) web/router.ex:1: TvRecipe.Router.do_call/2
       (tv_recipe) lib/tv_recipe/endpoint.ex:1: TvRecipe.Endpoint.phoenix_pipeline/1
       (tv_recipe) lib/tv_recipe/endpoint.ex:1: TvRecipe.Endpoint.call/2
       (phoenix) lib/phoenix/test/conn_test.ex:224: Phoenix.ConnTest.dispatch/5
       test/controllers/session_controller_test.exs:5: (test)



Finished in 0.1 seconds
1 test, 1 failure
```
测试失败，因为 new.html 模板不存在。

## 创建 `new.html.eex` 模板文件

在 `web/templates/session` 目录中新建一个空白 `new.html.eex` 模板文件。

现在运行测试：

```bash
mix test test/controllers/session_controller_test.exs
Compiling 1 file (.ex)


  1) test renders form for new sessions (TvRecipe.SessionControllerTest)
     test/controllers/session_controller_test.exs:4
     Assertion with =~ failed
     code:  html_response(conn, 200) =~ "登录"
     left:  "<!DOCTYPE html>\n<html lang=\"en\">\n  <head>\n    <meta charset=\"utf-8\">\n    <meta http-equiv=\"X-UA-Compat
ible\" content=\"IE=edge\">\n    <meta name=\"viewport\" content=\"width=device-width, initial-scale=1\">\n    <meta name=\"
description\" content=\"\">\n    <meta name=\"author\" content=\"\">\n\n    <title>Hello TvRecipe!</title>\n    <link rel=\"
stylesheet\" href=\"/css/app.css\">\n  </head>\n\n  <body>\n    <div class=\"container\">\n      <header class=\"header\">\n
        <nav role=\"navigation\">\n          <ul class=\"nav nav-pills pull-right\">\n            <li><a href=\"http://www.p
hoenixframework.org/docs\">Get Started</a></li>\n          </ul>\n        </nav>\n        <span class=\"logo\"></span>\n
  </header>\n\n      <p class=\"alert alert-info\" role=\"alert\"></p>\n      <p class=\"alert alert-danger\" role=\"alert\"
></p>\n\n      <main role=\"main\">\n      </main>\n\n    </div> <!-- /container -->\n    <script src=\"/js/app.js\"></scrip
t>\n  </body>\n</html>\n"
     right: "登录"
     stacktrace:
       test/controllers/session_controller_test.exs:7: (test)



Finished in 0.1 seconds
1 test, 1 failure
```
这是因为我们的页面还是空白的，并没有“登录”的字眼。

那么，`new.html.eex` 文件内容要怎么写？

首先，当然是加个“登录”的标题，保证我们此前的测试正确：

```eex
<h2>登录</h2>
```

接着是登录表单。首先想到的，自然是参照 `web/templates/user` 目录下的 `form.eex.html` 文件：

```eex
<%= form_for @changeset, @action, fn f -> %>
  <%= if @changeset.action do %>
    <div class="alert alert-danger">
      <p>Oops, something went wrong! Please check the errors below.</p>
    </div>
  <% end %>

  <div class="form-group <%= if f.errors[:email], do: "has-error" %>">
    <%= label f, :email, class: "control-label" %>
    <%= text_input f, :email, class: "form-control" %>
    <%= error_tag f, :email %>
  </div>

  <div class="form-group <%= if f.errors[:password], do: "has-error" %>">
    <%= label f, :password, class: "control-label" %>
    <%= password_input f, :password, class: "form-control" %>
    <%= error_tag f, :password %>
  </div>

  <div class="form-group">
    <%= submit "Submit", class: "btn btn-primary" %>
  </div>
<% end %>
```
但测试结果告诉我们：

```bash
mix test test/controllers/session_controller_test.exs
Compiling 1 file (.ex)


  1) test renders form for new sessions (TvRecipe.SessionControllerTest)
     test/controllers/session_controller_test.exs:4
     ** (ArgumentError) assign @changeset not available in eex template.

     Please make sure all proper assigns have been set. If this
     is a child template, ensure assigns are given explicitly by
     the parent template as they are not automatically forwarded.

     Available assigns: [:conn, :view_module, :view_template]
```

报错是必然的，我们前面草草写就的 `new` 函数里，只是一行 `render "new.html"`，并没有传递 `changeset` - 因为我们根本没有 `changeset` 可以传递。

怎么办？来看看 `Phoenix.HTML.Form` 的[文档](https://hexdocs.pm/phoenix_html/Phoenix.HTML.Form.html)描述的 `form_for` 的三种应用场景：

1. with changeset data - when information to populate the form comes from a changeset
2. with connection data - when a form is created based on the information in the connection (aka Plug.Conn)
3. without form data - when the functions are used directly, outside of a form

我们没有 `changeset`，但是涉及表单数据，适用第二种。

根据 `form_for` 的[用法](https://hexdocs.pm/phoenix_html/Phoenix.HTML.Form.html#form_for/4)，我们将 `new.html.eex` 做以下修改：

```elixir
diff --git a/web/templates/session/new.html.eex b/web/templates/session/new.html.eex
index 9c1f842..1df67cc 100644
--- a/web/templates/session/new.html.eex
+++ b/web/templates/session/new.html.eex
@@ -1,11 +1,5 @@
 <h2>登录</h2>
-<%= form_for @changeset, @action, fn f -> %>
-  <%= if @changeset.action do %>
-    <div class="alert alert-danger">
-      <p>Oops, something went wrong! Please check the errors below.</p>
-    </div>
-  <% end %>
-
+<%= form_for @conn, session_path(@conn, :create), [as: :session], fn f -> %>
   <div class="form-group <%= if f.errors[:email], do: "has-error" %>">
     <%= label f, :email, class: "control-label" %>
     <%= text_input f, :email, class: "form-control" %>
```
`session_path(@conn, :create)` 是表单数据要提交的路径，`as: :session` 则表示表单数据提交时，是保存在 `session` 的键名下的。

现在运行测试：

```bash
mix test test/controllers/session_controller_test.exs
Compiling 10 files (.ex)


  1) test renders form for new sessions (TvRecipe.SessionControllerTest)
     test/controllers/session_controller_test.exs:4
     ** (ArgumentError) No helper clause for TvRecipe.Router.Helpers.session_path/2 defined for action :create.
     The following session_path actions are defined under your router:

       * :new
     stacktrace:
       (phoenix) lib/phoenix/router/helpers.ex:269: Phoenix.Router.Helpers.raise_route_error/5
       (tv_recipe) web/templates/session/new.html.eex:2: TvRecipe.SessionView."new.html"/1
       (tv_recipe) web/templates/layout/app.html.eex:29: TvRecipe.LayoutView."app.html"/1
       (phoenix) lib/phoenix/view.ex:335: Phoenix.View.render_to_iodata/3
       (phoenix) lib/phoenix/controller.ex:642: Phoenix.Controller.do_render/4
       (tv_recipe) web/controllers/session_controller.ex:1: TvRecipe.SessionController.action/2
       (tv_recipe) web/controllers/session_controller.ex:1: TvRecipe.SessionController.phoenix_controller_pipeline/2
       (tv_recipe) lib/tv_recipe/endpoint.ex:1: TvRecipe.Endpoint.instrument/4
       (tv_recipe) lib/phoenix/router.ex:261: TvRecipe.Router.dispatch/2
       (tv_recipe) web/router.ex:1: TvRecipe.Router.do_call/2
       (tv_recipe) lib/tv_recipe/endpoint.ex:1: TvRecipe.Endpoint.phoenix_pipeline/1
       (tv_recipe) lib/tv_recipe/endpoint.ex:1: TvRecipe.Endpoint.call/2
       (phoenix) lib/phoenix/test/conn_test.ex:224: Phoenix.ConnTest.dispatch/5
       test/controllers/session_controller_test.exs:5: (test)



Finished in 0.07 seconds
1 test, 1 failure
```
测试结果提示我们：No helper clause for TvRecipe.Router.Helpers.session_path/2 defined for action :create.。

我们需要在 `router.ex` 文件添加一个路由：

```elixir
diff --git a/web/router.ex b/web/router.ex
index aac327c..e0406d2 100644
--- a/web/router.ex
+++ b/web/router.ex
@@ -19,6 +19,7 @@ defmodule TvRecipe.Router do
     get "/", PageController, :index
     resources "/users", UserController
     get "/sessions/new", SessionController, :new
+    post "/sessions/new", SessionController, :create
   end
```
很好，我们的测试终于通过了。

但我们才迈出了一小步。

## create 动作

如果我们此时在浏览器里访问 `/sessions/new` 页面，并提交用户登录数据，会怎样？不不不，不要在浏览器里尝试，我们用测试代码：

```elixir
diff --git a/test/controllers/session_controller_test.exs b/test/controllers/session_controller_test.exs
index 0372448..6835e40 100644
--- a/test/controllers/session_controller_test.exs
+++ b/test/controllers/session_controller_test.exs
@@ -1,9 +1,24 @@
 defmodule TvRecipe.SessionControllerTest do
   use TvRecipe.ConnCase

+  alias TvRecipe.{Repo, User}
+  @valid_user_attrs %{email: "chenxsan@gmail.com", username: "chenxsan", password: String.duplicate("a", 6)}
+
   test "renders form for new sessions", %{conn: conn} do
     conn = get conn, session_path(conn, :new)
     # 200 响应，页面上带有“登录”
     assert html_response(conn, 200) =~ "登录"
   end
+
+  test "login user and redirect to home page when data is valid", %{conn: conn} do
+    user_changeset = User.changeset(%User{}, @valid_user_attrs)
+    # 插入新用户
+    Repo.insert! user_changeset
+    # 用户登录
+    conn = post conn, session_path(conn, :create), session: @valid_user_attrs
+    # 显示“欢迎你”的消息
+    assert get_flash(conn, :info) == "欢迎你"
+    # 重定向到主页
+    assert redirected_to(conn) == page_path(conn, :index)
+  end
 end
```
我们的测试结果是：

```bash
$ mix test test/controllers/session_controller_test.exs
Compiling 1 file (.ex)
warning: variable "user" is unused
  test/controllers/session_controller_test.exs:16

.

  1) test login user and redirect to home page when data is valid (TvRecipe.SessionControllerTest)
     test/controllers/session_controller_test.exs:13
     ** (UndefinedFunctionError) function TvRecipe.SessionController.create/2 is undefined or private
     ```
`TvRecipe.SessionController.create` 未定义。

打开 `session_controller.ex` 文件，添加 `create` 动作：

```elixir
diff --git a/web/controllers/session_controller.ex b/web/controllers/session_controller.ex
index 66a5304..40ad02f 100644
--- a/web/controllers/session_controller.ex
+++ b/web/controllers/session_controller.ex
@@ -1,7 +1,20 @@
 defmodule TvRecipe.SessionController do
   use TvRecipe.Web, :controller
+  alias TvRecipe.{Repo, User}

   def new(conn, _params) do
     render conn, "new.html"
   end
+
+  def create(conn, %{"session" => %{"email" => email, "password" => password}}) do
+    # 根据邮箱地址从数据库中查找用户
+    user = Repo.get_by(User, email: email)
+    cond do
+      # 用户存在，且密码正确
+      user && Comeonin.Bcrypt.checkpw(password, user.password_hash) ->
+        conn
+        |> put_flash(:info, "欢迎你")
+        |> redirect(to: page_path(conn, :index))
+    end
+  end
 end
```

还记得模式匹配吗？我们上面的代码用它来抽取 `session` 键下的数据，然后跟数据库中存储的用户 `password_hash` 做比较，如果通过 `Comeonin.Bcrypt.checkpw` 的检查，我们就显示“欢迎你”，并重定向用户到主页。

此外，上面的代码中有两个新知识需要提一下：

1. `alias TvRecipe.{Repo, User}` - [alias](http://elixir-lang.org/getting-started/alias-require-and-import.html#alias) 允许我们给模块设置别名，这样可以减少后期输入，不必写完整的 `TvRecipe.Repo` 与 `TvRecipe.User`。
2. `cond do` - 条件判断语句。

现在运行测试：

```bash
$ mix test test/controllers/session_controller_test.exs
..

Finished in 0.2 seconds
2 tests, 0 failures
```
通过了。

但我们只处理了用户邮箱存在且密码正确的情况。还有两种情况未处理：

1. 邮箱存在，密码不正确
2. 邮箱不存在

同样的，我们先写测试：

```elixir
diff --git a/test/controllers/session_controller_test.exs b/test/controllers/session_controller_test.exs
index cc35f0a..dd5bc02 100644
--- a/test/controllers/session_controller_test.exs
+++ b/test/controllers/session_controller_test.exs
@@ -21,4 +21,24 @@ defmodule TvRecipe.SessionControllerTest do
     # 重定向到主页
     assert redirected_to(conn) == page_path(conn, :index)
   end
+
+  test "redirect to session new when email exists but with wrong password", %{conn: conn} do
+    user_changeset = User.changeset(%User{}, @valid_user_attrs)
+    # 插入新用户
+    Repo.insert! user_changeset
+    # 用户登录
+    conn = post conn, session_path(conn, :create), session: %{@valid_user_attrs | password: ""}
+    # 显示“用户名或密码错误”
+    assert get_flash(conn, :error) == "用户名或密码错误"
+    # 返回登录页
+    assert html_response(conn, 200) =~ "登录"
+  end
+
+  test "redirect to session new when nobody login", %{conn: conn} do
+    conn = post conn, session_path(conn, :create), session: @valid_user_attrs
+    # 显示“用户名或密码错误”
+    assert get_flash(conn, :error) == "用户名或密码错误"
+    # 返回登录页
+    assert html_response(conn, 200) =~ "登录"
+  end
 end
```
然后实现代码：

```elixir
diff --git a/web/controllers/session_controller.ex b/web/controllers/session_controller.ex
index 40ad02f..400a33c 100644
--- a/web/controllers/session_controller.ex
+++ b/web/controllers/session_controller.ex
@@ -15,6 +15,18 @@ defmodule TvRecipe.SessionController do
         conn
         |> put_flash(:info, "欢迎你")
         |> redirect(to: page_path(conn, :index))
+      # 用户存在，但密码错误
+      user ->
+        conn
+        |> put_flash(:error, "用户名或密码错误")
+        |> render("new.html")
+      # 其它
+      true ->
+        # 预防暴力破解
+        Comeonin.Bcrypt.dummy_checkpw()
+        conn
+        |> put_flash(:error, "用户名或密码错误")
+        |> render("new.html")
     end
   end
 end
```
再次测试：

```bash
mix test test/controllers/session_controller_test.exs
....

Finished in 0.2 seconds
4 tests, 0 failures
```
悉数通过。

到现在为止，我们还没有打开浏览器测试过页面，现在你可以试试。在浏览器上，我们会更容易发现一些可用性上的问题。

比如这个问题：登录账号后，刷新页面，我们就不知道自己是否登录了，因为页面上没有任何标识表明我们当前是登录的状态。

我们需要在页面上显示登录后的用户名。

## 登录后的页面显示 `username`

我们来改造下我们的测试代码：

```elixir
diff --git a/test/controllers/session_controller_test.exs b/test/controllers/session_controller_test.exs
index dd5bc02..52e8801 100644
--- a/test/controllers/session_controller_test.exs
+++ b/test/controllers/session_controller_test.exs
@@ -13,13 +13,19 @@ defmodule TvRecipe.SessionControllerTest do
   test "login user and redirect to home page when data is valid", %{conn: conn} do
     user_changeset = User.changeset(%User{}, @valid_user_attrs)
     # 插入新用户
-    Repo.insert! user_changeset
+    user = Repo.insert! user_changeset
     # 用户登录
     conn = post conn, session_path(conn, :create), session: @valid_user_attrs
     # 显示“欢迎你”的消息
     assert get_flash(conn, :info) == "欢迎你"
     # 重定向到主页
     assert redirected_to(conn) == page_path(conn, :index)
+    # 读取首页，页面上包含已登录用户的用户名
+    conn = get conn, page_path(conn, :index)
+    assert html_response(conn, 200) =~ Map.get(@valid_user_attrs, :username)
+    # 读取用户页，页面上包含已登录用户的用户名
+    conn = get conn, user_path(conn, :show, user)
+    assert html_response(conn, 200) =~ Map.get(@valid_user_attrs, :username)
   end
```
我们在测试中确保新建的用户登录后，页面上包含用户名。

这里，我们有两个问题需要解决：

1. 我们要显示的用户名要写在哪个模板文件
2. 模板文件中的用户名从何而来

Phoenix 创建时生成的页面里，均有 **Get Started** 菜单，这正是我们的用户名想要达到的效果。

查找一下 Get Started，我们就能定位到它在 `web/templates/layout/app.html.eex` 文件中，我们的 `username` 将加在 `app.html.eex` 文件中：

```eex
diff --git a/web/templates/layout/app.html.eex b/web/templates/layout/app.html.eex
index 82259d8..2d39904 100644
--- a/web/templates/layout/app.html.eex
+++ b/web/templates/layout/app.html.eex
@@ -17,6 +17,9 @@
         <nav role="navigation">
           <ul class="nav nav-pills pull-right">
             <li><a href="http://www.phoenixframework.org/docs">Get Started</a></li>
+            <%= if @current_user do %>
+              <li><%= link @current_user.username, to: user_path(@conn, :show, @current_user) %></li>
+            <% end %>
           </ul>
         </nav>
         <span class="logo"></span>
```

那么，用户名从哪来。

我们之前说过，一个请求到响应结束的过程是这样：

1. 路由
2. 控制器
3. 视图
4. 模板

如果用代码表示，则是这样：

```elixir
conn
|> router
|> controller
|> view
|> template
```

所以我们只要在模板的上游环节存储用户数据即可。但要如何保证，每个路由中都存储了 `:current_user` 数据？

我们来看看上游的 `router.ex` 文件，其中的部分内容如下：

```elixir
  scope "/", TvRecipe do
    pipe_through :browser # Use the default browser stack

    get "/", PageController, :index
    resources "/users", UserController
    get "/sessions/new", SessionController, :new
    post "/sessions/new", SessionController, :create
  end
```
定义在 `scope "/"` 中的所有路由，在进入控制器之前，要经过叫 `:browser` 的 pipeline：

```elixir
  pipeline :browser do
    plug :accepts, ["html"]
    plug :fetch_session
    plug :fetch_flash
    plug :protect_from_forgery
    plug :put_secure_browser_headers
  end
```
很明显，`pipeline` 就只是一些 `plug` 的组合。

解决办法已经浮出水面了：我们在 `plug` 中准备 `:current_user`，然后把这个 `plug` 放到 `:browser` 这个 `pipeline` 里，这样，我们就在模板渲染前准备好了 `:current_user`。

`plug` 有两种，一种是函数式的（function plug），一种是模块式的（module plug），我们这里将使用模块式的 plug。

模块式 plug 需要定义两个函数：

1. `init/1` - 用于初始化参数或选项，然后传递给 `call`
2. `call/2` - 第一个参数为 Plug.Conn 结构体，第二个参数为 `init/1` 的结果

请注意函数后的 `/1` 与 `/2`，它们表示函数接收的参数的数量。

我们在 `web/controllers` 目录下新增一个 `auth.ex` 文件：

```elixir
diff --git a/web/controllers/auth.ex b/web/controllers/auth.ex
new file mode 100644
index 0000000..84b17f7
--- /dev/null
+++ b/web/controllers/auth.ex
@@ -0,0 +1,16 @@
+defmodule TvRecipe.Auth do
+  import Plug.Conn
+
+  @doc """
+  初始化选项
+
+  """
+  def init(opts) do
+    Keyword.fetch!(opts, :repo)
+  end
+
+  def call(conn, repo) do
+    assign(conn, :current_user, user)
+  end
+
+end
```
当然，我们的代码在编译时报错了，因为 `user` 还没有定义，那么 `user` 要从哪儿来？

我们回到 `session_controller.ex` 文件，其中有一段：

```elixir
  def create(conn, %{"session" => %{"email" => email, "password" => password}}) do
    # 根据邮箱地址从数据库中查找用户
    user = Repo.get_by(User, email: email)
    cond do
      # 用户存在，且密码正确
      user && Comeonin.Bcrypt.checkpw(password, user.password_hash) ->
        conn
        |> put_flash(:info, "欢迎你")
        |> redirect(to: page_path(conn, :index))
```

用户登录时，我们根据他们提供的邮箱取得数据库中的用户，然后比对密码，如果密码正确，我们就得到了 `user`。

一处有 `user`，一处需要 `user`，怎么传递？

我们可以通过[会话（session）](http://www.phoenixframework.org/docs/sessions)。用户第一次访问网站时，服务端会分配一个唯一的 session id，这样每次请求进来，服务端解析 session id 就能知道是谁。听起来很复杂？不必担心，因为 Phoenix 已经帮我们打理好。我们只要关心 session 的存储、读取等就好。

让我们在用户登录时，把用户的 id 存储在 session 中：

```elixir
diff --git a/web/controllers/session_controller.ex b/web/controllers/session_controller.ex
index 400a33c..b5218f2 100644
--- a/web/controllers/session_controller.ex
+++ b/web/controllers/session_controller.ex
@@ -13,6 +13,7 @@ defmodule TvRecipe.SessionController do
       # 用户存在，且密码正确
       user && Comeonin.Bcrypt.checkpw(password, user.password_hash) ->
         conn
+        |> put_session(:user_id, user.id)
         |> put_flash(:info, "欢迎你")
         |> redirect(to: page_path(conn, :index))
```
然后我们就能在 `auth.ex` 文件中读取 session 中的 `:user_id` 了：

```elixir
diff --git a/web/controllers/auth.ex b/web/controllers/auth.ex
index 84b17f7..994112d 100644
--- a/web/controllers/auth.ex
+++ b/web/controllers/auth.ex
@@ -10,6 +10,8 @@ defmodule TvRecipe.Auth do
   end

   def call(conn, repo) do
+    user_id = get_session(conn, :user_id)
+    user = user_id && repo.get(TvRecipe.User, user_id)
     assign(conn, :current_user, user)
   end
```
最后，将 `Auth` plug 加入 `:browser` pipeline 中：

```elixir
diff --git a/web/router.ex b/web/router.ex
index e0406d2..1265c86 100644
--- a/web/router.ex
+++ b/web/router.ex
@@ -7,6 +7,7 @@ defmodule TvRecipe.Router do
     plug :fetch_flash
     plug :protect_from_forgery
     plug :put_secure_browser_headers
+    plug TvRecipe.Auth, repo: TvRecipe.Repo
   end

   pipeline :api do
```
现在运行测试：

```bash
$ mix test
...................................

Finished in 0.4 seconds
35 tests, 0 failures
```
全部通过。

你可能会问，为什么在登录时，不直接保存 `user` 数据到 session 中，而是保存了 `user.id` 的数据？假如我们保存了 `user` 数据，而用户又修改了个人信息，会导致 session 中的 `user` 数据与数据库中不一致，所以我们只存了 id，然后根据 id 从数据库中读取 `user` 数据，保证了数据的有效性。

以上，我们完成用户登录的功能。[下一章，我们将自动登录注册成功的用户](02-auto-login-user.md)。