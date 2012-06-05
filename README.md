JetSet
======

Use JetSet to easily add "settings" or configuration data (i.e. booleans, strings, integers, etc) to your ActiveRecord models that you might otherwise store in extra database columns.  JetSet adds a key/value store to your existing records, giving you the ability to design your schema on-the-fly for those data or atttributes which are essential to store but don't need `find`-ability via SQL `WHERE` statements.

Quick, Show Me!
---------------

```ruby
class Post < ActiveRecord::Base
  include JetSet

  # Create a JetSet store called `settings` and define the schema
  jetset :settings do |s|
    s.boolean  :allow_comments, :default => true
    s.datetime :published_at,   :default => lambda { |post| post.created_at }
    s.string   :author_name
    s.text     :body
    s.integer  :number_in_series
    s.float    :magic_number
  end 
end

# Create a new post and observe the default values for our settings
post = Post.new
post.settings.allow_comments
# => true
post.settings.author_name
# => nil

# Set a new value, read it, and then see it doesn't stick without a save
post.settings.allow_comments = false
# => false
post.settings.allow_comments
# => false
post.reload
post.settings.allow_comments
# => true

# Set a new value, save the parent record, and see that it sticks
post.settings.allow_comments
# => true
post.settings.allow_comments = false
# => false
post.save
post.settings.allow_comments
# => false
```

Defining a JetSet Store
-----------------------

Every ActiveRecord instance can have 1 or more JetSet Stores.  Think of the Store like a hash - its just set of keys with a value stored in each.  The values are typed, however, according to your definition.

**Stores are defined using the `jetset` class method:**

```ruby
jetset :options do |o|
  o.boolean, :is_awesome, :default => true
end

jetset :config do |c|
  c.string :backend, :default => 'json', :allowed => %w(json xml)
end
```

The first argument to the `jetset` method is the Store name.  It is used to access the store on your object.

**Access your store via its name, and the attributes via the key names (as methods):**

```ruby
model.options.is_awesome
# => true
model.options.is_awesome?
# => true
model.config.backend
# => 'json'
```

JetSet supports many of the same column types as ActiveRecord column definitions: `boolean`, `datetime`, `string`, `text`, `integer`, and `float`.

You pass a block to the `jetset` method to define your "schema", providing a type, default value (optional), and allowed values (optional).

### Value Types

* `boolean`
* `string`
* `integer`
* `text`
* `float`
* `datetime`

### Storage Options

JetSet can store its data in the same table as the base class (the default) or in a separate table.


#### Storing Data in the Same Table

Storing data in the same table is the default.  This is very similar to using Rails' built-in `serialize` declaration to store a Hash, but with JetSet each value also has a defined type.

```ruby
class Product < ActiveRecord::Base
  # Store the `attributes` in the `products` table in the `attributes` column.
  # (Storing in the same table is the default)
  jetset :attributes(:table => :same) do |a|
    a.string :sku
  end
end
```

Specifying same-table storage can be explicitly specified by passing `:table => :same` to the `jetset` method options hash.  The data is stored in a database column of the same name as the JetSet store - like with Rails' serialize, the column must be defined as type `text`.
    
#### Storing Data in Separate Table

Storing data in another table is a great option when you don't want to add a column to your main table.

```ruby
class Product < ActiveRecord::Base
  # Store the `settings` in the `product_settings` table in the `storage` column.
  # The `product_settings` table must also have a foreign key reference
  # back to a Product record.
  jetset :settings(:table => 'product_settings') do |s|
    s.string :access_level 
  end
end
```

This is accomplished by passing a table name to the `:table` key in the options hash.Under the covers, JetSet creates an ActiveRecord association for you.

When using separate table storage, the JetSet data will be lazy loaded from the association, which means you should eager load the association when fetching sets of rows for which you'll need the data to prevent n+1 database reads.  (TODO: JetSet can manage this for you)


You might want to use JetSet if...
----------------------------------

* Adding database columns is painful for you due to the size of your
  tables
* You just want to store more data for your models and you don't need to
  search for it or use the values in `WHERE` clauses
* You can't use the hstore Postgres extension, or you need multiple
  database compatibility (i.e. sqlite3 and/or MySQL and/or Postgres)
* The data you want to store is rarely written (writes to large blob
  columns are generally less performant)

References
----------

<http://www.mysqlperformanceblog.com/2010/01/21/when-should-you-store-serialized-objects-in-the-database/>
