---
layout: post
title: "Caveat when using an ActiveRecord association that use database view underneath"
description: ""
date: 2016-02-29
tags: []
comments: true
---

I recently utilized database views to cut down the number of rows of a table where 2/3 of its data are just an aggregate of the remaining 1/3. The speed bump was noticeable, but I struggled to make it work with the existing query that eagerloads a bunch of associations, and orders by one of the association column.

## Modeling the scenario

```ruby
class Author
  has_many :books
end

class Book
  has_many :ratings
  has_one :statistics
end

class Rating
  belongs_to :book
end

class Statistics
  self.table_name = 'statistics'

  belongs_to :book
end

```

where `Statistics` is a database view summarizing data about `Book`.

Suppose I want to retrieve the list of books of authors that have 'Kutt' as their surname. I also want to display the average rating of each book beside its name so I eagerloaded its corresponding statistics:

```ruby

authors = Author.includes(books: :statistics).where(last_name: 'Kutt').order('books.name ASC')

#=> #<ActiveRecord::Relation [#<Author id: 1, last_name: "Kutt">]>

```

## Beginning of head scratching

Inspecting the statistics of any book returned by the query returns `nil`:

```ruby
book = authors.first.books.first

#=> #<Book id: 1, name: "Cushy Tower", author_id: 1>

book.statistics

#=> nil
```

No additional database call was made so I assumed something was wrong with the book-statistics relationship. Next, I tried calling the statistics of an explicitly loaded book:

```ruby
book = Book.includes(:statistics).first
book.statistics

#=> #<Statistics book_id: 1, average_rating: #<BigDecimal:7f8bce239f40,'0.4E1',9(27)>>

```

It seems statistics somehow just got lost in the first query.

I retraced my steps and inspected the generated SQL of the two queries:

```ruby
Author.includes(books: :statistics).where(last_name: 'Kutt').order('statistics.average_rating DESC')

#=> SELECT "authors"."id" AS t0_r0, "authors"."name" AS t0_r1, "books"."id" AS t1_r0, "books"."name" AS t1_r1, "books"."author_id" AS t1_r2, "statistics"."book_id" AS t2_r0, "statistics"."average_rating" AS t2_r1 FROM "authors" LEFT OUTER JOIN "books" ON "books"."author_id" = "authors"."id" LEFT OUTER JOIN "statistics" ON "statistics"."book_id" = "books"."id" WHERE "authors"."last_name" = $1  ORDER BY statistics.average_rating DESC  [["last_name", "Kutt"]]
```

```ruby
Book.includes(:statistics).first

#=> Book Load (0.6ms)  SELECT  "books".* FROM "books"  ORDER BY "books"."id" ASC LIMIT 1
#=> Statistics Load (0.7ms)  SELECT "statistics".* FROM "statistics" WHERE "statistics"."book_id" IN (1)
```

## Discovering `includes` behaviour

`includes` can behave in two ways depending on the query's form. For simple queries where the loading of the first table (books) is independent from any of the included association's value, `includes` does a `preload` where the tables are queried separately.

If the rows retrieved from the first table is dependent on the second table (in the first query's case, the books are sorted by `statistics.average_rating`), `includes` performs a single query using a `LEFT JOIN`.

## Diving into the code path

I executed the generated SQL of the first query and found that the expected values of statistics are being pulled properly. It appears ActiveRecord can't assign the raw values to an object so I decided to follow ActiveRecord's path to instantianting an association object.

## Realizing ActiveRecord's agnosticism to views and tables

An ActiveRecord class has properties that are set through convention (i.e tables have `id` as their default primary key, the database table name is the plural form of the class name, etc). Views are stored queries that do not have an actual data. So it was a bit surprising when I found that assignment to association was looking for a primary key:

```ruby
  # lib/active_record/associations/join_dependency.rb
  def construct(ar_parent, ...)
    ...
    # node = statistics at this point
    key = aliases.column_alias(node, node.primary_key)
    id = row[key]
    if id.nil?
      nil_association = ar_parent.association(node.reflection.name)
      # statistics is established as nil so it won't get queried again
      nil_association.loaded!
      next
    end
    ...
  end
```

Not wanting to fight convention, I just set the primary key of the class to the most practical column:

```ruby
class Statistics
  self.table_name = 'statistics'

  # reusing book_id since there's a one-to-one relationship
  # between book and statistics
  self.primary_key = 'book_id'

  belongs_to :book
end
```

