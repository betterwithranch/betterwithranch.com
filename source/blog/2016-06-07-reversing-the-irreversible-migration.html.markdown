---
title: Reversing The Irreversible Migration
date: 2016-06-07 14:43 UTC
tags: ActiveRecord Rails
---

If you've been writing migrations for ActiveRecord for very long, you have eventually encountered this
error when trying to rollback a migration.  

```
== 20160607145211 Irreversible: reverting =====================================
rake aborted!
StandardError: An error has occurred, this and all later migrations canceled:

ActiveRecord::IrreversibleMigration
```

This typically occurs because the previous state is unknown when rolling back, so a reversal can't be
generated.  For example, we cannot rollback this migration.

```ruby
class MyMigration < ActiveRecord::Migration
  def change
    change_column :children, :id, :bigint
  end
end
```

Nothing in this migration indicates what the previous type was, so ActiveRecord cannot generate
the reverse statement.  Fortunately, there are a couple of ways to fix this problem.  First,
we could split the `change` method out into `up` and `down` methods.

```ruby
class MyMigration < ActiveRecord::Migration
  def up
    change_column :children, :id, :bigint
  end

  def down
    change_column :children, :id, :integer
  end
end
```

This works great in this simple example, but what if you have a longer migration and only part of
it is irreversible?

```ruby
class MyMigration < ActiveRecord::Migration
  def change

    # This is reversible
    add_index :children, :parent_id
    add_index :children, :active

    # This is irreversible
    change_column :children, :id, :bigint

  end
end
```

You could rewrite this as `up` and `down` methods.

```ruby
class MyMigration < ActiveRecord::Migration
  def up

    add_index :children, :parent_id
    add_index :children, :active

    change_column :children, :id, :bigint

  end

  def down
    change_column :children, :id, :integer

    remove_index :children, :active
    remove_index :children, :parent_id
  end
end
```

This will work, but is undesirable for a couple of reasons.  First, the `remove_index` calls
are duplicating functionality that ActiveRecord migrations already provide.  This is a pretty
straightforward example, but for some other migration commands, it would be best to allow the
tested framework code to do this.

Also, by explicitly calling the `add_index` and `remove_index` statements,
we have introduced the possibility of a bug by typing the wrong table or column names.

Finally, in this case, the order of the statements is not significant, but if it were, this would
be a much more complex `down` method to implement because we would need to call them in the
reverse order of the `up` method.

Fortunately, ActiveRecord migrations already provide a solution for us.  The `reversible` method.
Basically, this method allows you to specify up and down behavior for part of the migration.

```ruby
class Irreversible < ActiveRecord::Migration
  def change

    add_index :children, :parent_id
    add_index :children, :active

    reversible do |dir|
      dir.up do
        change_column :children, :id, :bigint
      end
      dir.down do
        change_column :children, :id, :int

      end
    end
  end
end
```

By using the `reversible` method, we can define the up and down behavior in blocks that will
execute when the change method is migrating up or down.  In addition to removing the duplication
and complexity of the previous example, this clearly specifies the fact that there is an
irreversible task occurring.  Since this is often a task that should be inspected carefully
because of it's possible data loss effects, this is a nice benefit.

It's a good habit to test the rollback of every migration that you write to make sure that it
will work when you need it.  I like to run my migration when I write it with `rake db:migrate` and
then immediately re-run it with `rake db:migrate:redo` to verify the rollback.  And, now that you know how to
reverse "irreversible" migrations, you can save the exception for times when the migration truly cannot be reversed.
