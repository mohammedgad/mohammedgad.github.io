---
layout: post
title:  "How alias_method Can Break Your Ruby Code (And How to Fix It)"
date:   2023-09-10 14:00:00 +0200
tags: ruby
categories: ruby
---

### Introduction

In Ruby, you can use the `alias_method` to create a second name for a method. This can be useful for making your methods more expressive or intuitive, and there is another interesting usage too that we will discuss later. However, there is a potential pitfall that you need to be aware of when using `alias_method`. In this article, I will show you an example of how `alias_method` can affect the behavior of your code and how to avoid it.

### The Problem

Let’s say we have a class `A` that has a method `print_secret` that prints a secret message:

```ruby
class A
  def print_secret
    puts "1337 SECRET REVEALED"
  end
end
```

Now, we want to create a subclass `B` that inherits from `A` and overrides the `print_secret` method to print something else:

```ruby
class B < A
  def print_secret
    puts "NOT ALLOWED"
  end
end
```

If we create an instance of `B` and call the `print_secret` method, we get the expected output:

```ruby
B.new.print_secret
# => NOT ALLOWED
```

However, what if we want to have another name for the `print_secret` method in `B`, such as `not_allowed_secret`? We might think that we can use `alias_method` to create an alias for the `print_secret` method in `B`, like this:

```ruby
class B < A
  alias_method :not_allowed_secret, :print_secret

  def print_secret
    puts "NOT ALLOWED"
  end
end
```

But if we try to call the `not_allowed_secret` method on an instance of `B`, we get a surprising result:

```ruby
B.new.not_allowed_secret
# => 1337 SECRET REVEALED
```

What happened here? Why did the alias point to the original method in `A`, instead of the overridden method in `B`?

This happens because `alias_method` makes a new method that calls the defined method where we use `alias_method`. In class `B`, we put `alias_method` before the new `print_secret` method, so it made a new method `not_allowed_secret` that calls the old `print_secret` method in `A`, because the new method is not defined yet where we call the `alias_method`. 

This means that using `not_allowed_secret` on a B object will do the same thing as using `print_secret` on an `A` object.
This is the second usage I mentioned earlier which is that you can retain access to overridden methods which is very cool. However, this can be a serious problem if we are not careful about the order of `alias_method`. We might unintentionally mess up the logic and result in the wrong output.

### The Solution

The solution to this problem is simple if you just need an alias: we just need to move the `alias_method` statement after the definition of the new method, like this:
```ruby
class B < A
  def print_secret
    puts "NOT ALLOWED"
  end

  alias_method :not_allowed_secret, :print_secret
end
```

Now, both names will point to the same method in `B`, and we get the expected output:
```ruby
B.new.print_secret
# => NOT ALLOWED

B.new.not_allowed_secret
# => NOT ALLOWED
```
This way, we can use `alias_method` safely and avoid any unwanted behavior.

### Conclusion

In this article, I showed you how the order of `alias_method` can affect the behavior of your code in Ruby. You learned that you need to be careful about when you use `alias_method`, and always put it after the definition of the new method if you want to alias a method. This will ensure that you create an alias for the correct method and avoid any potential pitfalls. Or you can put it before the new definition to regain access to the original method.

I hope you found this article useful and learned something new about Ruby. Thank you for reading!

### References

- [alias_method](https://apidock.com/ruby/Module/alias_method)
