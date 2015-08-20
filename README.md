# Bookshelf Proposal
## Glossary

 - *Active Record* A model instance that represents the state of a row and exposes `save`, `set` etc.
 - *Table Data Gateway*
 
## Gateway vs Model and Collection

#### Model and Collection
Bookshelf implements two main functionalities. An ability to generate queries from an understanding of relationships and tables (table data Gateway), and a way to model row data in domain objects (active record). Currently both of these responsibilities are shared by two classes: `Model` and `Collection`. 

#### Sync
This is conceptually messy, but also has resulted in code duplication between the two. The internal `Sync` class currently unifies the implementation of query building (table data Gateway). It also allows chaining of query methods by storing state. This means that query state and row state are interwoven in a way that is confusing and non-representative underlying data structure.



#### Collections

No more `Collection`s. Result sets are returned as plain arrays. Instead `Gateway` offers an API for bulk `save`, `insert` or `update`.

```
// No longer doing this:
Person.collection([fred, bob]).invokeThen('save').then( // ...
Person.collection([fred, bob]).invokeThen('save', null, {method: 'insert'}).then( // ...
Person.collection([fred, bob]).invokeThen('save', {surname: 'Smith'}, {patch: true}).then( //...

// Instead:
bookshelf('Person').save([fred, bob]).then( // ...
bookshelf('Person').insert([fred, bob]).then( // ...
bookshelf('Person').patch([fred, bob], {surname: 'Smith'}).then( //...
```

This offers a cleaner API, and greatly simplifies the `save` function. `save` currently has a lot of different options and flags, some of which make no sense when used together. Separating `patch` into its own method means that `save` no longer needs to support the `attrs` argument.

This means no more Lodash helpers and no intelligent `.add()` method.



### Proposal

#### Gateway

Defining table access information (`relations`, `idAttribute`, `tableName`) on `Model` is strange because models are our references to table rows. Ideally models and client code should be using a specific abstracted interface to access the database.

```
// -- Current --

// Why are we `forge`ing a person here?
// Note that `Person.where()` calls `forge` internally.
var Person = require('./person');
Person.forge().where('age', '>', 5).fetchAll().then((people) => // ...

// Or this strange thing:
Person.forge({id: 5}).fetch().then((person) => // ...

// -- Proposal --

// Now we create a new `Gateway` chain like this:
bookshelf('Person').all().where('age', '>', 5).fetch().then((people) => // ...

// Or this:
bookshelf('Person').one().where('id', 5).fetch().then((person) => // ...

```

`Gateway` takes on most of the current responsibilities of `Model`:

```
// A Gateway definition:

var Person = bookshelf.Gateway.extend({
  tableName: 'people',
  idAttribute: 'id',
  relations: function() {
    house: this.belongsTo('House'),
    children: this.hasMany('Person', {otherReferencingColumn: 'parent_id'})
  },
  adults: function() {
    return this.where('age', '>=', 18);
  },
});

// Register Gateway.
bookshelf('Person', Person);

// Same thing in ES6 notation.

// Note I'm using `getters` here because you can't define properties on
// the prototype using `class` syntax.
class PersonGateway extends bookshelf.Gateway {
  get tableName() { return 'people'; }
  get idAttribute() { return 'id'; }
  get relations() {
    return {
      house: this.belongsTo('House'),
      children: this.hasMany('Person', 'parent_id');
    }
  }
  adults() {
    return this.all().where('age', '>=', 18);
  }
}

bookshelf('Person', Person);
```

Note that I've added a simple filter for `adults` above. Most methods on `Gateway` should be chainable to help build up queries (much like knex).

```
// Example gateway methods.
class Gateway {

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

```
// Get a person from the Smith family, and their house and daughters.
bookshelf('Person')     // returns Person Gateway instance.
  .withRelated('house')
  .withRelated('children', query => query.where('sex', 'female'))
  .one()
  .where('surname', 'Smith')
  .fetch();

```

##### Gateway chain state: options, query and client

Gateway chains have three properties.

```
import Immutable, { Iterable } from 'immutable';

class Gateway {
  constructor(client, options = {}, query = null) {
    this._client = client;
  	this._query = query;
  	this._mutable = true;
  	
  	this._options = Immutable.fromJS(_.defaults(options, {
  	  withRelated: {},
  	  require: false,
  	  single: false,
  	  defaultAttributes: {},
  	}));
  	
  	this.withMutations(this.initialize);
  	this.asImmutable();
  }
  
  initialize() { /* noop */ }
  
  // -- Options --
  
  getOption(option) {
  	return this._options.get(option);
  }
  
  setOption(option, value) {
  
  	// The first time we call 'setMutability' on `_options` while mutable,
  	// it will return a mutable copy. Each additional call will return the
  	// same instance.
  	const newOptions = this._setMutability(this._options).set(option, value);
  	
  	// If there was a change, return the new instance.
    return newOptions === this._options
      ? this
      : new this.constructor(this._client, newOptions, this._query);
  }
  
  changeOption(option, setter) {
    let value = this.getOption(option);
    if (Iterable.IsIterable(value)) {
      value = this._setMutability(value);
    }
    return this.setOption(option, setter(value));
  }
  
  // -- Options examples --
  
  withRelated(related, query = noop) {
    return this.changeOption('withRelated', (withRelated) => {
      return _.isObject(related)
      	? withRelated.merge(related)
      	: withRelated.set(related, query);
    });
  }
  
  all(ids) {
    return this.mutate(instance => {
      instance.setOption('single', false);
      if (!_.isEmpty(ids)) {
        instance.query(query => query.whereIn(this.idAttribute, ids));
      }
    });
  }
  
  one(id) {
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
  	  const result = new this.constructor(this._client, this._options, this._query.clone());
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
  	if (_.isFunction(callback)) {
  	  const wasMutable = this._mutable;
  	  const mutable = this.asMutable(); 
      callback.bind(mutable)(mutable);
      mutable._mutable = wasMutable;
      return mutable;
    }
    return this;
  }
  
  // -- Helper --
   
  _setMutability(object) {
  	return object[this._mutable ? 'AsMutable' : 'AsImmutable']();
  }
}
```

#### bookshelf(Gateway, GatewayConstructor)
Moving the registry plugin to core is integral to the new design.

Currently `Model` is aware of its Bookshelf client (and internal knex db connection) - and can only be reassigned by setting the `transacting` option. This is less flexible than it could be. Now every `Gateway` chain must be initialized via a client object.

```
class User extends bookshelf.Gateway { /* ... */ }

// Using `User` directly.
bookshelf(User).save(newUser);

// Registering and reusing (helps break `require()` dependency cycles).
bookshelf('User', User);
bookshelf('User').where('age', '>', 18).fetch().then((users) =>

// Transaction objects are richer and take `Gateway` constructors similarly.
bookshelf.transation(trx =>
  trx('User').adults().fetch()
    .then(adults =>
      trx('User').patch(adults, {is_adult: true});
    )
);
```

This simply instantiates a new `Gateway` instance with the correct `client` attached (either a `Bookshelf` or `Transaction` instance).


```
import GatewayBase from './base/Gateway';

let bookshelf = function(gateway, Gateway) {

	// Store a Gateway for later retrieval...
	if (Gateway instanceof GatewayBase) {
		bookshelf.gatewayMap.set(gateway, Gateway);
		return Gateway;
	}
	
	// ...Otherwise, instantiate a new Gateway:
	
	// Handle either identifier or constructor.
	Gateway = _.isFunction(gateway)
		? gateway
		: bookshelf.gateway(gateway);
	
	// Create instance with client reference.
	return = new Gateway(bookshelf);
}

bookshelf.gatewayMap = new Map();

bookshelf.gateway = function(gateway) {
	// Retrieve a previously stored gateway.
	let Gateway = bookshelf.gatewayMap.get(gateway);
	if (!Gateway) {
		throw new Error(`Unknown Gateway: ${Gateway}`)
	}
	return Gateway;
};
```

#### Relations

Currently relation code is mixed into `Collection` and `Model` via `Sync`. A `Relation` instance is created by the relation factory function (`hasMany`, `belongsTo` etc.). This is then attached to the `Model` or `Collection` instance as the `relatedData` propperty. `relatedData` is referenced by `Sync` when fetching models, and by collections when `create`ing new models.

This means relation logic is interpersed throughout all classes.

##### Proposal

Using `Gateway` relations become much simpler. A `Relation` is an interface that provides some methods: `forOne()`, `forMany()` and `attachMany()`.

For instance:

```
class HasOne {
  constructor(SelfGateway, OtherGateway, keyColumns) {
  	this.Self = OtherGateway;
  	this.Other = OtherGateway;
  	
  	// Should consider how composite keys will be treated here.
  	// This can be done on a per-relation basis.
    this.selfKeyColumn = keyColumns['selfKeyColumn'];
    this.otherReferencingColumn = keyColumns['otherReferencingColumn'];
  }
  
  getSelfKey(instance) {
    return this.Self.getAttributes(instance, this.selfKeyColumn);
  }
  
  // Returns an instance of `Gateway` that will only create correctly
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

```
bookshelf('User', class extends bookshelf.Gateway {
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

```
import _, {noop} from 'lodash';

function normalizeWithRelated(value) {
  if (_.isString(value)) {
    let result = {};
    result[value] = _.noop;
    return result;
  } else {
    return value;
  }
}

// `thing.other` -> `thing`
function withRelatedHead(value) {
  const headRegex = /(.*?)(\.|$)/;
  return (value.match(headRegex) || [])[1];
}

// 'thing.other' -> 'other'
function withRelatedTail(value) {
  const tailRegex = /(\.|0)(.+?)$/;
  return (value.match(tailRegex) || [])[2];
}

/*

Turns
[
  'some.other',
  {'some.thing.dude': cba},
  {friend: cbb},
  'parents'
]

Into
{
  some:    {callback: noop, nestedRelated: ['other', {'thing.dude': cba]},
  friend:  {callback: cbb,  nestedRelated: []}
  parents: {callback: noop, nestedRelated: []}
}

*/
function organizeWithRelated(withRelated) {
// TODO
}

class Gateway {
  // ...
  related(instance, relationName) {
  	// Either bookshelf instance or transaction.
  	const client = this.getOption('client');
  	const relation = _.isString(relationName)
  		? this.getRelation(relationName)
  		: relationName;
  	
  	// Deliberately doing this check here to simplify relation code.
  	// ie. simplify overrides by giving explit 'one' and 'many' methods.
  	const gateway = _.isArray(instance)
  	  ? relation.forMany(client, instance)
  	  : relation.forOne(client, instance);
  }
  
  load(target) {
    const withRelated = this.getOption('withRelated').toJS();
    const organized = organizeWithRelated(withRelated);
    
    const relationPromises = _.mapValues(grouped, ({callback, nestedRelated}, relationName) => {
      const relation = this.getRelation(relationName);
      const gateway = this.related(target, relation)
      return gateway.mutate(r =>
        // Ensure nested relations are loaded.
    	// Optionally apply query callback.
      	r.all().withRelated(nestedRelated).mutate(callback)
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
  // ...
}
```


```
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

Because the core of the proposed API is taken care of by functions of `Gateway` instances, it's possible to use Bookshelf without `Model` instances at all. However, they are still useful and should be enabled by default.

However, because the new design is so different, the entire active record module can be separated into its own plugin that overrides hooks used within `forge`, `save`, `update` etc.

Base `Model` will look something like this:

```
class Model {
	constructor: (Gateway, client, attributes) {
		this.Gateway = Gateway;
		this.client = client;
		this.initialize.apply(this, arguments);
	}
	
	// Overridable.
	intialize();

	// Loading stuff.
	refresh() {}
	load(relation) {}
	related(relation) {}
	
	// Shorthand for `client(Gateway).save(this.attributes, arguments)` etc.
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


```
// This is the default bookshelf.Gateway (just returns plain objects/arrays)
class Gateway {
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

// This is the new ModelGateway, that produces `bookshelf.Model` instances.

class ModelGateway extends bookshelf.Gateway {
	
	get Model() {
		return bookshelf.Model;
	}

	plain() {
		return this.option('plain', true);
	}
	
	createRecord(attributes) {
		// Allow any processing from other plugins.
		superRecord = super.createRecord(attributes);
		
		// If chained with `.plain()` ModelGateway is bypassed.
		return this.option('plain')
		  ? superRecord
		  : this.createModel(super.getAttributes(superRecord));
	}
	
	createModel(attributes) {
		let model = new Model(this.constructor, this.option('client'));
		return model.set(attributes);
	}
	
	setAttributes(record, attributes) {
		if (record instanceof bookshelf.Model) {
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
bookshelf.Model = Model;
bookshelf.Gateway = ModelGateway;

```

#### Parse/Format

You can also use this same override pattern for parse/format (whether using `ModelGateway` plugin or not).

```
class FormattingGateway extends bookshelf.Gateway {
	
	unformatted() {
		return this.option('unformatted', true);
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
bookshelf.Model = Model;
bookshelf.Gateway = FormattingGateway;
```


<script src="http://yandex.st/highlightjs/7.3/highlight.min.js"></script>
<link rel="stylesheet" href="http://yandex.st/highlightjs/7.3/styles/github.min.css">
<script>
  hljs.initHighlightingOnLoad();
</script>