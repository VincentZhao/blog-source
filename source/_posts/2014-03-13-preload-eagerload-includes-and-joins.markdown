---
layout: post
title: "Preload, Eagerload, Includes 和 Joins"
date: 2014-03-13 21:21
comments: true
categories: translation
---
原文：[Preload, Eagerload, Includes and Joins](http://blog.bigbinary.com/2013/07/01/preload-vs-eager-load-vs-joins-vs-includes.html)

Rails 提供了4种方式来加载关联表的数据。在这篇文章中，我们来分别来看看这些方法。

## Preload

`preload` 使用一条附加的查询语句来加载关联数据。

```ruby
User.preload(:posts).to_a

# =>
SELECT "users".* FROM "users"
SELECT "posts".* FROM "posts"  WHERE "posts"."user_id" IN (1)
```

这也是 `includes` 默认的加载数据的方式。

`preload` 总是会生成两条 SQL 语句，所以我们不能在 where 条件中使用 `posts` 表。比如下面的查询就会报错。

```ruby
User.preload(:posts).where("posts.desc='ruby is awesome'")

# =>
SQLite3::SQLException: no such column: posts.desc: 
SELECT "users".* FROM "users"  WHERE (posts.desc='ruby is awesome')
```

`preload` 也可以指定 where 条件。

```ruby
User.preload(:posts).where("users.name='Neeraj'")

# =>
SELECT "users".* FROM "users"  WHERE (users.name='Neeraj')
SELECT "posts".* FROM "posts"  WHERE "posts"."user_id" IN (3)
```

## Includes

和 `preload` 一样，`includes` 也使用一条附加的查询语句来加载关联数据。

然而，它要比 `preload` 更聪明一些。我们刚刚看到了使用 `preload` 无法查询 `User.preload(:posts).where("posts.desc='ruby is awesome'")`。让我们使用 `includes` 来试试看。

```ruby
User.includes(:posts).where('posts.desc = "ruby is awesome"').to_a

# =>
SELECT "users"."id" AS t0_r0, "users"."name" AS t0_r1, "posts"."id" AS t1_r0, 
       "posts"."title" AS t1_r1, 
       "posts"."user_id" AS t1_r2, "posts"."desc" AS t1_r3 
FROM "users" LEFT OUTER JOIN "posts" ON "posts"."user_id" = "users"."id" 
WHERE (posts.desc = "ruby is awesome")
```

如你所见，`includes` 不再使用两条查询语句，而是使用了单独一条 `LEFT OUTER JOIN` 语句来获取数据，而且也加载了 where 条件。

所以，`includes` 在某些场合会从两次查询变成一次查询。默认的简单情况下它会使用两次查询。如果出于某些原因，你想强制其使用单次查询，可以使用 `reference`。

```ruby
User.includes(:posts).references(:posts).to_a

# =>
SELECT "users"."id" AS t0_r0, "users"."name" AS t0_r1, "posts"."id" AS t1_r0, 
       "posts"."title" AS t1_r1, 
       "posts"."user_id" AS t1_r2, "posts"."desc" AS t1_r3 
FROM "users" LEFT OUTER JOIN "posts" ON "posts"."user_id" = "users"."id"
```

上面的例子就只执行了一次查询。

## Eager load

`eager_load` 使用 `LEFT OUTER JOIN` 进行单次查询，并加载所有的关联数据。

```ruby
User.eager_load(:posts).to_a

# =>
SELECT "users"."id" AS t0_r0, "users"."name" AS t0_r1, "posts"."id" AS t1_r0, 
       "posts"."title" AS t1_r1, "posts"."user_id" AS t1_r2, "posts"."desc" AS t1_r3 
FROM "users" LEFT OUTER JOIN "posts" ON "posts"."user_id" = "users"."id"
```

这与 `includes` 在 `where` 或 `order` 语句中指定了 `posts` 表的属性的情况下的单次查询完全相同。

## Joins

`joins` 使用 `INNER JOIN` 来加载关联数据。

```ruby
User.joins(:posts)

# =>
SELECT "users".* FROM "users" INNER JOIN "posts" ON "posts"."user_id" = "users"."id"
```

上面的语句不会查询出 `posts` 表的数据。这个查询还可能会得到重复的结果，我们先创建一些数据。

```ruby
def self.setup
  User.delete_all
  Post.delete_all

  u = User.create name: 'Neeraj'
  u.posts.create! title: 'ruby', desc: 'ruby is awesome'
  u.posts.create! title: 'rails', desc: 'rails is awesome'
  u.posts.create! title: 'JavaScript', desc: 'JavaScript is awesome'

  u = User.create name: 'Neil'
  u.posts.create! title: 'JavaScript', desc: 'Javascript is awesome'

  u = User.create name: 'Trisha'
end
```

有了上面的数据后，当我们执行 `User.joins(:posts)` 后得到的结果为

```
#<User id: 9, name: "Neeraj">
#<User id: 9, name: "Neeraj">
#<User id: 9, name: "Neeraj">
#<User id: 10, name: "Neil">
```

要去除重复数据，可以使用 `distinct`。

```ruby
User.joins(:posts).select('distinct users.*').to_a
```

另外，如果我们想要得到 `posts` 表中的属性值，就必须显式地 `select` 它们。

```ruby
records = User.joins(:posts).select('distinct users.*, posts.title as posts_title').to_a
records.each do |user|
  puts user.name
  puts user.posts_title
end
```

值得注意的是，使用 `joins` 就意味着 `user.posts` 会执行一次新的查询。
