# Bookshelf Proposal

#### Model and Collection
Bookshelf implements two main functionalities. An ability to generate queries from an understanding of relationships and tables (data gateway), and a way to model row data in domain objects (active records). Currently both of these responsibilities are shared by two classes: `Model` and `Collection`. 

#### Sync
This is conceptually messy and also has resulted in code duplication between the two. The internal `Sync` class currently unifies the implementation of query building (table data gateway). It also allows chaining of query methods by storing state. This means that query state and row state are interwoven in a way that is confusing and non-representative underlying data structure.

#### Collections

No more `Collection`s. Result sets are returned as plain arrays. Instead `Mapper` offers an API for bulk `save`, `insert` or `update`.

```js
// No longer doing this:
Person.collection([fred, bob]).invokeThen('save').then( // ...
Person.collection([fred, bob]).invokeThen('save', null, {method: 'insert'}).then( // ...
Person.collection([fred, bob]).invokeThen('save', {surname: 'Smith'}, {patch: true}).then( //...

// Instead:
Person = bookshelf('Person')
Person.save([fred, bob]).then( // ...
Person.insert([fred, bob]).then( // ...
Person.patch([fred, bob], {surname: 'Smith'}).then( //...
```

This offers a cleaner API, and greatly simplifies the `save` function. `save` currently has a lot of different options and flags, some of which make no sense when used together. Separating `patch` into its own method means that `save` no longer needs to support the `attrs` argument.

This means no more Lodash helpers and no intelligent `.add()` method.



### Proposal

#### Mapper

Defining table access information (`relations`, `idAttribute`, `tableName`) on `Model` is strange because models are our references to table rows. Ideally models and client code should be using a specific abstracted interface to access the database.

```js
// -- Current --

// Why are we `forge`ing a person here?
// Note that `Person.where()` calls `forge` internally.
var People = require('./person');
People.forge().where('age', '>', 5).fetchAll().then((people) => // ...

// Or this strange thing:
People.forge({id: 5}).fetch().then((person) => // ...

// -- Proposal --

// Now we create a new `Mapper` chain like this:
bookshelf('People').all().where('age', '>', 5).fetch().then(people => // ...

// Or this:
bookshelf('People').one().where('id', 5).fetch().then(person => // ...

// Or if you want to be really terse:
People = bookshelf('People')
People.fetchOne(5).then(person =>
```

`Mapper` takes on most of the current responsibilities of `Model`:

```js
// A Mapper definition:

var Person = bookshelf('Mapper').extend({

  initialize: function() {
  
    // Configure schema information.
    this.tableName('people').idAttribute('id').relations({
      home: belongsTo('House'),
      children: hasMany('Person', {otherReferencingColumn: 'parent_id'})
    });
  },
  
  // Add scopes if you want.
  adults: function() {
    return this.where('age', '>=', 18);
  },
});

bookshelf.registerMapper('Person', Person);

// Or like this.

bookshelf.extendMapper('Person', 'Mapper', {
  initialize() {
    this.tableName('people')
      .idAttribute('id')
      .relations({
        home: belongsTo('House'),
        children: hasMany('Person', 'parent_id')
      });
  }
  adults() {
    return this.all().where('age', '>=', 18);
  }
});

// Or even like this:
// Note that 'Mapper' is default, so we don't have to list it as a parent class.
bookshelf.extendMapper('Person', {
  initialize() {
    this.tableName('people')
      .idAttribute('id')
      .relations({
        home: belongsTo('House'),
        children: hasMany('Person', 'parent_id')
      });
  }
  adults() {
    return this.all().where('age', '>=', 18);
  }
});

// Or, if you prefer.
bookshelf.extendMapper('Person', {
  initialize() {
    return {
      tableName: 'people',
      idAttribute: 'id',
      relations: {
        home: belongsTo('House'),
        children: hasMany('Person', 'parent_id')
      };
    }
  }
  adults() {
    return this.all().where('age', '>=', 18);
  }
});

// If you don't need scopes, you can just add an initializer function. (It can
// return a 'mutations hash' which is applied as functions).
//
bookshelf.initMapper('Person', {
  tableName: 'people',
  idAttribute: 'id',
  relations: { home: belongsTo('House') };
});

// Or, same deal, but procedural.
bookshelf.initMapper('Person', Person =>
  Person
    .tableName('people')
    .idAttribute('id')
    .relations({ home: belongsTo('House') });
);

// Or to go totally crazy meta:
bookshelf.initMapper('Person', {
  tableName: 'people',
  idAttribute: 'id',
  relations: { home: belongsTo('House') },
  extend: { // Triggers inheritance internally.
    adults() {
      return this.all().where('age', '>=', 18);
    }
  }
});

// If we want to override an already registered model, do so like this:
bookshelf.extendMapperReplace('Model', {
  fromLastWeek() {
    return this.where('created_at', '>=', moment().subtract(1, 'week'));
  }
});
```

##### 'extending' with an initializer callback

```js
import bookshelf from './bookshelf-instance';
import Bookshelf from 'Bookshelf';
const  { belongsTo, hasMany } = Bookshelf.relations;

const People = bookshelf('Mapper').extend({
  initialize() {
    this.tableName('people')
      .idAttribute('id')
      .relations({
        house: belongsTo('House'),
        children: hasMany('People', 'parent_id');
      });
  }
  
  drinkingAge(age) {
    return this.setOption('drinkingAge', age);
  }
  
  drinkers() {
    return this.all().where('age', '>=', this.getOption('drinkingAge'));
  }
});

bookshelf.registerMapper('People', People);

// Bizarro inheritance/scoping by supplying an 'initializer'.
bookshelf.initMapper('Australians', 'People', aus =>
  aus.where('country', 'australia').drinkingAge(18)
);
bookshelf.initMapper('Americans', 'People',
  {where: {country: 'america'}, drinkingAge: 21}
);

// Or, if you prefer:
bookshelf
.registerMapper('People', People)
.initMapper('Australians', 'People', aus =>
  aus.where('country', 'australia').drinkingAge(18)
)
.initMapper('Americans', 'People',
  {where: {country: 'america'}, drinkingAge: 21}
);

Americans = bookshelf('Americans');
Australians = bookshelf('Americans');

Americans.where('sex', 'male').drinkers().query().toString();
// select users.* from users where country = 'america' and sex = 'male' and age >= 21;

Australians.drinkers().query().toString();
// select users.* from users where country = 'australia' and where age >= 18

// Hm, if `where` called `defaultAttributes` internally we could do this:
FemaleAustralians = Australians.where('sex', 'female');
FemaleAustralians.save({name: 'Jane'}, {name: 'Janette'});
// insert into people (name, sex) values ('Jane', 'female'), ('Janette', 'female');

```

Note that I've added a simple filter for `adults` above. Most methods on `Mapper` should be chainable to help build up queries (much like knex).

```js
// Example Mapper methods.
class Mapper {

  // Chainable:
  query(queryCallback)
  where(attribute, operator, value)
  setOption(option, value)
  changeOption(option, oldValue => newValue)
  withRelated(relations)
  one([id])
  all([ids])

  // Not chainable:
  fetch()
  save(records)
  update(id, record)
  update(record)
  insert(records)
  patch(ids, attributes)
  patch(records, attributes)
  load(records, relations)
  getOption(option)  // (for internal use)
}

```

Example use:

```js
// Get a person from the Smith family, and their house and daughters.
bookshelf('Person')     // returns Person Mapper instance.
  .withRelated('house')
  .withRelated('children', query => query.where('sex', 'female'))
  .one()
  .where('surname', 'Smith')
  .fetch();

```

##### Mapper chain state: options, query and client

Mapper chains have two main state objects. The `query`, and their `options` map.
Additionally they have a flag that states whether they are currently mutible or
not.

Mappers can be made mutible temporarily for bulk changes. This is not something
a user would typically do, as other helper methods are available for bulk
changes. eg. `Mapper#withMutations`, or `Bookshelf#initMapper`.

```js
import Immutable, { Iterable } from 'immutable';

class Mapper {

  // NOTE: client code never calls this constructor. It is called from within
  // the `Bookshelf` instance.
  //
  // bookshelf('MyModel').getOption('single') -> false
  //
  constructor(options = {}, query = null) {

    options = Immutable.fromJS(options).asImmutable();

    if (!options.has('client')) {
      throw new Error('Cannot create Mapper without `options.client`');
    }

    this._query = query;
    
    // This instance is entirely mutable for the duration
    // of the constructor.
    this._mutable = true;
    
    // First set defaults.
    this._options = Immutable.fromJS({
      withRelated: {},
      require: false,
      single: false,
      defaultAttributes: {},
      relations: {}
    }).asMutable();
    
    // Now allow extra mutations to be set by inheriting class. Typically
    // setting options or the query.
    //
    // NOTE: Doing something weird here. There's a small chance that one of the
    // mutations might be a call to `Mapper#extend`. If this happens the
    // returned instance will actually inherit from this object, turning this
    // constructor call into a factory method.
    //
    const mapper = this.withMutations(this.initialize);
    
    // Override those with supplied options. (This is not client facing,
    // it's for use when mapper instances clone themselves from
    // `.query` and `.setOption`).
    //
    // Calling `asImmutable()` here locks the instance.
    //
    mapper._options.merge(options).asImmutable();
    
    // Now lock it down. We return it in the off chance that `extend` was called
    // in a callback.
    return mapper.asImmutable();
  }
  
  initialize() { /* noop */ }
  
  // -- Options --
  
  getOption(option) {
  
    // Options must be initialized before they are accessed.
    if (!this._options.has(option)) {
      throw new InvalidOption(option, this);
    }
      
    // Have to ensure references are immutable. Mutable mapper chains
    // could leak.
    return Iterable.IsIterable(result)
      ? result.AsImmutable()
      : result;
  }

  getOptions(...options) {
    return _(options)
      .flatten() // Allow either (...options) or options[]
      .map(option => [option, this.getOption(option)])
      .zipObject()
      .value()
  }
  
  // Change an option on the mapper chain. If 
  setOption(option, value) {
  
    // The first time we call 'setMutability' on `_options` while mutable,
    // it will return a mutable copy. Each additional call will return the
    // same instance.
    //
    // Wrapping the value in `Immutable.fromJS` will cause all Arrays and
    // Objects to be converted into their immutable counterparts.
    //
    // Calls to `_setMutability` and `Immutable.fromJS` will often be
    // called redundantly. This is meant to ensure that the fewest possible
    // copies are constructed.
    //
    const newOptions = this._setMutability(this._options)
      .set(option, Immutable.fromJS(value));
    
    // If there was a change, return the new instance.
    return newOptions === this._options
      ? this
      : new this.constructor(newOptions, this._query);
  }
  
  changeOption(option, setter) {
    let value = this.getOption(option);
    if (Iterable.IsIterable(value)) {
      value = this._setMutability(value);
    }
    return this.setOption(option, setter(value));
  }
  
  // -- Options examples --
  
  // See relations section for more info on `withRelated`.
  withRelated(related, query = null) {
    const normalized = normalizeWithRelated(related, query);
    return this.changeOption('withRelated', (withRelated) => {
      return withRelated.mergeDeep(normalized);
    });
  }
  
  // Do these two in the `withMutations` callback to prevent an extra copy
  // being made.
  //
  // Might prefer defer these IDs until `fetch` is called and then pass them
  // through a simple hook that can be overridden. Then we can use an identity
  // map to grab cached instances if desired.
  all(ids) {
    return this.withMutations(mapper => {
      mapper.setOption('single', false);
      if (!_.isEmpty(ids)) {
        idAttribute = this.getOption('idAttribute')
        mapper.query('whereIn', idAttribute, ids);
      }
    });
  }
  
  one(id) {
    return this.mutate(g => {
      g.setOption('single', true);
      if (id != null) {
        idAttribute = this.getOption('idAttribute')
        g.where(idAttribute, id)
      }
    });
  }
  
  // -- Initialization type stuff --
 
  tableName(tableName) {
    if (_.isEmpty(arguments)) {
      return this.getOption('tableName');
    }
    return this.setOption('tableName', tableName);
  }
  
  idAttribute(idAttribute) {
    if (_.isEmpty(arguments)) {
      return this.getOption('idAttribute');
    }
    return this.setOption('idAttribute', idAttribute)
  }
  
  relations(relationName, relation) {
  
    // Getter: .relations();
    if (_.isEmpty(arguments)) {
      return this.getOption('relations');
    }
    
    // Setter: .relations({projects: hasMany('Project'), posts: hasMany('Post')});
    if (_.isObject(relation)) {
      return this.withMutation(g =>
        _.each(relation, (factory, name) => this.relations(name, factory))
      )
    }
    
    // Setter: .relations('projects', hasMany('Project'));
    return this.changeOption('relations', (relations) =>
      relations.set(relationName, relation)
    );
  }
  
  // -- Query --
  query(method, ...methodArguments) {
  
    // Ensure we have a query.
    const query = this._query || this._client.knex(this.constructor.tableName);
    
    // Support `.query()` no argument syntax.
    if (_.isEmpty(arguments)) {
      return query.clone();
    } 
    
    // If immutable we must clone the query object.
    const newQuery = this._mutable ? query : query.clone();
    
    // Support string method or callback syntax.
    if (_.isString(method)) {
      newQuery[method].apply(newQuery, methodArguments);
    } else {
      method(newQuery);
    }
    
    // Now return a chain object.
    return this._mutable
      ? this
      : new this.constructor(this._client, this._options, newQuery);
  }
  
  // -- Query example --
  
  where(filter, ...args) {
    // Handle either of these:
    // `.where(age, '>', 18)`
    // `.where({firstName: 'John', lastName: 'Smith'})`
    filter = _.isString(filter)
      ? this.constructor.attributeToColumn(filter)
      : this.constructor.getAttributes(filter);
      
    return this.query('where', filter, ...args);
  }
  
  // -- Utility --
  
  // Create a copy of this instance that is mutable.
  asMutable() {
    if (this._mutable) {
      return this;
    } else {
      const result = new this.constructor(this._options, this._query.clone());
      result._mutable = true;
      return result;
    }
  }
  
  // Lock this instance.
  asImmutable() {
    this._mutable = false;
    return this;
  }
  
  // Chain some changes that wont create extra copies.
  withMutations: (callback) {

    // Apply our callback function.
    if (_.isFunction(callback)) {

      // Deal with a mutable instance. If it was already mutable then
      // `mapper === this`.
      const wasMutable = this._mutable;

      // The mutable mapper should be updated in place for all mutations.
      const mapper = this.asMutable(); 

      // Allow config object to be returned.
      const hash = callback.bind(mapper)(mapper);
      mapper._applyMutationHash(hash)

      if (!wasMutable) {
        // Restore previous immutability.
        mapper.asImmutable();
      }

      // Return 
      return mapper;
    }

    if (!_.isPlainObject(callback) || Iterable.isIterable(callback)) {
      throw new TypeError('Expected `callback` to be of type `Function`, `Object`, or `Immutable.Iterable`');
    }

    // Now apply hash.
    return mapper._applyMutationHash(callback)
  }

  // Definitely should not be called on an immutable mapper.
  _applyMutationHash: (hash) {
    let mapper = this;
    hash.forEach((argument, method) =>
      const func = mapper[method];
      if (!_.isFunction(func)) throw new TypeError(
        `Expected ${method} to be a function, got '${func}'`
      );

      mapper = func(argument);

      if (!mapper instanceof this.constructor) throw new TypeError(
        `Expected mutation hash options to call chainable methods. Returned non-Mapper value '${mapper}'`
      );
    );
    return mapper;
  }

  // -- Helper --
   
  _setMutability(object) {
    return object[this._mutable ? 'AsMutable' : 'AsImmutable']();
  }


  // -- Extending --

  extend(methods) {
    // Create a clone of self.
    class ChildMapper extends this.constructor {
      constructor(...args) {
        super(...args);
      }
    }
    
    // Mix in the new methods.
    _.extend(ChildMapper.protoype, methods);

    // Instantiate the instance.
    return new ChildMapper(this._options, this._query.clone());
  }
}
```

#### bookshelf(mapper)
Moving the registry plugin to core is integral to the new design.

Currently `Model` is aware of its Bookshelf client (and internal knex db connection) - and can only be reassigned by setting the `transacting` option. This is less flexible than it could be. Now every `Mapper` chain must be initialized via a client object.

```js

// Using `User` directly.
bookshelf('User').save(newUser);

// Registering and reusing (helps break `require()` dependency cycles).
bookshelf('User').where('age', '>', 18).fetch().then((users) =>

// Transaction objects are richer and take `Mapper` objects similarly.
bookshelf.transation(trx =>
  trx('User').adults().fetch()
    .then(adults =>
      trx('User').patch(adults, {is_adult: true});
    )
);

StrictUser = User.require();
StrictUser.one(12).fetch().catch(NotFoundError, error =>
  console.error('User with ID 12 not found!')
);
```

This simply instantiates a new `Mapper` instance with the correct `client` attached (either a `Bookshelf` or `Transaction` instance).


```js
import MapperBase from './base/Mapper';

// Store an instantiated instance of a mapper. It doesn't matter that
// it's an instance because it's immutable. It's kind of the prototype
// pattern.
//
// Note the initializer. This is so you can do this for simple tables:
//
// bookshelf('User', 'Model', user => user.tableName('users').idAttribute('id'));
//
// The initializer is applied **after** the constructor is called.

function doRegister(name, Mapper) {
   bookshelf.registry.set(name, Mapper);
   return bookshelf;
}

function doExtendMapper(name, ParentMapper, methods) {
  if (_.isUndefined(methods)) {
    methods = ParentMapper;
    ParentMapper = null;
  }
  const parent = ensureMapper(ParentMapper);
  return doRegister(name, parent.extend(methods));
}

function doInitMapper(name, ParentMapper, initializer) {
  if (_.isUndefined(initializer)) {
    initializer = ParentMapper;
    ParentMapper = null;
  }
  const parent = ensureMapper(ParentMapper);
  return doRegister(name, ParentMapper.withMutations(initializer));
}

function assertReplacing(name, shouldExist) {
  if (shouldExist && !bookshelf.registry.has(name)) {
    throw new Error(`Cannot replace Mapper '${name}', which is not registered.`);
  }
  if (!shouldExist && bookshelf.registry.has(name)) {
    throw new Error(`Already registered Mapper '${name}'`);
  }
}

function registerMapper(name, Mapper) {
  assertReplacing(name, false);
  return doRegister(name, Mapper);
}

function registerMapperReplace(name, Mapper) {
  assertReplacing(name, true);
  return doRegister(name, Mapper);
}

// etc...


// Retrieve a previously stored mapper instance.
//
function retrieveMapper(mapper) {
  const mapper = bookshelf.registry.get(mapper);
  if (!mapper) {
    throw new Error(`Unknown Mapper: ${mapper}`)
  }
  return mapper;
}

// Gets an immutable instance of a stored mapper, or the one passed in.
function ensureMapper(mapper) {
  if (mapper instanceof MapperBase) {
    return mapper.asImmutable();
  }
  return retrieveMapper(mapper || 'Mapper');
}

const bookshelf = retrieveMapper;
bookshelf.registerMapper = registerMapper;
bookshelf.registry = new Map();

export default bookshelf;
```

#### Relations

Currently relation code is mixed into `Collection` and `Model` via `Sync`. A `Relation` instance is created by the relation factory function (`hasMany`, `belongsTo` etc.). This is then attached to the `Model` or `Collection` instance as the `relatedData` propperty. `relatedData` is referenced by `Sync` when fetching models, and by collections when `create`ing new models.

This means relation logic is interpersed throughout all classes.

##### Proposal

Using `Mapper` relations become much simpler. A `Relation` is an interface that provides some methods: `forOne()`, `forMany()` and `attachMany()`.

For instance:

```js
class HasOne {
  constructor(SelfMapper, OtherMapper, keyColumns) {
    this.Self = OtherMapper;
    this.Other = OtherMapper;
    
    // Should consider how composite keys will be treated here.
    // This can be done on a per-relation basis.
    this.selfKeyColumn = keyColumns['selfKeyColumn'];
    this.otherReferencingColumn = keyColumns['otherReferencingColumn'];
  }
  
  getSelfKey(instance) {
    return this.Self.getAttributes(instance, this.selfKeyColumn);
  }
  
  // Returns an instance of `Mapper` that will only create correctly
  // constrained models.
  forOne(client, target) {
    let targetKey = this.getSelfKey(instance);
    return client(this.Other)
      // Constrain `select`, `update` etc.
      .one().where(this.otherReferencingColumn, targetKey)
      // Set default values for `save` and `forge`.
      .defaultAttribute(this.otherReferencingColumn, targetKey);
  }
  
  // We need to specialize this for multiple targets. We don't need to
  // worry about setting default attributes for `forge`, as it doesn't
  // really make sense.
  forMany(client, targets) {
    let targetKeys = _.map(targets, this.getSelfKey, this);
    return client(this.Other)
      .all().whereIn(this.otherReferencingColumn, targetKeys);
  }
  
  // Associate retrieved models with targets. Used for eager loading.
  attachMany(targets, relationName, others) {
    let Other = this.Other;
    let Self = this.Self;
    
    let othersByKey = _(others)
      .map(Other.getAttributes, otherProto)
      .groupBy(this.otherReferencingColumn)
      .value();

  return _.each(targets, target => {
    const selfKey = getSelfKey(target);
    const other = othersByKey[selfKey] || null;
    Self.setRelated(target, relationName, other);
  });
  }
}
```

You can then work with relations like this:

```js
bookshelf('User', class extends bookshelf.Mapper {
  static get tableName() { return 'users' },
  static get relations() {
    return {
      projects: this.hasMany('Project', {otherReferencingColumn: 'owner_id'});
    }
  }
});

user = {id: 5};
bookshelf('User').related(user, 'projects').save([
  { name: 'projectA' },
  { name: 'projectB' }
]).then((projects) {
  // projects = [
  //   { id: 1, owner_id: 5, name: 'projectA' },
  //   { id: 2, owner_id: 5, name: 'projectB' }
  // ]
});
  
```

Internally `.related()` and `.load()` do something like this:

```js
import _, {noop} from 'lodash';

/*
Turns
[
  'some.other',
  {'some.thing.dude': queryCallback}, // Maintain the query callback here.
  {friends: 'adults' }, // Call scopes directly (could also chain ['adults', 'australian'])
  'parents^'
]

Into
{
  some: {
    nested: {
      other: {},
      thing: {
        nested: {
          dude: { callback: g => q.query(queryCallback) }
        }
      }
    }
  }
  friends: { callback: g => g.adults() }
  parents: {
    recursive: true,
    // always adds one extra for a recursive at the root.
    // This is how recursive relations can be solved!! :-D
    nested: {
      parents: { recursive: true }
    }
  
}
*/

function normalizeWithRelated(withRelated) {
// TODO
}

class Mapper {
  // ...
  related(instance, relationName) {
    // Either bookshelf instance or transaction.
    const client = this.getOption('client');
    const relation = _.isString(relationName)
      ? this.getRelation(relationName)
      : relationName;
    
    // Deliberately doing this check here to simplify relation code.
    // ie. simplify overrides by giving explit 'one' and 'many' methods.
    const mapper = _.isArray(instance)
      ? relation.forMany(client, instance)
      : relation.forOne(client, instance);
  }
  
  load(target, related) {
    const normalized = normalizeWithRelated(related);
    
    const relationPromises = _.mapValues(normalized, ({callback, nested}, relationName) => {
      const relation = this.getRelation(relationName);
      const gateway = this.related(target, relation)
      return gateway.withMutations(r =>
        // Ensure nested relations are loaded.
      // Optionally apply query callback.
        r.all().withRelated(nested).mutate(callback)
      )
      .fetch()
      .then(result => {
        // Get all the models and attach them to the targets.
      const targets = _.isArray(target) ? target : [target];
      return relation.attachToMany(targets, relationName, models);
      })
    });
    
    Promise.props(relationPromises).return(target);
  }
  
  fetch() {
    const query = this.query();
    let handler = null;
    if (this.getOption('single')) {
      query.limit(1);
    }
    query.bind(this).then(this._handleFetchResponse)
  }
  
  fetchOne(id) {
    return this.asMutable().one(id).fetch();
  }
  
  fetchAll(ids) {
    return this.asMutable().all(ids).fetch();
  }
  
  _handlefetchResponse(response) {
    const required = this.getOption('required');
    const single   = this.getOption('single')

    if (required) {
    this._assertFound(response);
    } 
    
    return this.forge(single ? _.head(response) : response);
  }
  
  _assertFound(result) {
    if (_.isEmpty(result)) {
      const single = this.getOption('single');
        // Passing `this` allows debug info about query, options, table etc.
      throw new (single ? NotFoundError : EmptyError)(this);
    }
  }
  
  // ...
}
```


```js
bookshelf(Person)
  .one().where('id', 5)
  .withRelated('pets')
  .fetch()
  .then((person) => {
    // person = {id: 5, name: 'Jane', pets: [{id: 2, owner_id: 5, type: 'dog'}]}
    
    // Create and save a new pet:
    
    let newPet = bookshelf(Person).related(person, 'pets').forge({type: 'mule'});    
    // newPet = {owner_id: 5, type: 'mule'};
    bookshelf('Animal').save(newPet);    

    // OR
    
    bookshelf('Person').related(person, 'pets').save({type: 'mule'})
        
    // OR (saving person and pets - with raw records)
    
    person.pets.push({type: 'mule'});
    bookshelf('Person').withRelated('pets').save(person);
    
    // OR (with active records)
    
    person.pushRelated('pets', {type: 'mule'}).save({withRelated: true});
  })
```

## Active Record Models

Because the core of the proposed API is taken care of by functions of `Mapper` instances, it's possible to use Bookshelf without `Model` instances at all. However, they are still useful and should be enabled by default.

However, because the new design is so different, the entire active record module can be separated into its own plugin that overrides hooks used within `forge`, `save`, `update` etc.

Base `Model` will look something like this:

```js
class Model {
  constructor: (Mapper, client, attributes) {
    this.Mapper = Mapper;
    this.client = client;
    this.initialize.apply(this, arguments);
  }
  
  // Overridable.
  intialize();

  // Loading stuff.
  refresh() {}
  load(relation) {}
  related(relation) {}
  
  // Shorthand for `client(Mapper).save(this.attributes, arguments)` etc.
  save() {}
  update() {}
  insert() {}
  patch(attributes) {}
  destroy() {}
  
  // Attribute management.
  hasChanged(attribute)
  set(attribute, value)
  get(attribute)
}
```

This is how it plugs in:


```js
// This is the default bookshelf.Mapper (just returns plain objects/arrays)
class Mapper {
  constructor: (client) {
    this.option('client', client);
  }
  
  // ...
  
  // Basic record modification methods to be overridden by plugin modules.
  createRecord(attributes) {
    return attributes;
  }
  
  setAttributes(record, attributes) {
    return _.extend(record, attributes);
  }
  
  getAttributes(record) {
    return record;
  }
  
  setRelated(record, relations) {
    return _.extend(record, relations);
  }
  
  getRelated(record, relation) {
    return record[relation];
  }
  
  // Private helper.
  
  _forgeOne(attributes) {
    _.defaults(attributes, this.option('defaultAttributes'));
    return this.createRecord(attributes);
  }
  
  // Public interface.
  
  forge(attributes = {}) {
  
    if (_.isArray(attributes)) {
      let instances = attributes.map(this._forgeOne, this);
      this.trigger('forged forged:many', instances);
      return instances;
    }
    
    if (_.isObject(attributes)) {
      let instance = this._forgeOne(attributes);
      this.trigger('forged forged:one', instance);
      return instance;
    }
    
    throw new Error('`attributes` must be instance of Object or Array');
  }
  
  // ...
}

// This is the new ModelMapper, that produces `bookshelf.Model` instances.

import BaseModel from 'base-model';

class ModelMapper extends bookshelf.Mapper {
  
  initiailize() {
    this.active().model(bookshelf.Model);
  }

  model(model) {
    return this.setOption('model', model);
  }

  active() {
    return this.setOption('plain', false);
  }

  plain() {
    return this.setOption('plain', true);
  }
  
  createRecord(attributes) {
    // Allow any processing from other plugins.
    const superRecord = super.createRecord(attributes);

    const {plain, Model} = this.getOptions('plain', 'Model');

    // If chained with `.plain()`, or no model has been specified.
    return plain || !Model
      ? superRecord
      : new Model(this, this.getAttributes(superRecord));
  }
  
  createModel(Model, attributes) {
    return 
  }
  
  setAttributes(record, attributes) {
    if (record instanceof BaseModel) {
      // Allow any processing from other plugins.
      attributes = super.setAttributes({}, attributes);
      return record.set(attributes);
    }
    return super.setAttributes(record, attributes);
  }
  
  getAttributes(record, attributes) {
    if (record instanceof bookshelf.Model) {
      return record.attributes;
    }
    return super.getAttributes(record, attributes);
  }
  
  setRelated(record, relations) {
    ...
  }
}

// Then we plug it in:
bookshelf.Model = BaseModel;
bookshelf.Mapper = ModelMapper;

```

#### Parse/Format

You can also use this same override pattern for parse/format (whether using `ModelMapper` plugin or not).

```js
class FormattingMapper extends bookshelf('Mapper') {

  initialize() {
    this.formatted();
  }

  formatted() {
    return this.setOption('unformatted', false);
  }
  
  unformatted() {
    return this.setOption('unformatted', true);
  }
  
  formatKey(key) {
    return _.underscored(key);
  }
  
  parseKey(key) {
    return _.camelCase(key);
  }
  
  createRecord(attributes) {
    let record = super.createRecord({});
    this.setAttributes(record, attributes);
    return record;
  }
  
  setAttributes(record, attributes) {
    if (!this.option('unformatted')) {
      attributes = _.mapKeys(attributes, this.parseKey, this);
    }
    return super.setAttributes(record, attributes);
  }
  
  getAttributes(record) {
    let unformatted = super.getAttributes(record);
    return this.option('unformatted')
      ? unformatted : _.mapKeys(unformatted, this.formatKey, this);
  }
}

// Then we plug it in:
bookshelf.registerMapper('Model', Model);
bookshelf.registerMapper('Mapper', FormattingMapper);
```


<script src="http://yandex.st/highlightjs/7.3/highlight.min.js"></script>
<link rel="stylesheet" href="http://yandex.st/highlightjs/7.3/styles/github.min.css">
<script>
  hljs.initHighlightingOnLoad();
</script>
