# Shepherd

Done it as an experiment with Crystal lang

1. Api centric
2. Offer familiar concepts like controllers, routing, and TODO: models
3. Offer easy websockets integration.
4. Models with query builders

- [controllers](#controllers)
- [routing](#routing)
- [websockets](#websockets)
- [models](#model)

# how to
clone
cd to it
run `crystall app.cr`


# Controllers <a name="controllers"></a>

We all know what they are. It's just plain old Crystal classes which will be instantiated, and whose method will be called when router mathces appropriate request path.

One thing though, they are supposed not to be the god objects like they often are, but rather "main" functions for each request, dispatching to other responsible classes.

Your controllers reside in app/controllers/, and to add one just add a class
following this name convention:
`
App::Controllers::YOURCONTROLLERNAME  < Shepherd::Controller::Base ; end
`

Conttrollers do have methods that:
1. have access to request and it's data
2. rendering methods.

##### Request object
you can access the `request : HTTP::Server::Request` of current request through getter `request`

##### Params
the methods to access the info including body, JSON payload, ecoded form data and etc. Controller has the property `params`, that is actually a lazily instantiated `Shepherd::Server::Request::Params`

so in controller you access as:

`params.json : JSON::Any` which will parse the body to JSON::Any

`params.route : Hash(String, String)` will return the hash, having the route params that where set in router, e.g.:

```
get "/foo/:id", to: "posts#bar"

posts#bar
  render plain: params.route["id"] #=> "some id"
end
```

`body : String?` returns the body of request

`encoded_form : HTTP::Params` if you had encoded form (anyway returns empty if cannot parse or wrong content type)

`url_query : HTTP::Params` obvious

TODO: `params[](val) : HTTP::Params | JSON::Any | Nil | String` generic accessor (will evaluate everything till it finds , or returns nil), not encouraged.

##### Q: you may ask why seaccess params separately, why no just params[:val]?
##### A:
that is for performance reasons, to not eavaluate and parse what you don't need.
e.g. you have class with schema JSON, so you want to be able to parse it with it like `json = SomeClass.parse(params.body)`.
keep in my mind that all params are lazily evaluated so not accessing them, gives no performance penalty. E.g. `paramas.route["id"]` will not parse `json` nor `url_query` and vice versa.

## rendering methods
`render(plain value : String) : Nil` will set content type to text/plain and print value to response

`render(json value ) : Nil` will set content type to application/json and render value.to_json

`head(status_code : Int32)` will set status_code on response

TODO: more will be added including views and etc. It's just 0.1 yet :)

### when rendering is happening
App prints to response only if you call the appropriate method, if you do not call any render method, or not change the status code. App will just head 200.
Controllers do not keep track if you render twice, so keep it in mind.

### functional controllers:
to define one in controller run `has_functional_actions` macro
functional controllers can access the params via `params(context : HTTP::Context) : Shepherd::Server::Request::Params`, and rendering via: `render(context, args) : Nil` (with overloads)
example usage:

```ruby
App::Routes::Map
    def draw
        scope "/api" do
          get "/foo", to: "users.index" #delimit method with dot so it won't be instantiated
        end
    end
end

class App::Controller::Users < Shepherd::Controller::Base

  has_functional_actions #macro that gives some special static methods

  def self.index(context)
    value_to_render = params(context.request).url_query["foo"]
    render context.response, plain: value_to_render
  end
end
```
# HTTP routing <a name="routing"></a>
You routes are defined in app/routes/map, in App::Routes::Map #draw method's body.
All possibilites are in this snippet.
on incoming request, if request.path matches the registered route, appropriate controller with appropriate method will be called.

```ruby
class App::Routes::Map
        #define routes here
    def draw
        #on request GET req with "/" will instantiate App::Controller::Home with context
        #and call its #index method
        #behind the scenes will just App::Controller::Home.new(context).index
        get "/" to: "home#index"

        #on PUT request to "/aaa" will call App::Controller::Home without instantiating
        #calling .show on it with passing it context
        #behind the scenes App::Controller::Home.show(context)
        put "/aaa/:id" to: "home.show"

        #ads wildcards route like Rails
        get "/*", to: "home#index"

        #adds parametrized routes like Rails
        get "/users/:id", to: "users#show" # later access in controller params.route["id"]

        #adds scoped routes like Rails
        scope "/api" do

            # path is /api/bar
            post "/bar", to: "posts.joe"

            #scopes can be nested
            scope "/foo/v1" do
                #path is /api/foo/v1/baz
                delete "/baz", to: "posts#delete"
                #resource routes will respect scoped paths
                resources "posts", with: ["#index", ".show", ".delete"]
            end

        end

        # will create REST methods to appropriate actions (all instance) the same as Rails
        resources "users"

        #will only create routes to those actions,
        #as well this way you can specify wheter instance or functional controller
        #action shall be called by adding (dot .) or (#) between name and method
        resources "posts", with: ["#index", ".show"]

    end

end
```

# Websockets <a name="websockets"></a>

well socket handling is a bit tricky,

##### Concept is:
app can have separate distinct websocket connection entries, e.g. one is general - which is responsible for json exchange, and others for any other staff (some protected connection, or even maybe you will write some voice transmition protocol and etc).

Most apps will need only one connection entry.

Each connection has it's own message processors for the incoming messages it recieves after the connection established.

So in broad sense, the connection entry point sits as hhtp route payload (the request to connect comes with get header) and when you add it to routes map, it starts it's own parralel listening to the messages it recieves (so called ws route map), and has the specialized for that connection type message controllers. So it's like server in server `exibit.jpg` ( they are not handlers in sense of Crystal's HTTP::Handler) .

To get it easier, just imagine that you would have some controller that is responsible for first user entry, later on as soon as user hits that controller - you will have the special routing and controllers specific to that users.
same here for ConnectionEntry (you can think of it as connection type), has it's own routing which will call own controllers when something is passed as message through that connections

Thanks to crystal you can literraly have tens of thousands connections on one machine (don't forget to set `ulimit -n overninethousand` on ubuntu)

This way there's no need to spam multiple connection (say e.g. rooms etc.), only one connection is used per user.

##### That sounds a bit complicated but just look at this snippet and you'll get how easy it is
you just reason about it the same as you would do with standart Http routing
```ruby
# in router
ws_connection "/general", to: "general" do
  msg "/index", to: "test#index"
  scope "/test" do
    msg "/show", to: "test#show"
  end
  msg "/delete", to "test#delete"
end

in app/ws/connection_entries
class App::WS::ConnectionEntries::General < Shepherd::WebSockets::ConnectionEntry::Base
  def self.on_connection_request(context : HTTP::Server::Context) : Nil
    if current_user.ok? #fantasy method
       #since this point server will try establish connection,
       #if socket connected #on_connection_established with fresh connection will be called
      connect
    else
     reject
    end
  end

  def on_connection_established(socket : HTTP::WebSocket, context : HTTP::Server::Context) : Nil
    socket.send "established #{socket}"
    #register connection in redis for example
    RedisBaz.register[current_user.id] = socket #fantasy
  end

  def on_connection_closing(socket : HTTP::WebSocket, context : HTTP::Server::Context) : Nil
    socket.send "disconnecting #{socket}"
    RedisBaz.cleanup current_user.id #fantasy
  end

end

in app/ws
class App::WS::MessageControllers::Test

  def index
    add connection, to: "test_channel" #fantasy
    send "message path was sent to /index, hello from test#index to #{connection}"
  end

  def show
    if connection, in: "test_channel" #fantasy
        user =  User.find(json_any_payload["user_id"]).to_s #this is fantasy method
        send "#{user}#{connection}"
    else
      send "sorry #{connection} you should've sent message to /bar/baz before to be able to see user "
    end
  end

  def delete
    send "bye #{connection}"
    purge connection, from: "test_channel" if in: "test_channel"
    to users, in : "test_channel", send: "#{connection} has left test_channel" #fantasy
    disconnect
  end
end

in js.
on client 1 (ok user)
var socket = new WebSocket("ws://localhost/general")
#=> established #<WebSocket:0x0000001>
socket.send("/index|")
#=> "message path was sent to /index, hello from test#index #<WebSocket:0x0000001>"
socket.send("/test/show|{user_id: 4}")
#=> {id: 4, name: "Joe"}#<WebSocket:0x0000001>
socket.close()
#=> disconnecting #<WebSocket:0x0000001>

on client 2 (ok user)
var socket = new WebSocket("ws://localhost/general")
#frame=> established #<WebSocket:0x0000002>
socket.send("/index|")
#frame=> "message path was sent to /index, hello from test#index #<WebSocket:0x0000002>"
socket.send("/test/show|{"user_id": 3}")
#frame=> {id: 3, name: "Schmoe"}#<WebSocket:0x0000002>
socket.send("/disc")
#frame=> bye #<WebSocket:0x0000002>
#frame=> disconnecting #<WebSocket:0x0000002>
socket.send("/index|")
#=> error socket is already in CLOSING or CLOSED state.

on client 3 (not ok user)
var socket = new WebSocket("ws://localhost/general")
#=> Error during WebSocket handshake: Unexpected response code: 401

on client 4 (ok user)
var socket = new WebSocket("ws://localhost/general")
#frame=> established #<WebSocket:0x0000003>
socket.send("/test/show|{user_id: 7}")
#frame=> "sorry #<WebSocket:0x0000003> you should've sent message to /index before to be able to see user "
socket.send("/index|")
#frame=> "message path was sent to /index, hello from test#index #<WebSocket:0x0000002>"
after client 2 and 1 sent message to  /disc
#=>"#<WebSocket:0x0000002> has left test_channel"
#=>"#<WebSocket:0x0000001> has left test_channel"

```
TODO: describe WS routing; ws connection entries; ws message controllers

# Models <a name="model"></a>
supported:
- persistence
- destruction
- querying
- collections
- associations, including polymorphic

# defining a model

define a repository, specify connection. One of the features is that you can use multiple dbs. only postgress adapter exists.
```crystal
class Model::AppDomainBase < Shepherd::Model::Base

  default_repository({
    connection: Shepherd::Database::DefaultConnection,
    adapter: :Postgres
  })
  #
  # add_repository(
  #   accessor_method: secondary_repository,
  #   connection: MyCustomConnection,
  #   adapter: :Postgres
  # )
  # def secondary_repository : Shepherd::Model::Repository::Base
  #   Shepherd::Model::Repository::Base(Shepherd::Model::QueryBuilder::Adapters::Postgress, MyCustomConnection, self).new(self)
  # end
end
```
define models and table mapping:
```crystal
class User < Model::AppDomainBase
  database_mapping(
    { table_name: "users",
      column_names: {
        "id": {type: Int32, primary_key: true},
        "name": {type: String},
        "email": {type: String},
        "user_id": {type: Int32},
        "age": {type: Int32}
      }
    }
  )
  #for demo to show hm and ho
  associations_config({
    accounts: {
      type: :has_many, class_name: Account,
      local_key: "id", foreign_key: "user_id"
    },
    account: {
      type: :has_one, class_name: Account,
      local_key: "id", foreign_key: "user_id"
    }
  })
end

class Post < Model::AppDomainBase

  database_mapping(
    { table_name: "posts",
      column_names: {
        "id": {type: Int32, primary_key: true},
        "title": {type: String},
        "user_id": {type: Int32}
      }
    }
  )

  associations_config({
    user: {
      type: :belongs_to, class_name: User,
      local_key: "user_id", foreign_key: "id"
    },
    account: {
      type: :has_one, class_name: Account,
      through: :user, this_joined_through: :user,
      that_joined_through: :account#, alias_on_join_as: "user_posts"
    }
  })
end

```
# Persisting
```crystal
user = User.new
user.name = "joe schmoe"
user.email = "joe@schmoe.com"
user.repo.create

user = User.repo.where({"name", :eq, "joe schmoe"})
  .limit(1)
  .get.not_nil! #User {id: 1, name: "joe schmoe", email: "joe@schmoe.com"}

user.repo
    .create("name") #only name field would be persisted

```
#Destroying
```crystal
user_to_be_destroyed = User.repo.where({"name", :eq, "to be destroyed"}).get.not_nil!
user.repo.delete
```
#querying
```crystal
#get collection
users = User.repo
      .where({"name", :eq, "asdasd"})
      .list #Shepherd::Model::Collection(User)
#get single entry
user = User.repo
      .where({"name", :eq, "asdasd"}) 
      .get #User
that_user = User.repo
      .where({"age", :gt, 20})
      .get.not_nil!
that_user = User.repo
      .where({"age", :lt, user.age.not_nil! + 1})
      .get.not_nil!
#chain conditions
user = User.repo
    .where({"id", :gt, 0})
    .where("users.age = $2", 20)
    .list[0]
#joins, including raw joins
user = User.repo
    .where({"id", :gt, 0})
    .inner_join(&.accounts)
    .raw_join("INNER JOIN accounts foos on foos.user_id = users.id")
    .get
#ordering
user = User.repo
        .order(User, "id", direction: :desc)
        .limit(1)
        .get.not_nil!
```
#updating
```crystal
that_user = User.repo.where({"name", :eq, "to_be_updated"}).get.not_nil!
that_user.name = "update_done" #updates only name
that_user.repo.update(:name) #otherwise all fields be updated
```
#associations
there's no automatic association creation. You have to manually create ones, like
```crystal
user = User.new
user.repo.create
account = Account.repo.where({"id", :eq, 10})
account.user_id = user.id
```
But you can query them in comfortable manner.
```crystal
account.user #will execute query and load it while assigning to field
account.user(load: false) #will not load

#joining
Account.repo
    .inner_join(&.user)
    .where(User, {"name", :eq, "joe"})
    .get
#eager loading
Account.repo
  .where(Account, {"name", :eq, "account"})
  .eager_load(&.user) #will load the collection/single if h.o
  .get.not_nil!
#adding conditions on associated
account.user(yield_repo: true) do |repo|
    repo.where("age", :gt, 20).get
end # will load user adding the conditions. As well there you can call eager loads on next associations and so on.
```

# credits and thanks
Super thanks to crystal core team and @asterite specifically for giving us Crystal

Router uses luislavena/radix, thanks that shard is awesome!

Learned from manastech/frank, sdogruyol/kemal you're awesome!

