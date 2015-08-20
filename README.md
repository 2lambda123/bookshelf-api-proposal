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

```js
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

```js
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

```js
// A Gateway definition:

var Person = bookshelf.Gateway.extend({

  initialize: function() {
  
    // Configure schema information.
    this.tableName('people').idAttribute('id').relations({
      house: belongsTo('House'),
      children: hasMany('Person', {otherReferencingColumn: 'parent_id'})
    });
  },
  
  // Add scopes if you want.
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
  initialize() {
  	this.tableName('people').idAttribute('id').relations({
      house: this.belongsTo('House'),
      children: this.hasMany('Person', 'parent_id');
  	});
  }
  adults() {
    return this.all().where('age', '>=', 18);
  }
}

bookshelf('Person', Person);
```

Here is a thing you could do:

```js
class Person extends bookshelf.Gateway {
  initialize() {
  	this.tableName('people').idAttribute('id').relations({
      house: this.belongsTo('House'),
      children: this.hasMany('Person', 'parent_id');
  	});
  }
  
  drinkingAge(age) {
    return this.setOption('drinkingAge', age);
  }
  
  drinkers() {
    return this.all().where('age', '>=', this.getOption('drinkingAge'));
  }
}

// Bizarro inheritance/scoping by supplying an 'initializer'.
bookshelf('Person', Person);
bookshelf('Australians', Person, g => g.drinkingAge(18));
bookshelf('Americans', 'Person', {drinkingAge: 21});

Americans = bookshelf('Americans');
Australians = bookshelf('Americans');

Americans.where('sex', 'male').drinkers().query().toString();
// select users.* from users where sex = 'male' and age >= 21;

Australians.drinkers().query().toString();
// select users.* from users where age >= 18

// Hm, if `where` called `defaultAttributes` internally we could do this:
FemaleAustralians = Australians.where('sex', 'female');
FemaleAustralians.save({name: 'Jane'}, {name: 'Janette'});
// insert into people (name, sex) values ('Jane', 'female'), ('Janette', 'female');

```

Note that I've added a simple filter for `adults` above. Most methods on `Gateway` should be chainable to help build up queries (much like knex).

```js
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

```js
// Get a person from the Smith family, and their house and daughters.
bookshelf('Person')     // returns Person Gateway instance.
  .withRelated('house')
  .withRelated('children', query => query.where('sex', 'female'))
  .one()
  .where('surname', 'Smith')
  .fetch();

```

##### Gateway chain state: options, query and client

Gateway chains have four properties:

```js
import Immutable, { Iterable } from 'immutable';

class Gateway {

  // NOTE: client code never calls this constructor. It is called from within
  // the `Bookshelf` instance.
  //
  // bookshelf('MyModel').getOption('single') -> false
  //
  constructor(client, options = {}, query = null) {
    this._client = client;
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
  	
  	// Now allow extra defaults to be set by inheriting class.
  	this.withMutations(this.initialize);
  	
  	// Override those with supplied options. (This is not client facing,
  	// it's for use when gateway instances clone themselves from
  	// `.query` and `.setOption`).
  	//
  	// Calling `asImmutable()` here locks the instance.
  	//
  	this._options.merge(Immutable.fromJS(options)).asImmutable();
  	
  	// Now lock this instance down.
  	this.asImmutable();
  }
  
  initialize() { /* noop */ }
  
  // -- Options --
  
  getOption(option) {
  
    // Options must be initialized before they are accessed.
    if (!this._options.has(option)) {
      throw new InvalidOption(option, this);
    }
      
  	// Have to ensure references are immutable. Mutable gateway chains
  	// could leak.
  	return Iterable.IsIterable(result)
  		? result.AsImmutable()
  		: result;
  }
  
  // Change an option on the gateway chain. If 
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
  
  // See relations section for more info on `withRelated`.
  withRelated(related, query = null) {
    const normalized = normalizeWithRelated(related, query);
    return this.changeOption('withRelated', (withRelated) => {
      return withRelated.mergeDeep(normalized);
    });
  }
  
  // Do these two in the `withMutations` callback to prevent an extra copy
  // being made.
  all(ids) {
    return this.withMutations(g => {
      g.setOption('single', false);
      if (!_.isEmpty(ids)) {
        g.query(query => query.whereIn(g.idAttribute, ids));
      }
    });
  }
  
  one(id) {
  	return this.mutate(g => {
  	  g.setOption('single', true);
  	  if (id != null) {
  	  	idAttribute = this.getOption()
  	  	g.where(this.getOption)
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

#### bookshelf(gateway, Gateway, initializer)
Moving the registry plugin to core is integral to the new design.

Currently `Model` is aware of its Bookshelf client (and internal knex db connection) - and can only be reassigned by setting the `transacting` option. This is less flexible than it could be. Now every `Gateway` chain must be initialized via a client object.

```js
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


```js
import GatewayBase from './base/Gateway';

// Store an instantiated instance of a gateway. It doesn't matter that
// it's an instance because it's immutable. It's kind of the prototype
// pattern.
//
// Note the initializer. This is so you can do this for simple tables:
//
// bookshelf('User', 'Model', user => user.tableName('users').idAttribute('id'));
//
// The initializer is applied **after** the constructor is called.
function storeGateway(name, GatewayConstructor, initializer) {
    const gateway = instantiateGateway(Gateway);
 	bookshelf.gatewayMap.set(gateway, gateway.withMutations(initializer));
 	return bookshelf;
}

// Retrieve a previously stored gateway instance.
//
function retrieveGateway(gateway) {
	const gateway = bookshelf.gatewayMap.get(gateway);
	if (!gateway) {
		throw new Error(`Unknown Gateway: ${Gateway}`)
	}
	return gateway;
}

// Gets an immutable instance of either a gateway constructor or a
// stored gateway.
function instantiateGateway(gateway) {
	if (gateway instanceof GatewayBase) {
		return new gateway(bookshelf);
	}
	return retrieveGateway(gateway);
}

const bookshelf = function(gateway, Gateway, initializer) {

	// Store a Gateway for later retrieval...
	if (Gateway instanceof GatewayBase) {
		return storeGateway(gateway, Gateway, initializer);
	}
	
	// ...or instantiate a new Gateway:
	return instantiateGateway(gateway);
}


bookshelf.gatewayMap = new Map();
```

#### Relations

Currently relation code is mixed into `Collection` and `Model` via `Sync`. A `Relation` instance is created by the relation factory function (`hasMany`, `belongsTo` etc.). This is then attached to the `Model` or `Collection` instance as the `relatedData` propperty. `relatedData` is referenced by `Sync` when fetching models, and by collections when `create`ing new models.

This means relation logic is interpersed throughout all classes.

##### Proposal

Using `Gateway` relations become much simpler. A `Relation` is an interface that provides some methods: `forOne()`, `forMany()` and `attachMany()`.

For instance:

```js
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

```js
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

Because the core of the proposed API is taken care of by functions of `Gateway` instances, it's possible to use Bookshelf without `Model` instances at all. However, they are still useful and should be enabled by default.

However, because the new design is so different, the entire active record module can be separated into its own plugin that overrides hooks used within `forge`, `save`, `update` etc.

Base `Model` will look something like this:

```js
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


```js
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

```js
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
