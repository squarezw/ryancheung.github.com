---
layout: post
title: "Ajax In Rails 3"
date: 2012-10-14 20:00
comments: true
categories: rails3 ajax
---

我们在使用 `link_to` 和 `form_for` API的时候经常会看到一个参数
 `:remote => true` ，这个参数是做什么用的呢？在阅读文档后发现
它是用来做 AJAX 异步调用的。虽然知道它是，但在使用 rails 这
么长时间一直都不知道怎么去用这个参数，直到这周的开发工作中
不得不使用 AJAX，因此我去详细了解了下怎么去使用 `:remote` 这
个参数。了解如何使用后发现 rails 中使用 remote 链接或表单
来做 AJAX 调用真的非常方便。下面我就分享下怎么去使用它。

我们可以看到，Rails 项目默认生成后 assets 目录的 application.js
 文件中会有一下两行

    //= require jquery
    //= require jquery_ujs

Rails 3 就是用这个 jquery_ujs.js 文件来驱动使用
 `:remote => true` 参数生存的异步链接和表单的。

## jquery_ujs 做了什么

1. 它会找出所有 remote 链接和表单并重写 `click` 事件来使他们
使用 AJAX 的方式提交到服务器

2. 它会触发5个 javascript 事件，通过绑定回调来处理 AJAX 返回
结果

在 Rails 2 中，实现异步链接或表单的方式是使用 `link_to_remote`
 和 `remote_form_for` ，这2个 API 可以设置 update 等参数去处理
返回结果，但在 Rails 3 中这2个 API 也被抛弃，而且返回结果只能
我们手动去处理。

## HTML 5

remote 链接和表单使用 Unobstrusive Javascript 方式来生成 HTML。
它会在生成的 `a` 和 `form` 标签中生成 `data-remote` 属性。这个
是有效的 HTML5 属性。jquery_ujs 正是判断这个属性来设置
其 AJAX 行为。 Unobstrusive JavaScript(不唐突的 javascript )，
这个东西的主要作用一个是将页面的行为和内容展现分离，另外一个更
关键的是 它可以减少 javascript 编写。也正是他方便 AJAX 的实现。
来看个例子:

    <%= link_to "修改", { action: "edit_struct", evaluation_input_id: @evaluation_input.id, type: the_type }, remote: true %>

将生成

    <a href="/admin/solutions/edit_struct?evaluation_input_id=1&amp;type=solution" data-remote="true">修改</a>


## 处理 AJAX 返回

AJAX 返回结果有两中处理方式：

1. 通过绑定上面提到的 javascript 事件的回调来更新页面的HTML

2. 通过 server 直接返回的 javascript 来更新页面

先来看第一种方式，当 jquery_ujs 将请求发送到 server 的时候，在
整个请求发送的前后它会触发5个事件。

这个5个事件是:

    ajax:before
    ajax:beforeSend
    ajax:success
    ajax:error
    ajax:complete

因此在页面中，我们可以绑定回调方法来处理返回结果，例如：

    $('#edit_struct').bind('ajax:success', function(data) {
        $('#solution_struct').html(data);
        });

第二中方式也是很简单，server 只需要返回更新 js 来让浏览器执行。
这是我们需要用到 js.erb 视图模版。

下面是 NICE Slim 中的一个例子。在方案编辑页面中需要可以在本页
编辑饮食结构：

{% img /images/nice_slim_1.png %}

点击*修改*按钮后变成：

{% img /images/nice_slim_2.png %}

在控制器中，我们需要添加 js 的返回格式

    class Admin::SolutionsController < Admin::BaseController
      def edit_struct
        render format: :js, layout: false
      end
    end

视图 edit_struct.js.erb

    $("solution_struct").html("<%= escape_javascript(render 'edit_struct') %>");

这里是添加了个 `_edit_struct.html.erb` partial 来生成编辑用的HTML

这2种方式各有优劣，第一种可以在前端可以方便处理请求完
成的各种情况， 如处理请求失败后的处理，而第2中方式前
端只能处理请求成功后的情况。当然第2中方式可以在后台使用 partial
来方便生成返回的 HTML 内容。
