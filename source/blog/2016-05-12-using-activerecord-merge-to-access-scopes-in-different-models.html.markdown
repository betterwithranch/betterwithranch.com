---
title: Using ActiveRecord's merge to access scopes in different models
date: 2016-05-12
author: Craig Israel
tags: ActiveRecord Rails
---

One of the things that I love about ActiveRecord is that it allows me to build sql by using 
composition.  By chaining together scopes, I have reusable snippets of sql code that can
be combined to build a complex query.  Anyone that has used ActiveRecord has seen this in action:

```ruby
class Child < ActiveRecord::Base
  def self.active
    where(active: true)
  end

  def self.recent
    where("created_at > ? ", Date.today - 5.days)
  end
end

Child.active.recent
```

This generates a sql statement by composing the chained scopes.

```sql
SELECT "children".* FROM "children" WHERE "children"."active" = 't' AND (created_at > '2016-05-06')"
```

Ok, that's great, but what if you want to limit the children by some attribute of its parent?
For example, what if instead of active children, we only want recent Child records where the parent is active?

First, we need to add the relationship to `Child`.

```ruby
class Child < ActiveRecord::Base

  belongs_to :parent

  def self.active
    where(active: true)
  end

  def self.recent
    where("created_at > ? ", Date.today - 5.days)
  end
end
```

Next, we create the parent class.

```ruby
class Parent < ActiveRecord::Base
  
  has_many :children

  def self.active
    where(active: true)
  end
end
```

If you've run into this before, you probably joined to (or included) the parent in the query and added a where clause for the
`active=true` condition like this ...

```ruby
Child.recent.
  joins(:parent).
  where(parents: { active: true })
```

This works and generates the sql

```sql
SELECT "children".* FROM "children" WHERE "children"."active" = 't' AND (created_at > '2016-05-06')" AND
"parents"."active" = 't'
```

But, besides being a little verbose, it doesn't reuse the existing `active` scope on the `Parent` class.  What
happens if the definition of the parent's `active` scope changes?  How would we know that this code needs to be changed as well?

`ActiveRecord#merge` to the rescue.  Using `merge`, you can just merge in the scope from the other class.

```ruby
Child.recent.
  joins(:parent).
  merge(Parent.active)
```

This generates the exact same sql as above, but provides a number of advantages to the readability and
maintainability of our code.  By reusing the existing scope on the `Parent` class, we have DRYed up our
code and made it more intention-revealing.  Also, we have now abstracted the implementation detail of
the parents table's schema.

One important thing to remember when using merge is that you need to either join or include the parent table
explicitly.  Without it, you'll get an error like `no such column: parents.active`.

ActiveRecord is a huge framework and there are a ton of great methods in it.  I hope you find this one
useful.  I'm going to try to introduce more in future posts.  Until next time ...
