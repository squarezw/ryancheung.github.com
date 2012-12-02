---
layout: post
title: "Things you should known about Rails Engine"
date: 2012-12-02 21:28
comments: true
categories: Rails-Engine Rails
---

Rails Engine 算是 Rails 框架中的一项伟大发明，由于 Rails Application 本身也是
个 Engine, 因此它让你的 App 挂载其他 Engine, 甚至在 Engine 中挂载 Engine。它
的主要目标是可以模块化大型业务系统，例如做 Authentication 的 [Devise](https://github.com/plataformatec/devise),
做论坛的 [Forem](https://github.com/radar/forem), 做电子商务的 [Spree](https://github.com/spree/spree) 等等。
Rails Engine 可以帮助将一个大型系统划分成多个小的独立的模块或业务，方便我们开发和后期维护。

因此，我们的 NICE 智能方案这边就考虑将方案的生成考虑做成了一个独立 Engine。
Rails Engine 很强大但其实也不算成熟，至少相对 Rails 来说算是。虽然大部分功能特性
跟 Rails Application 一样，但仍然会有些细节上的不同，而且这些不同就导致不能像在
 Rails 中方便的去使用一些功能。下面就从几个方面来谈谈我在使用 Engine 过程中的一些经验。

## gem

Rails Engine 中引用的 gems 不会被自动加载，你需要在 engine 使用它们之前手动把它们加载进来。
Rails 中有些 gem 之所以不需要 require 是因为这些 gem 里有 Railtie，Railtie 是 Rails 初始化
过程中很关键的部分，初始化过程找到所有包括 gems 中的 Railtie 并将其加载。但 Railtie 在 engine 中
就不起作用了，因为 engine 已经是一个 gem，在一个 gem 中使用另外个 gem 必须得先 require 它。
例如：

```ruby
require "program-service/engine"
require 'enumerize'
require 'draper'
require 'will_paginate'
require 'simple_filter'

module ProgramService
end
```

## Rspec

Engine 中使用 Rspec 可没那么直接，需要做一些手动配置。在生成 Engine 的时候得额外使用2个参数: -T --dummy-path=spec/dummy

* `-T` 跳过 test/unit
* `--dummy-path` 设置 spec 和 dummy app 目录，默认是 test/dummy

在生成的 spec_helper.rb 中 require 一些测用的的 gem，如：

    require 'capybara/rspec'
    require 'database_cleaner'

再把

    Dir[Rails.root.join("spec/support/**/*.rb")].each {|f| require f}

改成

    ENGINE_RAILS_ROOT=File.join(File.dirname(__FILE__), '../')
    Dir[File.join(ENGINE_RAILS_ROOT, "spec/support/**/*.rb")].each {|f| require f }

在 Engine 中配置 rspec 的 generators

```ruby
module ProgramService
  class Engine < ::Rails::Engine
    config.generators do |g|
      g.test_framework :rspec, :view_specs => false
    end
  end
end
```

这样配置完后在 rails g 命令后才会自动生成对于的 spec 文件。

## FactoryGirl

FactoryGirl 在 engine 中也是不能直接使用的，首先你需要在 spec_helper.rb 中添加

    require 'factory_girl'

然后再添加

```ruby
# FactoryGirl extra configuration in engine
FactoryGirl.definition_file_paths << File.join(File.dirname(__FILE__), 'factories')
FactoryGirl.find_definitions
```

来设置 factory 文件查找路径，在 engine 中他的默认的查找路径是在 spec/dummy/spec/factories 中，
所以必须的手动添加 spec/factories 路径进去。

## autoload_paths

有时候你会需要添加额外的加载路径，如 `app/services` 目录，你可以在 Engine 中配置：

```ruby
module ProgramService
  class Engine < ::Rails::Engine
    config.autoload_paths += Dir[File.expand_path("../../../app/services", __FILE__)]
    config.autoload_paths += Dir[File.expand_path("../../../app/services/**/", __FILE__)]

    ...
  end
end
```
## assets & precompiling

有时候你需要加入一些只有 Engine 中会用到的一些资源文件，如 admin 后台管理中用的 admin.js 和 admin.css，
你可以将他们加入到 precompile 中：

```ruby
module ProgramService
  class Engine < ::Rails::Engine
    initializer "program-service.assets.precompile" do |app|
      app.config.assets.precompile += %w(admin.css admin.js)
    end
  end
end
```

以上提到的都是 engine 和 rails 差异的地方，解决完所有问题后基本还是可以像 rails 中一样
正常工作的。
