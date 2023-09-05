---
layout: post
title:  "What if I used create_or_find_by instead of find_or_create_by?"
date:   2023-09-06 01:00:00 +0200
tags: ruby rails ActiveRecord
categories: rails
---

I was working on a rake task that needed to create a record in the assignments table if it did not exist. The assignments table had two columns: `lecturer_id` and `student_id`. There was also a unique index on these two columns in the database.

The Assignment model had a validation that ensured the uniqueness of the `student_id` in the scope of the `lecturer_id`:

```ruby
class Assignment < ApplicationRecord
  belongs_to :lecturer
  belongs_to :student

  validates :student_id, uniqueness: {scope: :lecturer_id}
end
```

The goal of the rake task was to assign a random lecturer to each student who did not have one yet.
Usually, I use `find_or_create_by` to achieve this functionality. This method runs a select query and returns the record if it exists, or creates a new record in the database and returns it if it does not exist.

However, for some reason, I used `create_or_find_by` instead.

When I finished writing the logic and the tests, I noticed that the tests were failing. The `create_or_find_by` method was returning an object without an ID and the record was not persisted in the database. I tried to debug the issue.

### Why My Logic Did Not Work

I thought this might be related to an issue in Rspec or something else that did not make sense. So, while debugging, I added a bang to `create_or_find_by` to make it `create_or_find_by!`. This raised an `ActiveRecord::RecordInvalid` error and said that the record was not unique. This was the error coming from the Rails validation that I added in the model.

I decided to remove the validation from the model and run the tests again. To my surprise, they worked fine. But this did not make sense to me. Why would a model validation prevent a Rails method from working properly?

After searching online, I found out that `create_or_find_by` relies on rescuing the `ActiveRecord::RecordNotUnique` exception that is raised from the database when there is a duplicate record that violates the unique constraint. This means that the model validation was actually causing the problem because it raised a different exception: `RecordInvalid` instead of `RecordNotUnique`.

I also found that there was already a [GitHub issue](https://github.com/rails/rails/issues/36027) for this. It was closed by a [pull request](https://github.com/rails/rails/pull/42625) that updated the documentation to state explicitly that a database constraint was required to use this method. But this still did not explain why having a model validation would break it. Obviously the implementation of `create_or_find_by` is causing a confusion.

### How I Fixed The Logic

Then it dawned on me that I had used the wrong method. I had swapped the words create and find. Instead of using `find_or_create_by`, I had used `create_or_find_by`.
I replaced `create_or_find_by` with `find_or_create_by` and added back the model validation. The tests passed successfully.

Using `create_or_find_by` was a nice mistake that I learned from. But it also left me wondering why Rails had a method that contradicted its own model validation.

You can read more about the benefits and drawbacks of each method in the sources below.

### References

- [create_or_find_by](https://apidock.com/rails/v6.0.0/ActiveRecord/Relation/create_or_find_by)
- [find_or_create_by](https://apidock.com/rails/v6.0.0/ActiveRecord/Relation/find_or_create_by)
