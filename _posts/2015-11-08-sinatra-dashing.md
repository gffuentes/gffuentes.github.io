---
layout: post
title:  "Run Sinatra App Alongside Dashing App"
date:   2015-11-08 21:47:45 -0500
categories: sinatra
---
[Dashing](http://dashing.io/) is a great, simple, dashboard-ing engine built on [Sinatra](http://www.sinatrarb.com/) made by Shopify. After having built a Sinatra application as an internal tool for my work, I was looking forward to integrating our Dashing dashboard to run on the same Rack. This post is to review the solution I found to running Dashing alongside a Sinatra application.

Let's call the Sinatra application `boom-tools` and the Dashing app `dash` from here on out. My goal was to have my `dash` live in a folder within `boom-tools` and use the latters `Gemfile`, `config.ru`, and so on but hold its own routes, views, etc within it's own folder. So something like this:

    ★ boom-tools
      |-- dash
      |    |-- assets
      |    |-- config
      |    |-- config.rb
      |    +-- dashboards
      |    |   +-- science.erb
      |-- app
      |-- views
      |-- public
      |-- config.ru
      |-- boom_tools_app.rb
      +-- Gemfile

Then I would simply map Rack to run each application:

{% highlight ruby %}
#config.ru
require './boom_tools_app.rb'
map "/" do
  run BoomTools
end

require './dash/config.rb'
map "/dash" do
  run Dash
end
{% endhighlight %}

To achieve this, there are a couple of steps:

- split up `boom-tools` and `dash` `Sinatra::Application` run times so that Rack treats them as separate apps
- get Dashing to `rackup` from a non-root directory
- map each app in config.ru
- Bonus: change Dashing to run on same web-server (future article)

## Modular Sinatra with *Sinatra::Base*
Both Dashing and the Sinatra Application need to have unique run spaces to make the built-in Sinatra settings correct for each unique app. Unfortunately, Dashing is not a modularized Sinatra application, but inherits from `Sinatra::Application` rather than `Sinatra::Base`. There is a work around, however, by modularizing the `boom-tools` app by wrapping it in the `Sinatra::Base` class. **NOTE: Some Sinatra specific things will need to change or will need registration when in use with sinatra/base. Make sure to look in Sinatra doc for more information.** [Modular Sinatra documentation](http://www.sinatrarb.com/intro.html#Sinatra::Base%20-%20Middleware,%20Libraries,%20and%20Modular%20Apps)

{% highlight ruby %}
#boom_tools_app.rb
require 'sinatra/base'
class BoomTools < Sinatra::Base
  get '/' do
    erb :index
  end
  get '/fire-extinguisher-manual'
    erb :extinguish
  end
  post '/itsboomtime'
  end
end
{% endhighlight %}

Additionally, Dashing uses its own `config.ru` for Sprockets requirements, Rack settings and other things. I pushed this information into `dash/config.rb` to be required from the root `config.ru` and also gave the Dashing application it's own class.

{% highlight ruby %}
#dash/config.rb
require 'dashing'

class Dash < Sinatra::Application
end

configure do
  set :auth_token, 'YOUR_AUTH_TOKEN'

  helpers do
    def protected!
     # Put any authentication code you want in here.
     # This method is run before accessing any resource.
    end
  end
end

#This can no longer be done here - this is used by Rack so must be in the root config.ru
#map Sinatra::Application.assets_prefix do
#  run Sinatra::Application.sprockets
#end
{% endhighlight %}

## Dashing file-path requirement change
At this point, if you `rackup`, you will likely only be able to see the `boom-tools` routes. Going to the URL of one of your dashboard `/dash/science` may result in a blank screen. However, a quick network panel inspection will show a bunch of load failures, indicating that Dashing is not properly loading. Here is the fix, with a little help from the `require_all` gem to make requiring folders easier:

{% highlight ruby %}
#dash/config.rb
require 'dashing'
require 'sprockets'
require 'require_all'

class Dash < Sinatra::Application
  settings.root = settings.root + '/dash' #originally was "/Users/acme/boom-tools"
  settings.public_folder = settings.root + '/public'
  settings.views = settings.root + '/dashboards'
end

require_all Dash.settings.root + '/lib'
require_all Dash.settings.root + '/jobs'

configure do
  set :auth_token, 'YOUR_AUTH_TOKEN'
  sprockets_paths = Dash.sprockets.paths
  set :sprockets, Sprockets::Environment.new(root = Dash.settings.root)
  sprockets_paths.each {|i| Dash.sprockets.append_path(i.gsub('boom-tools/','boom-tools/dash/'))}

  helpers do
    def protected!
     # Put any authentication code you want in here.
     # This method is run before accessing any resource.
    end
  end
end
{% endhighlight %}

As you can see, Dashing was pointing to Rack's runtime root (`/boom-tools`) and could not find the dashboard or any of its requirements. All that is left is wrapping up `config.ru`.

## Map Apps in config.ru
{% highlight ruby %}
#config.ru
require './boom_tools_app.rb'
map "/" do
  run BoomTools
end

require './dash/config.rb'
map Dash.assets_prefix do
  run Dash.sprockets
end

map "/dash" do
  run Dash
end
{% endhighlight %}

And you should now be able to `rackup` with no issues.
