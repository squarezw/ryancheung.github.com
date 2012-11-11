---
layout: post
title: "Use Presenters in Rails"
date: 2012-11-11 21:44
comments: true
categories: rails pattens decorator
---

通常在 Rails App 中我们提倡 fat model skinny controller，但当业务的复杂度不断增加的时候，
也可能会出现 fat controller 或者 fat view 的情况。例如 NICE-Program 减重方案智能生成项目中，
展示方案的 view 堆积了一堆的展示逻辑，包括如身高体重单位转换、性别文字展示、动态显示设定数量
的减肥建议等:


    <tr class="bg0">
      <td><b>Sex:</b> <%= @program.sex_type == "1" ? "Male" : "Female" %></td>
      <td><b>Birthday:</b> <%= @program.birthday %></td>
    </tr>
    <tr>
      <td><b>Height:</b> <%= cm_to_inch(@program.height) %></td>
    </tr>

    <% if @tip_set.special? %>
      <output class="bg1"><b>Special conditions</b></output>
      <p>
        <ul>
          <% @tip_set.special_tips[0..4].each do |tip| %>
            <li><%= tip.try(:html_safe) %></li>
          <% end %>
        </ul>
      </p>
    <% end %>

这些都是 view 的展示(presentation) 逻辑，这些逻辑多了以后也会变的难以维护。
为了解决这个问题，我们再抽象出一层: Presenters 。

## How it works

Presenter 模式通过创建一个 class 来表现 view 当状态, 并且把 controller 中的装备 doman model
的逻辑和 view 中的展示逻辑以 OO 的方式包含在其中。

## When to use it

在一个复杂的 app 中，复杂的 controller action 可能需要装备很多的 model 对象来生成 view，例如
在 NICE 方案展示的 action 中我们需要准备 @user_data, @nice_contract, @program 等对象
来生成 view ，然而我们可以通过 Presenter 来封装聚合 view 所有的数据的逻辑，这样可以让 controller
的职责更单一，而且能方便测试。

在 view 中也会有很多非业务相关的逻辑，如格式化，计算等。这些逻辑放在 view 中会变得难以测试。
Presenter 可以封装这些逻辑来方便测试，并且使我们的 view 变得更清晰。

## How to use it

### The Decorator Pattern

{% img /images/decorator_uml.png %}

Decorator 模式，顾名思义就是装饰对象，即向对象添加额外的功能。它的实现是通过一个对象来包裹一个
现有的对象来添加其他需要的功能，并且以代理的方式提供现有对象的功能。这个恰巧符合 Presenter 的
需求，view 中及需要 model 及业务原本的行为，又需要跟展示相关的行为，因此我们可以添加一个
Decorator 来同时封装业务逻辑和展示逻辑。

### The Refactor

我们现在可以添加一个 Decorator 来装饰我们的方案对象：

    class ProgramDecorator
      def initialize(program)
        @program = program
      end

      # extra display logic
      def sex_type_text
        @program.sex_type == "1" ? "Male" : "Female"
      end

      def height
        cm_to_inch(@program.height)
      end

      ...

      # delegates to @program
      def birthday
        @program.birthday
      end

      def bmi
        @program.birthday
      end

      ...
    end

Controller:

    class ProgramsController < ApplicationController
      def show
        @program = ProgramDecorator.new(Program.find(params[:id]))
      end
    end

View:


    <tr class="bg0">
      <td><b>Sex:</b> <%= @program.sex_type_text %></td>
      <td><b>Birthday:</b> <%= @program.birthday %></td>
    </tr>
    <tr>
      <td><b>Height:</b> <%= @program.height %></td>
    </tr>

这样看来 view 就变得短小清晰了

### The Gem

Github 上有个非常不错的 gem 叫 [draper](https://github.com/drapergem/draper)，它可以用来帮助
我们写 Decorator(及 Presenter)，它与 ActiveRecord 深度集成，帮我们封装了代理部分的 API，
在 Decorator 中可以直接使用 model 的 API，并提供了 view_helper 来方便编写展示相关逻辑。

## Wrap Up

相比直接把代码散步在 controller 和 view 中，使用 Presenter 有几点优势：

* controller 和 view 的代码短小清晰，职责分明
* 方便测试
* 使用 OO 方式代替 view helpers，便于维护和扩展
