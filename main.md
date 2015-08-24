# Bookshelf.js Mapper API Proposal

## Overview

A proposal to remove the concepts of `Model` and `Collection` from Bookshelf and
replace them with raw objects and arrays. Model configuration and query building
is handled by the `Mapper` object.

The design aims to provide the following:

 - A highly flexible/customizable composable interface to the database layer.
 - Readable and intuitive fluent interface.
 - Decoupling of connection (`bookshelf` instance) and domain model.
 - Simplified function contracts; fewer arguments.
 - Query 'scope' support.
 - Composite ID support.
 - Better support for more relation types.
 - Improved plugin support, considering:
   - Custom record models.
   - Parsing/formatting.
   - Custom relation types.
   - Virtual attributes

## Initialization

Initialization remains unchanged.

```js
// file: bookshelf-instance.js

import Bookshelf from 'bookshelf';
import Knex from 'knex';

const knex = Knex({
  client: 'mysql',
  connection: {
    host     : '127.0.0.1',
    user     : 'your_database_user',
    password : 'your_database_password',
    database : 'myapp_test',
    charset  : 'utf8'
  }
});

exports default Bookshelf(knex);
```


## Mapper Object

The Mapper is designed to be highly flexible and extensible object. It targets
a specific table and has knowledge of relations and primary keys. It is
immutable&em;any method that would mutate its state instead returns a modified
copy.

You use the Mapper as a specializable interface to your data. It handles all
query building and data persistence.

### Basic options

There are two required settings for a `Mapper`.

**`table()`**
Set the name of the table targeted by this Mapper.

**`idAttribute()`**
Name of column that acts as the primary key for this table. If an array of
columns is supplied it is a composite key.

```js
// Get mapper instance from bookshelf.
const Mapper = bookshelf('Mapper');

// Create a specialization of the mapper that targets the 'users' table.
Users = Mapper.table('users').idAttribute('id');

Users.fetch().then(users =>
// -> select users.* from users

Users.fetch([10, 25]).then(users =>
// -> select users.* from users where users.id in (10, 25)

// Specifying `idAttribute` can't change the immutable `Users` instance.
// Here `identify` is called on a new mapper with a different `idAttribute`
// setting.
Users.idAttribute('user_id').identify([
  {user_id: 25, name: 'Mary'},
  {user_id: 8, name: 'Peter'}
]);
// -> [25, 8]

// The Mapper uses its `idAttribute` setting to find primary keys.
// `identify` is mainly for internal use.
Users.identify({id: 10, name: 'John'});
// -> 10

// Works with composite keys.
const Membership = Mapper.table('groups_users').idAttribute(['group_id', 'user_id']);
Membership.identify([
  {group_id: 2, user_id: 2, role: 'owner'},
  {group_id: 2, user_id: 5, role: 'member'}
])
// -> [[2, 2], [2, 5]]
```

### Immutability

Mappers are chainable in a way that is familiar to Bookshelf users.

```js
import moment from 'moment';

Books = bookshelf('Books');

Books
  .where({genre: 'sci-fi'})
  .where('publication_date', '>', moment().subtract(1, 'day'))
  .withRelated(['author'])
  .fetch()
  .then(books => // ...
```

The major difference is that the chainable instances do not modify themselves.
Each call to `where` and `withRelated` returns a new, reusable instance of the
original book mapper.

A Mapper contains two types of state: the underlying knex QueryBuilder
instance, which can be modified by the following methods:

 - `query()`
 - `where()`
 - `whereIn()`
 - etc.

Other mutations occur when setting options on the Mapper. For example:

 - `table()`
 - `idAttribute()`
 - `all()`
 - `one()`
 - `withRelated()`
 - `defaultAttributes()`
 - `require()`
 - `columns()`
 - etc.

Any of these has the potential to modify the mapper instance. If this happens
a new instance will be returned.

These chainable setters replace the `options` argument originally adopted from
Backbone. 

```js
const Mapper = bookshelf('Mapper');

Mapper.tableName('users') === Mapper;
// -> false, `tableName` returned a modified copy.

const Users = Mapper.tableName('users');
Users === Users.tableName('users');
// -> true, nothing changed.
```

This enables building mappers that support your domain model.

```js
const Users = bookshelf('Mapper')
  .table('users')
  .defaultAttributes({is_admin: false});

const Admins = Users.where('is_admin', true).defaultAttributes({is_admin: true});
// or
const Admins = Users.whereDefault('id_admin', true);

Users.fetch().then(users =>
// SQL: select * from users;

Admins.fetch().then(admins =>
// SQL: select * from users where is_admin = true;

Users.save({name: 'John'}).then(user =>
// SQL: insert into users (name, is_admin) values ('John', false);
// -> {id: 4, name: 'John'}

Admins.save([{name: 'Jane'}, {name: 'Mary').then(admins =>
// SQL: insert into users (name, is_admin) values ('Jane', true), ('Mary', true);
// -> [{id: 5, name: 'Jane', is_admin: true}, {id: 6, name: 'Mary', is_admin: true}]

// We can check if John, {id: 4}, who was inserted earlier, is an admin.
Admins.require().fetchOne(4)
  .then(john => console.log('John is an admin!'))
  .catch(Bookshelf.NotFoundError, error => console.error('John is not an admin!'));
// SQL: select users.* from users where id = 4 and is_admin = true;
// throws an error because we set `require`.
```

### Mapper creation and Mapper Registry

Usually we want to define a set of mappers before our program runs and then
reference them in domain logic.

Mappers can be registered and retrieved from the Mapper registry. String
identifiers are used stored identify Mappers, helping break node dependency
loops.

```js
const Mapper = bookshelf('Mapper');
const People = Mapper.table('people');

bookshelf.registerMapper('Person', People);
```

Or, more simply:

```js
bookshelf.initMapper('People', {
  table: 'people'
  idAttribute: 'person_id'
});
```

We can also extend a Mapper to add or override methods. This can be used
like ActiveRecord's scopes.

```js
bookshelf.inheritMapper('Posts', {
  initialize() {
    this.table('post').idAttribute('post_id');
  },

  inPeriod(from, to) {
    return this.query('whereBetween', 'created_at', from, to);
  }

  fromLastWeek() {
    return this.inPeriod(moment(), moment().subtract(1, 'week');
  }
});

bookshelf('Posts').fromLastWeek().fetch().then(posts =>
// select posts.* from posts where created_at between [now] and [last week];
```

You can also set "options" on the Mapper. These can be read later by other
functions. For example calling `.withRelated(relation)` before `.fetch()`.

```js
bookshelf.inheritMapper('People', {
  initialize() {
    this.tableName('people');
  }

  adultAge(adultAge) {
    return this.setOption('adultAge', adultAge);
  }

  adults() {
    const adultAge = this.getOption('adultAge');
    const maxDateOfBirth = moment().subtract(adultAge, 'years');
    return this.query('date_of_birth', '<=', maxDateOfBirth);
  }
});

bookshelf.initMapper('Americans', {
  where: {country: 'usa'}
  adultAge: 21
});

bookshelf.initMapper('Australians', {
  where: {country: 'aus'}
  adultAge: 18
});

bookshelf('Americans').fetch().then(americans =>
// SQL: select people.* from people where country = 'usa' and date_of_birth <= [21 years ago]

bookshelf('Australians').fetch().then(americans =>
// SQL: select people.* from people where country = 'aus' and date_of_birth <= [18 years ago]
```


### Fetching records

The mapper handles all fetching.

```js
bookshelf('Users');

Users.all().fetch().then(users =>
Users.fetchAll().then(users =>
Users.fetch().then(users =>
// SQL: select users.* from users;
// -> [{id: 1, ...}, ...]

Users.one().fetch().then(user =>
Users.fetchOne().then(user =>
// SQL: select users.* from users limit 1;
// -> {id: 1, ...}

Users.where('id', 5).one().fetch().then(user =>
Users.one(5).fetch().then(user =>
Users.fetchOne(5).then(user =>
Users.fetch(5).then(user =>
// SQL: select users.* from users where id = 5 limit 1;
// -> {id: 5, ...}

Users.where('logged_in', true).all().fetch().then(users =>
Users.where('logged_in', true).fetchAll().fetch(users =>
// SQL: select users.* from users where logged_in = true;
// -> [{id: ?, ...}, ...]

Users.all([20, 3, 5]).fetch(users =>
Users.fetchAll([20, 3, 5]).then(users =>
Users.fetch([20, 3, 5]).then(users =>
// SQL: select users.* from users where id in (20, 3, 5);
// -> [?, ?, ?]
```

### Persisting state

Each Mapper provides an interface for doing bulk insertion, patch and update operations.

Note that the Mapper layer alone does not do any dirty checking. This can be achieved
with the Model plugin.

```js
// inserts all objects into the database.
Mapper.insert(records).then(inserted =>

// Updates all supplied records. This is not a bulk operation, it will do as
// many `update`s as there are records. If any fail an `Mapper.isNew` test the
// promise will be rejected.
Mapper.update(records).then(updated =>

// Update a set of records with the same data.
Mapper.patch(records, is_a_record: true).then(patched =>

// updates or inserts records based on result of `Mapper.isNew(record)`.
Mapper.save(records).then(saved =>

// deletes records from the database.
Mapper.delete(records).then(deleted =>
}
```

```js
ShoppingList = bookshelf.extendMapper({

  initialize() {
    this.table('shopping_list_items')
  }

  purchasedItems() {
    return this.where('is_purchased', true);
  }

  markPurchased(records) {
    return this.patch(records, {is_purchased: true});
  }
});

const createListPromise = ShoppingList.save(
  {name: 'Watermelon', qty: 1, unit: 'each'},
  {name: 'Vodka', qty: 2, unit: 'liter'},
)
// SQL: insert into shopping_list (name, qty, unit) values ('Watermelon', ...), ('Vodka', ...)
// -> [{id: 1, name: 'Watermelon', ...}, {id: 2, name: 'Vodka', ...}]

createListPromise.tap(items =>
  items[0].name = 'Wading pool';
  items[1].qty = 100;

  ShoppingList.save(items);
  // SQL:
  //   update shopping_list set name = 'Wading Pool', qty = 1, unit = each where id = 1;
  //   update shopping_list set name = 'Vodka', ...
  //
  // No bulk updates, nor dirty checking. These can be achieved with rich models.
})
.tap(items => userIterface.presentItems(items))
.tap(items => {
  
  const purchasedItems = items.filter(item =>
    userInterface.isChecked(item)
  );

  return ShoppingList.all(purchasedItems).markPurchased();
  // SQL: update shopping_list set is_purchased = true where id in (1, 2);

}).then(purchased => 

  if (getSetting('clearOnPurchase')) {
    return ShoppingList.all(purchasedItems).destroy();
    // SQL: delete from shopping_list where id in (1, 2);

    // Or, if preferable: 

    return ShoppingList.purchasedItems().destroy();
    // SQL: delete from shopping_list where is_purchased = true;
);
```

### Relations

Support for the pre-existing relation types:

 - `hasOne`
 - `belongsTo`
 - `hasMany`
 - `belongsToAndHasMany` (was `belongsTo`)
 - `morphOne`
 - `morphMany`
 - and variants of above with `.through`

These fields now exist on the `bookshelf.Relations` object.

```js
import Bookshelf, {Relations} from bookshelf;

// You can just use the relations you need.
{hasMany, belongsTo} = Relations;
```

#### Definition

```js
bookshelf.initMapper('Staff', {
  table: 'staff',
  relations: {
    projects: hasMany('Project', {otherReferencingKey: 'creator_id'}),
    workplace: belongsTo('Building', {selfReferencingKey: 'work_adress_id'}),
    teamMates: belongsToAndHasMany('Staff').through('ProjectMemberships')
  }
});
```
