---
layout: post
title:  "Internal Toolkit"
date:   2016-10-22 14:30:00 -0500
categories: sinatra
image: internal-toolkit.png
---
Soon after starting this new gig, I saw a need to create an internal toolset for employees to not only view reports about our customers, but also to save information about customers, make new employee onboarding easier, make support tasks more efficient etc. It is sort of a customer toolkit and intranet mashed into one. For lack of a better name, it is referred to as the Toolkit.

Although initially straightforward, this [Sinatra](http://www.sinatrarb.com/) based toolkit grew into a much larger project spanning two years of consistent changes and additions. I write this post to share the solutions for some of the bigger challenges I was able to overcome. 

**Part I**

1. [Intro: Working with Sinatra](#working-with-sinatra)
2. [Google Auth](#google-auth)
3. [Multiple Databases](#multiple-databases)
   1. [Active Record](#active-record)
   2. [database.yml in Opsworks](#databaseyml-in-opsworks)
4. [Environment Variable Use For Rake Tasks in OpsWorks](#environment-variables-in-opsworks)
5. [Empty Params Middleware](#empty-params-middleware)
6. [Password Filtering in Logs](#password-filtering-in-logs)

**Part II (Coming Soon)**

1. sinatra as rack app for opsworks (jul 24 2015)
2. timeout issues
3. error handling. god? middleware? i forget. unicorn killer?
4. exception notifier working during rake
5. Whenever gem in Opsworks

#Tools Used

- Sinatra
- AWS Opsworks

#Working with Sinatra

Going into the project, Rails felt like overkill. I liked the simplicity of routes and views in Sinatra and thought it would suffice for the project. However, after building and mainting the toolkit for two years, I now understand what people mean when they say "Rails magic".

Many if not all of the obstactles that appeared have pretty straightforward Rails solutions, or don't happen at all. Not only that, Rails is much more thoroughly documented. Example: Need to filter out passwords from logs in Rails? Check out [Rails Guides](http://guides.rubyonrails.org/configuring.html#rails-general-configuration). How to do it in Sinatra? Well, more on that later.

Looking back, I prefer the challenge. I now know so much more about the moving pieces in Rails by toppling down these Sinatra obstactles than I would have starting with Rails in the first place.

#Google Auth

I built the app to be [modular](http://www.sinatrarb.com/intro.html#Sinatra::Base%20-%20Middleware,%20Libraries,%20and%20Modular%20Apps) - this allowed for compartmentalizing components in a cleaner way. That is why you will see the inheritance of Sinatra::Base in the config and routes files rather than the classic style where you do not have to wrap the routes in a class.

Out of the box, there are a handful of configurations you can make in Sinatra. This root `sinatra_config.rb` file became a place to house settings and other "in the background" type configurations like middleware or exceptions. Great place for authentication.

I used the `omniauth-google-oauth2` gem, found [here](https://github.com/zquestz/omniauth-google-oauth2). It's a wrapper for the Google OAuth API. The README gave instruction for Rails, but a Sinatra example is included in the repo as well. Here is what I did.

1) `use` the `OmniAuth::Builder` as [Rack Middleware](http://www.sinatrarb.com/intro.html#Rack%20Middleware). Cool stuff, Sinatra!

{% highlight ruby %}
#toolkit_config.rb

class Toolkit < Sinatra::Base
  use OmniAuth::Builder do
    provider :google_oauth2, ENV['GOOGLE_CLIENT_ID'], ENV['GOOGLE_CLIENT_SECRET'], {:scope => 'email'}
  end
end
{% endhighlight %}

2) Setup the route that calls back to Google OAuth for users to then login with Google. This also requires you setting up an app in the [Google Developer Console](https://console.developers.google.com/apis/library). Also add routes for a login page and have the sign-out page clear the session storage.

{% highlight ruby %}
#toolkit.rb - routes file

namespace '/auth' do
  get '/google_oauth2/callback' do
    session[:uid] = env['omniauth.auth']['uid']
    session[:email] = env['omniauth.auth']['info']['email']
    redirect to('/')
  end

  route("/sign-in", "/failure")  { get { erb :sign_in } }
  get("/sign-out") { session.clear; erb :sign_in }
end

{% endhighlight %} 

The namespace extension is great for organizing your routes into sectors. More info on `Sinatra::Namespace` extension [here](http://www.sinatrarb.com/contrib/namespace.html).

3) Create `helpers` to test for valid domain name from authenticated user and redirect if not.

{% highlight ruby %}
#toolkit_config.rb

class Toolkit < Sinatra::Base 
  helpers do
    def session_user_authenticated?
      if (!session[:uid].nil? && session[:email].include?('@domain.com'))
        return true
      elsif (!session[:uid].nil? && !session[:email].include?('@domain.com'))
        #Receive alert when the toolkit rejects someone
        small_email_alert("Toolkit: Failed Login","<h2>The following email was rejected from the site:</h2><h3>#{session[:email]}</h3>")
        return false
      end
    end

    def require_auth
      redirect to('/auth/sign-out') unless session_user_authenticated?
    end 
  end
end
{% endhighlight %}



4) Implement the `require_auth` helper in the routes except for the authorization (log-in) route itself.

{% highlight ruby %}
#toolkit.rb

namespace '/' do
  get { require_auth; erb :index }
  before do
    root_route_namespace = request.path_info.split('/').reject(&:empty?).first
    require_auth unless %w{ auth }.include?(root_route_namespace)
  end
end
{% endhighlight %}

5) Create your `/sign-in` views to give users a place to log-in.

{% highlight erb %}
<!-- views/sign_in.erb -->

<h2>Welcome to the Toolkit</h2>
<div class="row">
  <div class="col-md-12 text-center">
    <a href='/auth/google_oauth2'>
      <button type="button" class="btn btn-primary btn-lg">
        <span class="fa fa-google" aria-hidden="true"></span> &nbsp; Sign In With Google
      </button>
    </a>
  </div>
</div>
{% endhighlight %}

#Multiple Databases

The toolkit has its own database to store info about employees, notes about our customers, and other helpful data used by the team. However, it also houses data from the main product itself. The product has a Rails backend and Node front end - the backend API is built based on the needs of the features that will be shown in the front end. As a growing company, the primary focus was customer features not internal features, which is how the Toolkit was born. As a member of the support team, I wanted to fill this need of not only displaying application data in the Toolkit but also doing it outside of the typical development process (i.e. touching the core API code base as little as possible). Plus, we also were using a PHP based Support Ticketing system at the time that had a terrible UI - tapping into that ugly database was a must!

####Active Record

I wanted the ease and power of an ORM for not just the Toolkit database, but also the backend API database and our backend Data Warehouse - also known as database sharding. After some research, I found [Octopus](https://github.com/thiagopradi/octopus). Although this seemed like a good tool, I thought it was a little more complex than what the Toolkit needed. Maybe there was something I could take advantage of in Active Record itself?

{% highlight ruby %}
#app/models/_initialize.rb

require 'mysql2'
require 'sinatra/activerecord'

class ToolkitModel < ActiveRecord::Base
  self.abstract_class = true
  establish_connection :"#{ENV['RACK_ENV']}"
end

class ApiModel < ActiveRecord::Base
  self.abstract_class = true
  self.pluralize_table_names = false
  establish_connection :"#{ENV['RACK_ENV']}-api"
end

class DwModel < ActiveRecord::Base
  self.abstract_class = true
  self.pluralize_table_names = false
  establish_connection :"#{ENV['RACK_ENV']}-dw"
end

require_rel "**/*.rb"
{% endhighlight %}

To use, create models as normal in their own corresponding folders like `app/models/toolkit/`. Then, instead of inheriting from `ActiveRecord::Base`, you inherit from the corresponding class setup in `_initialize.rb`. See example below:

{% highlight ruby %}
#app/models/toolkit/user.rb

class ToolkitUser < ToolkitModel
  self.table_name = "user"
end

#app/models/api/user.rb

class ApiUser < ApiModel
  self.table_name = "user"
end

{% endhighlight %}

Active Record will assume the table name is the class that [inherited directly from ActiveRecord::Base](http://api.rubyonrails.org/classes/ActiveRecord/ModelSchema/ClassMethods.html#method-i-table_name), but you can explicitely set it with `self.table_name =`.

####database.yml in OpsWorks

For deploying the server, I was given a blank Opsworks project by our DevOps team - this was so support could manage the server ourselves to not add responsibility to Engineering. However, this came with its own complications.

When setting up a new Opsworks project, you select the Application Server Layer. By choosing the Rails layer, Opsworks then searches for the corresponding part of its default chef cookbooks to create a new server and set up any dependencies like Apache, Unicorn and Ruby. The cookbook also handles some things that on initial thought seem to make life easier or be more secure, but become problematic later on. Case in point - [connecting to a database](http://docs.aws.amazon.com/opsworks/latest/userguide/workinglayers-rails.html#workinglayers-rails-db).

Basically, if you add a database layer to your Opsworks project, Chef will automatically create the database.yml to point to that database. If you want to use a different database, you can use custom JSON in the Opsworks GUI to override it. Multiple databases not supported. Also, environment variables not supported, meaning anyone with access to Opsworks can easily see the database credentials in the custom JSON. The original `database.yml` template with this logic can be viewed [here](https://github.com/aws/opsworks-cookbooks/blob/release-chef-11.10/rails/templates/default/database.yml.erb).

After forking the aws cookbooks, I hacked away at the `database.yml` template to end up with this:

{% highlight erb %}
#opsworks-cookbooks/rails/templates/default/database.yml.erb

production:
  adapter: <%= @database[:adapter].to_s %>
  database: <%= @database[:database].to_s %>
  encoding: utf8
  host: <%= @database[:host].to_s %>
  username: <%= @database[:username].to_s %>
  password: <%= @database[:password].to_s %>
  port: <%= @database[:port].to_i %>
<% @database[:additional_databases].each do |db_name, db_value| %>
<%= db_name.to_s %>:
  adapter: <%= db_value[:adapter].to_s %>
  database: <%= db_value[:database] %>
  encoding: <%= db_value[:encoding].to_s %>
  host: <%= db_value[:host] %>
  username: <%= db_value[:username] %>
<% if db_value[:password] %>
  password: <%= db_value[:password] %>
<% end %>
<% if db_value[:pool] %>
  pool: <%= db_value[:pool].to_i %>
<% end %>
<% if db_value[:port] %>
  port: <%= db_value[:port].to_i %>
<% end %>
<% end %>
production-api:
  adapter: mysql2
  database: api-production
  encoding: utf8
  reconnect: true
  host: <%= node[:deploy]['customer_tk'][:environment_variables][:API_READ_DATABASE_HOST] %>
  username: <%= node[:deploy]['customer_tk'][:environment_variables][:API_READ_DATABASE_USERNAME] %>
  password: <%= node[:deploy]['customer_tk'][:environment_variables][:API_READ_DATABASE_PASSWORD] %>
production-dw:
  adapter: mysql2
  database: dw-production
  encoding: utf8
  reconnect: true
  host: <%= node[:deploy]['customer_tk'][:environment_variables][:DW_READ_DATABASE_HOST] %>
  username: <%= node[:deploy]['customer_tk'][:environment_variables][:DW_READ_DATABASE_USERNAME] %>
  password: <%= node[:deploy]['customer_tk'][:environment_variables][:DW_READ_DATABASE_PASSWORD] %>
production-tickets:
  adapter: mysql2
  database: tickets-production
  encoding: utf8
  reconnect: true
  host: <%= node[:deploy]['customer_tk'][:environment_variables][:TICKETS_READ_DATABASE_HOST] %>
  username: <%= node[:deploy]['customer_tk'][:environment_variables][:TICKETS_READ_DATABASE_USERNAME] %>
  password: <%= node[:deploy]['customer_tk'][:environment_variables][:TICKETS_READ_DATABASE_PASSWORD] %>
{% endhighlight %}

With this, the template will automatically create the Opsworks MySql layer production configurations, it supports adding additional databases through custom JSON if needed (like for different environments) and also keeps the production environment credentials out of the Opsworks GUI by populating those sections with environment variables.

#Environment Variables In OpsWorks

OpsWorks handles environment variables with a simple form within the GUI itself. The below picture, borrowed from [AWS docs](https://aws.amazon.com/blogs/devops/aws-opsworks-supports-application-environment-variables/), should give you a good idea. 

![opsworks env variables](https://dmhnzl5mp9mj6.cloudfront.net/application-management_awsblog/images/envar2.png)

The idea is kind of nice - I could see this being great in simpler applications where I may not even ever have to log-in to the server, I can just set it all up in OpsWorks. What I quickly found out was these variables are not stored in a place accessible by the servers `$PATH` at all. Basically, the Chef script does some magic to make the variables only accessible to a currently running Rails app (or in my case, Sinatra app).

What I found was that the cookbook will instead save these variables in the unicorn config file. I still am unsure why AWS did it this way instead of making them available in the `$PATH`, but I was able to exploit it. To give rake tasks access, all I had to do was read `unicorn.conf` before executing any of the tasks. See example below:

{% highlight ruby %}
#Rakefile

if File.exist?('../../shared/config/unicorn.conf')
  env = File.read('../../shared/config/unicorn.conf').scan(%r{\ENV.*})
  env.each {| i | eval i}
end
{% endhighlight %}

#Empty Params Middleware

One of the first forms created in the Toolkit was to aid in the new-hire onboarding process - things like triggering emails to create them an email address, order them a computer, set them up for different accounts would all kick off during form submission. My manager quickly created an HTML form which on initial testing worked, but I guess we were not thorough enough.

What I was able to find out is on form submital, any fields left empty are still passed on to the server. Sinatra does not ignore those parameters, it saves them as empty strings. This bugged me enough that I created middleware to remove any of these empty strings.

{% highlight ruby %}
#lib/ext/params_middleware.rb

class ParamsParser < Sinatra::Base
  def initialize(app)
    @app = app
  end
  
  def call(env)
    @env = env
    sanitize unless @env["QUERY_STRING"].empty?
    status, headers, body = @app.call(@env)
  end

  def sanitize
    query = Rack::Utils.parse_query(@env["QUERY_STRING"]).delete_if { |k,v| v.nil? || v.empty? }
    
    #Handles nested params ex. ?foo=bar&ba[]=23&ba[]=
    query.values.each do |i|
      i.delete_if { |a| a.nil? || a.empty? } if i.class == Array
    end
    
    env["QUERY_STRING"] = Rack::Utils.build_query(query)
  end
end
{% endhighlight %}

Check out [Sinatra Rack Middleware](http://www.sinatrarb.com/intro.html#Rack%20Middleware) for more info.

Basically, what this does is takes the raw `@env["QUERY_STRING"]`, parses it, removes any empty or nil parameters, and saves it back for Sinatra to finish parsing into @params. Just add `use ParamsParser` to the apps config file and you are good to go.

#Password Filtering in Logs

Another form being used by our deployment team allowed users to create logins for internal use. I quickly found out that error emails had the password parameters unfiltered. Although an easy fix, this was not well documented and took a lot of reserch to understand exactly where to filter out the passwords.

{% highlight ruby %}
#toolkit_config.rb

class Toolkit < Sinatra::Base
  use Rack::Config do |env|
    env ["action_dispatch.parameter_filter"] = [:password]
  end
end
{% endhighlight %}

Sinatra holds `action_dispatch` settings in Rack::Config. Once I found this, it was as easy as adding `:password` to the filter.
