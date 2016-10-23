# Phoenix 初探

在[上一章](01-create-project.md)，我们创建了一个 PhoenixMoment 项目，并且通过 Mix 工具启动：

```bash
$ mix phoenix.server
```
现在，打开浏览器，访问 [http://localhost:4000](http://localhost:4000) 网址，我们会看到如下截图中的内容：

![PhoenixFramework Page](img/02-phoenixframework-page-screenshot.png)

从输入网址到返回页面之间，都发生了什么？我们来简单了解一下。

1. 你在浏览器请求了 `http://localhost:4000` 网址
2. Phoenix 在服务器端接收到请求，它会查看 `phoenix_moment` 目录下的 `web/router.ex` 文件，接着就找到以下内容：

    ```elixir
    scope "/", PhoenixMoment do
      pipe_through :browser # Use the default browser stack

      get "/", PageController, :index
    end
    ```
    我们看到，`router.ex` 文件里已经设定好了规则：有用户以 `get` 方式请求网址 `/` 时，`PageController` 模块里的 `index` 动作将接手处理。
3. 查看 `web/controllers/page_controller.ex` 文件内容：

    ```elixir
    defmodule PhoenixMoment.PageController do
      use PhoenixMoment.Web, :controller

      def index(conn, _params) do
        render conn, "index.html"
      end
    end
    ```
    目前文件中只有一个 `index` 函数，其中 `render conn, "index.html"` 渲染了上面截图中我们看到的页面。

很简单是不是，你是否已经跃跃欲试？

让我们依葫芦画瓢，添加一个 `/help` 页面试试。

1. 在 `web/router.ex` 文件中添加一个路由：

    ```elixir
    scope "/", PhoenixMoment do
      pipe_through :browser # Use the default browser stack

      get "/help", HelpController, :index # <- 这是我们新增的路由
      get "/", PageController, :index
    end
    ```
2. 在 `web/controllers` 目录下新建一个 `help_controller.ex` 文件，添加如下内容：

    ```elixir
    defmodule PhoenixMoment.HelpController do
      use PhoenixMoment.Web, :controller

      def index(conn, _params) do
        render conn, "index.html"
      end
    end
    ```
3. 接着访问 `http://localhost:4000/help` 网址：

    ```html

    UndefinedFunctionError at GET /help
    function PhoenixMoment.HelpView.render/2 is undefined (module PhoenixMoment.HelpView is not available)
    ```
    报错了。错误里显示，我们还没有定义 `PhoenixMoment.HelpView` 模块。

4. 在 `web/views` 目录下新建 `help_view.ex` 文件，内容参照 `web/views/page_view.ex` 文件，如下：

    ```elixir
    defmodule PhoenixMoment.HelpView do
      use PhoenixMoment.Web, :view
    end
    ```
5. 再访问 `http://localhost:4000/help` 网址：

    ```html
    Phoenix.Template.UndefinedError at GET /help
    Could not render "index.html" for PhoenixMoment.HelpView, please define a matching clause for render/2 or define a template at "web/templates/help". No templates were compiled for this module.
    ```
    又报错了。错误消息提示我们要在 `web/templates/help` 目录下创建一个模板文件。

6. 在 `web/templates/help` 目录下新建一个 `index.html.eex` 文件，添加如下内容：

    ```html
    <p>这是帮助内容</p>
    ```
7. 再访问 `http://localhost:4000/help` 网址：

    ![帮助页面截图](img/02-help-page-screenshot.png)

    这一次，页面显示正常。

从路由，到控制器，到视图，到模板，每一个步骤，Phoenix 都会有完整的错误提示。所以哪怕我们不了解它们是什么，也是可以按部就班，成功添加新页面的。

我们来整理下这一章涉及的几个概念：

1. [路由](http://www.phoenixframework.org/docs/routing)（router）- 决定某个控制器中的某个动作来处理请求
2. [控制器](http://www.phoenixframework.org/docs/controllers)（controller）- 决定了怎么处理请求
3. [视图](http://www.phoenixframework.org/docs/views)（view）- 渲染模板
4. [模板](http://www.phoenixframework.org/docs/templates)（template）- 要展示怎样的内容及样式给用户

目前为止，我们还没有接触到模型（Model），凑上模型，就是我们经常见到的 [`MVC` 模式](https://en.wikipedia.org/wiki/Model–view–controller)。

不过在这里，我不打算论述 MVC 的好处，毕竟只是个入门教程。你目前只需要了解，Phoenix 应用了 MVC 模式，我们以后的代码里，将会大量涉及。