Search API
----------

The Search API allows entities to be search-able through an auto-generated index
of words. You can make any table "index-able" by attaching the `SearchableBehavior`
to it.


### Searchable Behavior

This [behavior][cake_doc_behaviors] is responsible
of index each entity under your tables, it also allows you to search any of those
entities by using human-friendly search criteria.


#### Using this Behavior

You must indicate which fields can be indexed when attaching this behavior
to your tables. For example, when attaching this behavior to `Users` table:

```php
$this->addBehavior('Search.Searchable', [
    'fields' => ['username', 'email']
]);
```

In the example above, this behavior will look for words to index in user's
"username" and user's "email" properties.

If you need a really special selection of words for each entity is being indexed,
then you can set the `fields` option as a callable which should return a list of
words for the given entity. For example:

```php
$this->addBehavior('Search.Searchable', [
    'fields' => function ($user) {
        return "{$user->name} {$user->email}";
    }
]);
```

You can return either, a plain text of space-separated words, or an array list
of words:

```php
$this->addBehavior('Search.Searchable', [
    'fields' => function ($user) {
        return [
            'word 1',
            'word 2',
            'word 3',
        ];
    }
]);
```

This behaviors will apply a series of filters (converts to lowercase, remove
line breaks, etc) to the resulting word list, so you should simply return a RAW
string of words and let this behavior do the rest of the job.


##### Banned Words

You can use the `bannedWords` option to tell which words should not be indexed
by this behavior. For example:

```php
$this->addBehavior('Search.Searchable', [
    'bannedWords' => ['of', 'the', 'and']
]);
```

If you need to ban a really specific list of words you can set `bannedWords` option
as a callable method that should return true or false to tell if a words should be
indexed or not. For example:

```php
$this->addBehavior('Search.Searchable', [
    'bannedWords' => function ($word) {
        return strlen($word) > 3;
    }
]);
```

- Returning TRUE indicates that the word is safe for indexing (not banned).
- Returning FALSE indicates that the word should NOT be indexed (banned).

In the example, above any word of 4 or more characters will be indexed
(e.g. "home", "name", "quickapps", etc). Any word of 3 or less characters will
be banned (e.g. "and", "or", "the").


#### Searching Entities

When attaching this behavior, every entity under your table gets a list of
indexed words. The idea is you can use this list of words to locate any entity
based on a customized search-criteria. A search-criteria looks as follow:

    "this phrase" OR -"not this one" AND this

Use wildcard searches to broaden results; asterisk (`*`) matches any one or
more characters, exclamation mark (`!`) matches any single character:

    "thisrase" OR -"not th!! one" AND thi!

Anything containing space (" ") characters must be wrapper between quotation
marks:

    "this phrase" special_operator:"[100 to 500]" -word -"more words" -word_1 word_2

The search criteria above will be treated as it were composed by the
following parts:

    [
        this phrase,
        special_operator:[100 to 500],
        -word,
        -more words,
        -word_1,
        word_2,
    ]

Search criteria allows you to perform complex search conditions in a human-readable
way. Allows you, for example, create user-friendly search-forms, or create some
RSS feed just by creating a friendly URL using a search-criteria.
e.g.: `http://example.com/rss/category:art date:>2014-01-01`

You must use the `search()` method to scope any query using a search-criteria.
For example, in one controller using `Users` model:

```php
$criteria = '"this phrase" OR -"not this one" AND this';
$query = $this->Users->find();
$query = $this->Users->search($criteria, $query);
```

The above will alter the given $query object according to the given criteria.
The second argument (query object) is optional, if not provided this Behavior
automatically generates a find-query for you. Previous example and the one
below are equivalent:

```php
$criteria = '"this phrase" OR -"not this one" AND this';
$query = $this->Users->search($criteria);
```


##### Creating Operators

An `Operator` is a search-criteria command which allows you to perform very
specific filter conditions over your queries. An operator **has two parts**,
a `name` and its `arguments`, both parts must be separated using the `:`
symbol e.g.:

    // operator name is: "author"
    // operator arguments are: ">2014-03-01"
    date:>2014-03-01

NOTE: Operators names are treated as **lowercase_and_underscored**, so
`AuthorName`, `AUTHOR_NAME` or `AuThoR_naMe` are all treated as: `author_name`.

You can define custom operators for your table by using the
`addSearchOperator()` method. For example, you might need create a custom
operator `author` which allows you to search a `Node` entity by `author name`.
A search-criteria using this operator may looks as follow:

    // get all nodes containing `this phrase` and created by `JohnLocke`
    "this phrase" author:JohnLocke

You must define in your Table an operator method and register it into this
behavior under the `author` name, a full working example may look as follow:

```php
class Nodes extends Table {
    public function initialize(array $config) {
        // attach the behavior
        $this->addBehavior('Search.Searchable');

        // register a new operator for handling `author:<author_name>` expressions
        $this->addSearchOperator('author', 'scopeAuthor');
    }

    public function scopeAuthor($query, $value, $negate, $orAnd) {
        // $query:
        //     The query object to alter
        // $value:
        //     The value after `author:`. e.g.: `JohnLocke`
        // $negate:
        //     TRUE if user has negated this command. e.g.: `-author:JohnLocke`.
        //     FALSE otherwise.
        // $orAnd:
        //     or|and|false Indicates the type of condition. e.g.: `OR author:JohnLocke`
        //     will set $orAnd to `or`. But, `AND author:JohnLocke` will set $orAnd to `and`.
        //     By default is set to FALSE. This allows you to use
        //     Query::andWhere() and Query::orWhere() methods.
    }
}
```

##### Fallback Operators

When an operator is detected in the given search criteria but no operator
callable was defined using `addSearchOperator()`, then
`SearchableBehavior.operator<OperatorName>` will be fired, so other plugins
may respond to any undefined operator. For example, given the search criteria
below, lets suppose `date` operator **was not defined** early:

    "this phrase" author:JohnLocke date:[2013-06-06..2014-06-06]

The `SearchableBehavior.operatorDate` event will be fired. A plugin may
respond to this call by implementing this event:

```php
// ...

public function implementedEvents() {
    return [
        'SearchableBehavior.operatorDate' => 'operatorDate',
    ];
}

// ...

public function operatorDate($event, $query, $value, $negate, $orAnd) {
    // alter $query object and return it
    return $query;
}

// ...
```

IMPORTANT:

- Event handler method should always return the modified $query object.
- The event's context, that is `$event->subject`, is the table instance that
  fired the event.


### Recommended Reading

[Behaviors][cake_doc_behaviors]
[Events System][events_system]

[cake_doc_behaviors]: http://book.cakephp.org/3.0/en/orm/behaviors.html
[events_system]: 01_Events_System.md