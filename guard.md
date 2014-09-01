使用guard 和 spring 监控测试文件、自动运行rspec。 可以大大提高开发效率。  
这里主要记录guard 和spring 相关的一些问题。

###相关gem

```ruby
  gem 'spring'
  #use guard to auto run rspec
  gem 'guard', '~> 2.6.0'
  gem 'guard-rspec', '~> 4.3.1'
  gem 'terminal-notifier-guard'
  gem "spring-commands-rspec", group: :development
```

****

### 关于spring
[spring](https://github.com/rails/spring) 是Rails 的application preloader(在Rails4.1是默认include), 通过在后台运行一个spring server，每次运行命令，会通过Process.fork 进行相关处理， 避免每次加载Rails 环境， 减少了启动时间。  

**spring 和 zeus、spork的比较：**

- spring 是后台自动运行， 第一次运行spring xx(比如spring rake -T) 会在后台启动spring server进程(通过spring status 查看spring 进程)。  
   spring server 是多个terminal共享的。关闭这个terminal的时候 关闭这个spring server。   
- zeus 和spork 差不多， 需要手动运行zeus s 和spork 来启动server。
- spring是ruby实现的, 比 zeus 功能更全面些问题也少些。

基于以上原因，所以这次采用guard + spring 的方式。

**spring的相关features**

a)、spring 会在后台自动运行spring server, 每次运行会fork一个新的子进程。通常会有两种类型的spring app:

```ruby
spring server | wheel-web | started 25 secs ago
spring app    | wheel-web | started 24 secs ago | development mode
spring app    | wheel-web | started 9 secs ago | test mode
```

b)、每次运行都会重新加载application code: 

rails runner  'puts User.object_id'

```ruby
2198952740
```
重新保存一下 User 这个model。  
rails runner  'puts User.object_id'(你会发现id 已经改变)
```ruby
2172526260
```

c)、当spring 检测到configs、initializers、gem dependencies 发生改变时， 会重启application

spring status
```ruby
1104 spring server | wheel-web | started 21 secs ago
1141 spring app    | wheel-web | started 5 secs ago | development mode
```

重新保存一下config/application.rb 等配置文件  
spring status（进程id 已有1141 变成了1168）
```ruby
1104 spring server | wheel-web | started 1 min ago
1168 spring app    | wheel-web | started 2 secs ago | development mode
```

d)、默认情况下，spring是 每隔0.2 检测 文件是否改变。

**spring 自定义配置文件**

spring有两个自定义方式:
- ~/.spring.rb 在bundler 之前加载
- config/spring.rb在bundle 之后加载

**spring log**

如果你想查看spring 的log， 可以先关闭spring stop, 然后在运行以下命令即可：  
export SPRING_LOG=/tmp/spring.log

### 关于 guard

guard 可以很好的检测文件的改变， 这里我们利用主要利用这个特性 进行自动运行rspec测试。配置如下：

```ruby
guard :rspec, cmd: 'time spring rspec -f doc' do
  watch(%r{^spec/.+_spec\.rb$})
  watch('spec/rails_helper.rb')                       { "spec" }
  watch('spec/spec_helper.rb')                        { "spec" }
  
  watch(%r{^lib/(.+)\.rb$})                           { |m| "spec/lib/#{m[1]}_spec.rb" }
  
  watch(%r{^app/(.+)\.rb$})                           { |m| "spec/#{m[1]}_spec.rb" }
  watch(%r{^app/controllers/(.+)_(controller)\.rb$})  { |m| ["spec/#{m[2]}s/#{m[1]}_#{m[2]}_spec.rb"] }
  watch('app/controllers/application_controller.rb')  { "spec/controllers" }

  watch(%r{^spec/support/(.+)\.rb$})                  { "spec" }
  watch('config/routes.rb')                           { "spec/controllers" }
end
```

**Gem terminal-notifier-guard**

使用gem terminal-notifier-guard 在每次运行之后， 将运行的结果 发送给Mac Notification Center。(在Mac 还有一款通知方式Grawl不过是收费的。)
