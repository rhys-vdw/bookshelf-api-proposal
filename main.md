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

// Check for presence of `idAttribute` to determine whether a model was
// retrieved from the database.
Users.isNew({id: null, name: 'Samantha'}) // -> true
Users.isNew({id: 5, name: 'Georgia'}) // -> false

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

// Get all science fiction titles published in the last year, including their
// authors.
Books
  .where({genre: 'sci-fi'})
  .where('publication_date', '>', moment().subtract(1, 'year'))
  .withRelated('author')
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

Mapper.table('users') === Mapper;
// -> false, `table` returned a modified copy.

const Users = Mapper.table('users');
Users === Users.table('users');
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

Admins.save([{name: 'Jane'}, {name: 'Mary'}]).then(admins =>
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

Mappers can be registered and retrieved from the mapper registry. String
identifiers are used stored identify mappers, helping break node dependency
loops.

```js
const Mapper = bookshelf('Mapper');
const People = Mapper.table('people').idAttribute('person_id');

bookshelf.registerMapper('Person', People);
```

Or, more simply:

```js
bookshelf.initMapper('People', {
  table: 'people'
  idAttribute: 'person_id'
});
```

We can also extend a Mapper to add or override methods.

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
    this.table('people');
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
// SQL: select people.* from people where country = 'usa';
//
bookshelf('Americans').adults().fetch().then(americans =>
// SQL: select people.* from people where country = 'usa' and date_of_birth <= [21 years ago];

bookshelf('Australians').adults().fetch().then(americans =>
// SQL: select people.* from people where country = 'aus' and date_of_birth <= [18 years ago];
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
Mapper.patch(records, {is_a_record: true}).then(patched =>

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
  //   update shopping_list set name = 'Wading Pool', qty = 1, unit = 'each' where id = 1;
  //   update shopping_list set name = 'Vodka', qty = 100, unit = 'liter' where id = 2;
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
{hasMany, belongsTo, belongsToAndHasMany} = Relations;
```

#### Definition

Relations are added to a `Mapper` via the `.relations()` setter.

```js
Staff = bookshelf('Mapper').table('staff').relations({
  department: belongsTo('Department'),
  teamMates: belongsToAndHasMany('Staff').through('ProjectMemberships'),
  projects: belongsToAndHasMany('Project').through('ProjectMemberships'),
  ownedProjects: hasMany('Project', {theirRef: 'owner_id'}),
  boss: belongsTo('Staff', {myRef: 'superior_id'})
});

bookshelf.registerMapper('Staff', Staff);

// Or like this:

bookshelf.initMapper('Staff', {
  table: 'staff',
  relations: {
    department: belongsTo('Department'),
    teamMates: belongsToAndHasMany('Staff').through('ProjectMemberships'),
    projects: belongsToAndHasMany('Project').through('ProjectMemberships'),
    ownedProjects: hasMany('Project', {theirRef: 'owner_id'}),
    boss: belongsTo('Staff', {myRef: 'superior_id'})
  }
});
```

#### Loading related data

Relations provide an interface to generate Mappers that can access and create
matching records.

##### Fetching and persisting relations

```js
.related(record, relationName)
```

Calling `.related` will return a `Mapper` configured to create and modify records
pertaining to its specific relation.

```js
const Staff = bookshelf('Staff');

const john = {id: 5, name: 'John', boss_id: 3};
const sarah = {id: 3, name: 'Sarah', boss_id: null};

Staff.related(john, 'projects').fetch().then(projects =>
// SQL: select projects.*
//      from projects
//      inner join projects_staff on projects.id = projects_staff.project_id
//      where projects_staff.staff_id = 5;
// -> [
//      {id: 6, name: 'Install Node.js',  owner_id: 3},
//      {id: 7, name: 'Learn JavaScript', owner_id: 5}
//    ]

SarahsProjects = Staff.related(sarah, 'ownedProjects');

// Create a new projects.
SarahsProjects.save({name: 'Bookshelf.js project'}).then(saved =>
// SQL: insert into projects (name, owner_id) values ('Bookshelf.js project', 3);
// -> {id: 8, name: 'Bookshelf...', owner_id: 3}

SarahsProjects.fetch().then(sarahsProjects =>
// SQL: select project.* from projects where owner_id = 3;
// -> [{id: 8, name: 'Bookshelf...', owner_id: 3}, {id: 6, name: 'Install...', owner_id: 3}]
```

It's also possible to get relations for multiple records at the same time:

```js
// Get all immediate bosses of members of a project with ID 8.
Project.related({id: 8}, 'members').fetch().then(members => {

  // Check result.
  assert.deepEqual(members, [
    {id: 1, name: 'Peter', boss_id: 3},
    {id: 5, name: 'John', boss_id: 3},
    {id: 4, name: 'Gavin', boss_id: 1}
  ]);

  return Staff.related(members, 'boss').fetch()

}).then(bosses => {

  // Check result.
  assert.deepEqual(bosses, [
    {id: 3, name: 'Sarah', boss_id: null},
    {id: 1, name: 'Peter', boss_id: 3}
  ]);

});

// The above might be written more compactly as:
Project.related(8, 'members.bosses').fetch().then(bosses =>
```

##### Eager loading 

Eager loading allows loading a record with its relations attached.

Methods used to control eager loading are:

###### `.load(target, relations)`

Load relations onto an existing record `target`. Returns a promise resolving
to the extended record.

```js
Projects.fetch(8)
.then(project => Projects.load(project, 'members'))
.then(project =>
  assert.deepEqual(project, {
    id: 8,
    name: 'Bookshelf project',
    members: [
      {id: 1, name: 'Peter', boss_id: 3},
      {id: 5, name: 'John', boss_id: 3},
      {id: 4, name: 'Gavin', boss_id: 1}
    ]
  });
)
```

###### `withRelated(relations)`

Sets an option on the Mapper to always fetch the given relations when fetching
records. Returns records extended as if `.load` had been called on them.

```js
Project.withRelated(['members.boss', 'owner.boss']).fetch(8).then(project =>
  assert.deepEqual(project, {
    id: 8,
    name: 'Bookshelf.js project',
    members: [
      {id: 1, name: 'Peter', boss_id: 3, boss: {id: 3, name: 'Sarah'}},
      {id: 5, name: 'John',  boss_id: 3, boss: {id: 3, name: 'Sarah'}},
      {id: 4, name: 'Gavin', boss_id: 1, boss: {id: 1, name: 'John' }}
    ]
    owner: {id: 3, name: 'Sarah', boss_id: null, boss: null}
  });
);
```

###### Recursive relationships

```js
// Fetch staff member Gavin with up to next three levels of bosses.
Staff.withRelated('boss^3').fetch(4).then(gavin =>
  assert.deepEqual(gavin, {
    id: 4,
    name: 'Gavin',
    boss_id: 1,
    boss: {
      id: 1,
      name: 'John',
      boss_id: 3,
      boss: {
        id: 3,
        name: 'Sarah',
        boss_id: null,
        boss: null
      }
    }
  }) 
);
```

###### Relation initializer

You can rescope relations with a callback.

```js
bookshelf
.initMapper('Review', { table: 'reviews' });
.inheritMapper('Accounts', {
  initialize() { return {
    table: 'accounts',
    relations: {
      reviews: hasMany('Review')
    }
  }},
  favourites() {
    return this.where('stars', '>', 4);
  }
})

const myAccount = {id: 2, name: 'Rhys'};

const MyFavourites = Accounts.related(myAccount, 'reviews', (Reviews) =>
  Reviews.where('stars', '>', 4)
);

// or (map arguments to mutator methods)

const MyFavourites = Accounts.related(myAccount, 'reviews', {
  'where' ['stars', '>', 4]
);

// or (array of scope methods to be called without arguments).

const MyFavourites = Accounts.related(myAccount, 'reviews', ['favourites']);
```

###### Relation DSL

```js
.withRelated(relations, [initializer]);
.withRelationTree(relationTree);
```

Relations can be any of the following values:

 - `true` - include all relations unmodified.
 - `string` - A relation name, or a description of nested relations in the
              simple DSL.
 - `Object` - A hash of relation DSL keys with initializers as values.
 - `RelationTree` - Normalized representation of the relation request.
 - `Array` - An array of any of the above applied additively to request multiple
             relations.

All of the above can be compiled into a `RelationTree` using `normalize`, which
is Bookshelf's internal representation.

As a user, it's not important to understand how `RelationTree` works as a user,
just how to supply the arguments. This is essentially the same as the current
API, but the callbacks now apply to the `Mapper` object rather than the underlying
`QueryBuilder` (This can still be access via `Mapper#query()`).

Examples of relations:

**relations: simple string**

```js
Staff.withRelated('department').fetch(5).then(staff =>
Staff.withRelated(['department', 'projects']).fetch(5).then(staff =>

// relation tree:
assert.deepEqual(
  Relations.normalize('department'),
  {department: {}}
);

assert.deepEqual(
  Relations.normalize(['department', 'projects']),
  {department: {}, projects: {}}
);
```

**relations: nested string**

```js
Staff.withRelated('projects.clients').all([5, 4, 1]).fetch().then(staff =>

// relation tree:
assert.deepEqual(
  Relations.normalize('projects.clients'),
  {
    projects: {
     nested: {clients: {}}
    }
  }
);
```

**relations: recursive relations**
```js
Staff.load(staffMember, 'boss^').then(staffMember
Staff.load(staffMember, 'boss^8').then(staffMember


const twoAboveTree = Relations.normalize('boss^');

// relation tree:
assert.deepEqual(twoAboveTree, {
  boss: {
    nested: {
      boss: { recursions: 1 }
    }
  }
});

const tenAboveTree = Relations.normalize('boss^10');

// relation tree:
assert.deepEqual(tenAboveTree, {
  boss: {
    nested: {
      boss: { recursions: 10 }
    }
  }
});

// Normalize extends recursive relations automatically.
const nestedBoss = Relations.normalize(tenAboveTree.boss.nested);
assert.deepEqual(nestedBoss, {
  boss: {
    recursions: 10
    nested: {
      boss: { recursions: 9 }
    }
  }
});
```

**relations: true**

```js
Staff.withRelated(true).fetchOne(5).then(staffMember =>

// True is expended internally to:
// this.getOption('relations').keys() ->
relations = ['department', 'teamMates', 'projects', 'ownedProjects', 'boss']

// relation tree:
assert.deepEqual(
  Relations.normalize(relations),
  { departments: {}, teamMates: {}, projects: {}, ownedProjects: {}, boss: {} }
);
```

**relations: intializer**

Supply an intializer to modify the Mapper returned by the relation.

```js
Staff.withRelated('teamMates', TeamMates =>
  TeamMates.whereNull('boss_id').where('title', 'programmer')
).fetch(staffMember);

// or, equivalently:

Staff.withRelated('teamMates', {
  whereNull: 'boss_id'
  where: ['title', 'programmer']
}).fetch(staffMember);


// relation tree:
assert.deepEqual(
  Relations.normalize(relations),
  {teamMates: {initializer: Function}}
);
```

The initializer can also be an array of scopes:

```js
bookshelf.extendMapper('Staff', {
  initialize() { return {
    table: 'staff',
    relations: {
      teamMates: hasMany('Staff').through('Project')
    }
  }},
  fullTime() {
    return this.query(query =>
      query.join('contracts', 'contracts.id', this.prefixColumn('contract_id'));
    );
  }
})

// Get all full time staff who are in a team with either bob or james.
Staff.withRelated('teamMates', ['fullTime']).all(bob, james).fetch();
```

**relations: Object**

Similar to initializer array (see above), but takes arguments to be passed to setters.

```js
Author.withRelated({
  comments: ['fromLastWeek']}
  articles: ['orderByAscending', 'created_at']
).fetchAll();
```

**relations: Aliasing relations**

Sometime you might want to do this:

```js
Author.withRelated('articles as favouriteArticle', [
  'one',
  {orderByAscending: 'popularity'}
]);
```

You can even use this to skip relations:

```js
// Get all record labels that have released a Black Sabbath album. Note that
// these are nested directly under the `band` record.
Band
  .where(name: 'Black Sabbath')
  .withRelated('(albums.recordLabel):recordLabels')
  .fetchOne()
  .then(band =>


// Get Gavin with a nested reference to the head of the company.
Staff
  .where(name: 'Gavin')
  .withRelated('(boss^Infinity):ceo')

// Let's see how many cars Gavin's boss owns.
Staff.related(gavin, '(boss^Infinity):ceo.cars').count().then(carCount =>

// Don't know about the syntax, but might as well consider every conceivable use
// case while we're here.

// Get `Black Sabbath` instance with a nested list of drummers who played with
// Ozzy Osbourne.
Bands
  .where(name: 'Black Sabbath')
  .withRelated({
    '(albums.members):drummers': {where: {role: 'drummer'}}
    'drummers::albums': {withSinger: 'Ozzy Osbourne'}
  })
  .fetchOne()
```



