# Ki Framework

[![Build Status](https://travis-ci.org/mess110/ki.svg?branch=master)](https://travis-ci.org/mess110/ki)
[![Code Climate](https://codeclimate.com/github/mess110/ki/badges/gpa.svg)](https://codeclimate.com/github/mess110/ki)
[![Test Coverage](https://codeclimate.com/github/mess110/ki/badges/coverage.svg)](https://codeclimate.com/github/mess110/ki)

Tiny REST JSON ORM framework with MongoDB.

Ki's goal is to help you protoype your ideas blazing fast. It has a db backend
and provides a fullblown REST api on top.

**Table of Contents**

- [TLDR](#tldr)
- [Install](#install)
- [Getting started](#getting-started)
  - [Create a new app](#create-a-new-app)
  - [Adding a view](#adding-a-view)
  - [Adding assets](#adding-assets)
  - [View layout](#view-layout)
  - [Middleware](#middleware)
  - [Helpers](#helpers)
  - [Adding a resource](#adding-a-resource)
    - [Create](#create)
    - [Get all](#get-all)
    - [Get by id](#get-by-id)
    - [Get advanced queries](#advanced-queries)
    - [Update](#update)
    - [Delete](#delete)
  - [Required attributes](#required-attributes)
  - [Unique attributes](#unique-attributes)
  - [Restricting resource requests](#restricting-resource-requests)
  - [Before/after callbacks](#beforeafter-callbacks)
    - [Accessing the json object within the callback](#accessing-the-json-object-within-the-callback)
  - [Exceptions](#exceptions)
- [Deploy](#deploy)
  - [Heroku deployment](#heroku-deployment)
- [CORS enabled](#cors-enabled)
- [Documentation](#documentation)
- [Administration Interface](#administration-interface)

## TLDR

To create an api endpoint '/todo.json' with GET/POST/PATCH and restricting
DELETE, with title and description as required attributes, with db backend,
with a unique title, and before/after filters you would need to write:

```ruby
class Todo < Ki::Model
  requires :title, :description
  unique :title
  forbid :delete

  def before_all
    puts 'hello'
  end

  def before_create
    params['created_at'] = Time.now.to_i
  end

  def after_find
    if result['keywords'].include? 'food'
      puts 'yummy'
    end
  end
end
```

## Install

```shell
gem install ki
```

## Getting started

Learn by example. We will create the traditional 'hello world' app for web
development: the dreaded TODO app.

View the [code](https://github.com/mess110/ki/tree/master/spec/examples/todo) for
the final webapp.

### Creating a new app

```shell
ki new todo
```

This will create the folder *todo* containing a bare bones ki application. Your
app will look [like this](https://github.com/mess110/ki/tree/master/spec/examples/base).

App directory structure:

* public/
  * javascripts/
  * stylesheets/
* views/
* app.rb
* config.yml

The entry point for the app is *app.rb*. You will add most of your code there.
Views are in the *views* folder and you can find some database info in
*config.yml*.

```shell
cd todo
bundle
ki server
```

*ki server* is the command which starts the webserver. By default it starts on
port 1337.

For an interactive environment, you can run the *console* command. It starts a
irb session with your application context already loaded.

```shell
ki console
```

### Adding a view

A view is a html page. By default, a view called 'index.haml' is created for
you in the 'views/' folder. All haml files from the '/views/' folder are
compiled to html at runtime.

For example, to serve a html file when a user accesses '/dashboard' create the
'views/dashboard.haml' dashboard.

### Adding assets

All files from the 'public/' folder are served to the client from the root url.

You can add any type of file. Coffee and sass files are compiled at runtime,
similar to haml.

### View layout

If the file *views/layout.haml* exists, it will be used as a layout for all the
other haml files. The content of the route will be placed in the layout *yield*

*views/layout.haml*

```haml
!!!
%html
  %head
    %title Ki Framework
  %body
    = yield
```

*views/index.haml*

```haml
%p Hello World!
```

### Middleware

Views, assets, haml, less, sass are all handled through Rack middleware.

You can learn more about ki middleware [here](MIDDLEWARE.md).

### Helpers

To reduce complexity in your views, you can use helpers. Here we create a
*say_hello* method which we can use in *views/index.haml*.

*app.rb*

```ruby
module Ki::Helpers
  def say_hello
    'hello'
  end
end
```

*views/index.haml*

```haml
%p= say_hello
```

### Adding a resource

Think of resources as the M in MVC. Except they also have routes attached.
All resources must inherit from Ki::Model. For example, to create a todo model
add the fallowing snippet to *app.rb*

```ruby
class Todo < Ki::Model
end
```

This will create the Todo resource and its corresponding routes. Each route is
mapped to a model method.

Method name | Verb | HTTP request|Required params|
------------|------|-------------|---------------|
find        |GET   | /todo.json  |               |
create      |POST  | /todo.json  |               |
update      |PATCH | /todo.json  |id             |
delete      |DELETE| /todo.json  |id             |

Below is a curl example for creating a todo item. Notice the data sent in the
body request is a JSON string. It can take the shape of any valid JSON object.
All json objects will be stored in the database under the resource name. In our
case *todo*.

#### Create

```shell
curl -X POST -d '{"title": "make a todo tutorial"}' http://localhost:9292/todo.json
```

#### Get all

```shell
curl -X GET http://localhost:9292/todo.json
```

#### Get by id

```shell
curl -X GET http://localhost:9292/todo.json?id=ITEM_ID
```

#### Advanced queries

Mongo syntax is supported. See [here](http://docs.mongodb.org/manual/tutorial/query-documents/) for more details.

Some examples:

Search in array

```
curl -k -X GET -d '{"category": {"$in": ["music"]}}' "https://json.northpole.ro/storage.json?api_key=guest&secret=guest"
```

Greater than number

```
curl -k -X GET -d '{"i": {"$gt": 0}}' "https://json.northpole.ro/storage.json?api_key=guest&secret=guest"
```

OR Query

```
curl -k -X GET -d '{"$or": [{"category": "music"}, {"category": "band"}]}' "https://json.northpole.ro/storage.json?api_key=secret&secret=secret"
```

#### Update

```shell
curl -X PATCH -d '{"id": "ITEM_ID", "title": "finish the todo tutorial"}' http://localhost:9292/todo.json
```

#### Delete

```shell
curl -X DELETE -d http://localhost:9292/todo.json?id=ITEM_ID
```

### Required attributes

You can add mandatory attributes on resources with the *requires* method. It
takes one parameter or an array of parameters as an argument.

```ruby
class Todo < Ki::Model
  requires :title
end
```

This will make sure a Todo item can not be saved or updated in the database
without a title attribute.

### Unique attributes

You can add unique attributes on resources with the *unique* method. It
takes one parameter or an array of parameters as an argument.

```ruby
class Todo < Ki::Model
  unique :title
end
```

This will make sure a Todo item can not be saved or updated in the database
if it is not unique.


### Restricting resource requests

Let's say you want to forbid access to deleting items. You can do that with
the *forbid* method.

```ruby
class Todo < Ki::Model
  forbid :delete
end
```

### Before/after callbacks

The framework has [these callbacks](https://github.com/mess110/ki/blob/master/lib/ki/modules/callbacks.rb).
Here is an example on how to use them:

```ruby
class Todo < Ki::Model
  def before_create
    # do your stuff
  end
end
```

#### Accessing the json object within a callback

Before the request is sent to the client, you can look at the result through
the *result* method. Modifying it will change what the client receives.

```ruby
class Todo < Ki::Model
  def after_find
    puts result
  end
end
```

### Exceptions

A list of exceptions can be found [here](https://github.com/mess110/ki/blob/master/lib/ki/api_error.rb)

```ruby
class Todo < Ki::Model
  def before_all
    ensure_authroization
  end

  private

  def ensure_authorization
    if params[:key] = 'secret-key'
      raise UnauthorizedError.new
    end
  end
end
```

## Deploy

It has a *config.ru* file. Ki is based on *rack*. You can deploy anywhere (ex:
nginx, thin, apache, webrick).

In the webserver config, just remember to point the virtual host to the
*public* directory.

### Heroku deployment

```shell
ki new heroku-ki
cd heroku-ki
bundle
git init
git add .
git commit -m 'initial commit'
heroku create
heroku config:set MONGODB_URI="mongodb://user:pass@mongo_url:mongo_port/db_name"
git push heroku master
heroku open
```

## CORS enabled

Yes.

## Documentation

Ki offers instant documentation. Add the middleware 'DocGenerator' and access
*/instadoc*.

## Administration Interface

Ki offers an instant administration interface. Add the middleware 'AdminIntefaceGenerator'
and access */instadmin*.

## Tasks

Ki offers a simple task framework. [More info here](https://github.com/mess110/ki/tree/master/spec/examples/tasks-examples/).
