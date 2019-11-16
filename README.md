1. 创建新项目，`rails new graphql_fun_demo --api --skip-test`
2. 建立模型数据（建表）， 
    `rails g model User email:string name:string`
  `rails g model Post user:belongs_to title:string body:text`
 `rails db:migrate`
3. 加入gem包graphiql-rails、faker，
```ruby
group :development do
  gem 'listen', '>= 3.0.5', '< 3.2'
  # Spring speeds up development by keeping your application running in the background. Read more: https://github.com/rails/spring
  gem 'spring'
  gem 'spring-watcher-listen', '~> 2.0.0'
  gem 'graphql'
  gem 'graphiql-rails'
  gem 'faker'
end
```
4. 在seed.rb中利用faker包快速生成，生成用户，文章测试数据,用命令 `rails db:seed`
```ruby
5.times do
  user = User.create(name: Faker::Name.name,email: Faker::Internet.email)
  5.times do
    user.posts.create(title: Faker::Lorem.sentence(word_count:3), body: Faker::Lorem.paragraph(sentence_count:3))
  end
end
```
6. `rails g graphql:object user`
   `rails g graphql:object post`
7. 修改route.rb
```ruby
Rails.application.routes.draw do
  if Rails.env.development?
    mount GraphiQL::Rails::Engine, at: "/graphiql", graphql_path: "graphql#execute"
  end
  post "/graphql", to: "graphql#execute"
  # For details on the DSL available within this file, see http://guides.rubyonrails.org/routing.html
end
```
在application.rb中取消`require "sprockets/railtie"`的注释
现在启动项目，访问http://localhost:3000/graphiql，发现以下报错：
```ruby
ActionController::RoutingError (No route matches [GET] "/javascripts/graphiql/rails/application.js"):

ActionController::RoutingError (No route matches [GET] "/stylesheets/graphiql/rails/application.css")
```
解决办法：
在app下新建目录assets/config，新建文件manifest.js,内容为：
```javascript
//= link graphiql/rails/application.css
//= link graphiql/rails/application.js
```
然后就成功访问`http://localhost:3000/rails/info/routes`，`http://localhost:3000/graphiql`
8. 编写UserType、PostType
```ruby
module Types
  class UserType < Types::BaseObject
    field :id, ID, null: false
    field :name, String, null: true
    field :email, String, null: true
    field :posts, [Types::PostType], null: true
    field :posts_count, Integer, null: true


    def posts_count
      object.posts.size
    end
  end
end


module Types
  class PostType < Types::BaseObject
    field :id, Integer, null: false
    field :title, String, null: false
    field :body, String, null: false
  end
end
```
9. 编写QueryType
```ruby
module Types
  class QueryType < Types::BaseObject
    # /users
    field :users, [Types::UserType], null: false

    def users
      User.all
    end

    # /user/:id
    field :user, Types::UserType, null: false do
      argument :id, ID, required: true
    end

    def user(id:)
      User.find(id)
    end
  end
end

```
10. 查询结果：
```ruby
query{
  users{
    id
    name
    email
    postsCount
  }
}


{
  "data": {
    "users": [
      {
        "id": "1",
        "name": "Williams Heller",
        "email": "collen.deckow@schambergersauer.io",
        "postsCount": 0
      },
        ......
      {
        "id": "6",
        "name": "Jackelyn Hoeger",
        "email": "darin@hansen.co",
        "postsCount": 5
      }
    ]
  }
}

query{
  user(id: 2){
    id
    name
    email
    posts{
      id
      title
      body
    }
  }
}


{
  "data": {
    "user": {
      "id": "2",
      "name": "Ira Cremin",
      "email": "rayford_hills@weimann.biz",
      "posts": [
        {
          "id": 1,
          "title": "Maxime impedit non.",
          "body": "Natus facilis ea. Eius at rerum. Omnis consequatur ut."
        },
        ......
        {
          "id": 5,
          "title": "Dolores nulla aut.",
          "body": "Tempora praesentium consequuntur. Voluptate vel aut. Cupiditate id voluptates."
        }
      ]
    }
  }
}
```
11. graphql中修改例子，添加base_mutation.rb文件
```ruby
class Mutations::BaseMutation < GraphQL::Schema::RelayClassicMutation
end
```
create_user.rb文件
```ruby
class Mutations::CreateUser < Mutations::BaseMutation
  argument :name, String, required: true
  argument :email, String, required: true

  field :user, Types::UserType, null: false
  field :errors, [String], null: false

  def resolve(name:, email:)
    user = User.new(name: name, email: email)
		if user.save
      {
       	user: user,
       	errors: []
      }
    else
      {
      	user: nil,
        errors: user.errors.full_messages
      }
    end
  end
end
```
修改mutation_type.rb
```ruby
module Types
  class MutationType < Types::BaseObject
    field :create_user, mutation: Mutations::CreateUser
  end
end
```
12. 测试mutaiton接口
```ruby
mutation{
  createUser(input:{
    name: "jujuxia",
    email: "nameJujuxia@test.com"
  }){
    user{
      id,
      name,
      email
    },
    errors
  }
}

{
  "data": {
    "createUser": {
      "user": {
        "id": "7",
        "name": "jujuxia",
        "email": "nameJujuxia@test.com"
      },
      "errors": []
    }
  }
}
```





 