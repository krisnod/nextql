# NextQL
> Yet another graph query language

Both love and hate GraphQL.
#### Love
* Elimination of N+1 queries
* Async resolve results from multi backends, then combine into single result.

#### Hate
* Text based query? It is so SQL - we back to era of query builder tools.
* Root query and mutation? It is nightmare to convert a tons of REST apis to GraphQL.
* Strong typed. Sorry I not fan of strong typed language.
* Not flexible. Most request features still go nowhere: Annotation, Input interface. 

# Introduction to NextQL
NextQL like GraphQL
* Specify the exactly data needed for a view and fetch that data in a single network request.
* Traverse graph data or related objects.

Different with GraphQL
* No strong typed or schema.
* JSON based query
* No root queries or mutations

## Query
NextQL query is a JSON object
```
{
	"person": {
		"me": {
			"name": 1
		}
	}
}
```

It meaning you tell NextQL service goto "person" model and invoke "me" method, then pick "name" field of the result of previos method call.

The JSON result should be
```
{
	"person": {
		"me": {
			"name": "Nguyen Huu Giap"
		}
	}
}
```

### Arguments
Argments pass over specify field "$params"
```
{
	"person": {
		"get": {
			"$params": { "id": "giapnh" },
			"name": 1
		}
	}
}
```
The JSON result should be
```
{
	"person": {
		"get": {
			"name": "Nguyen Huu Giap"
		}
	}
}
```
### Aliases
Field name has postfix to seperate the same method call with different arguments
```
{
	"person": {
		"get/giapnh": {
			"$params": { "id": "giapnh" },
			"name": 1
		},
		"get/nguyen": {
			"$params": { "id": "nguyen" },
			"name": 1
		}
	}
}
```
The JSON result should be
```
{
	"person": {
		"get/giapnh": {
			"name": "Nguyen Huu Giap"
		},
		"get/nguyen": {
			"name": "Dinh Thi Kim Nguyen"
		}
	}
}
```
You should able config alias seperator whatever you want "get$giapnh" or "get#giapnh"

### Traverse related object
Follow link field.
```
{
	"person": {
		"get/giapnh": {
			"$params": { "id": "giapnh" },
			"name": 1,
			"children": {
				"name": 1
			}
		},
		"get/nguyen": {
			"$params": { "id": "nguyen" },
			"name": 1,
			"children": {
				"name": 1
			}
		}
	}
}
```

The JSON result should be
```
{
	"person": {
		"get/giapnh": {
			"name": "Nguyen Huu Giap",
			"children": [{
				"name": "Nguyen Huu Vinh"
			}]
		},
		"get/nguyen": {
			"name": "Dinh Thi Kim Nguyen",
			"children": [{
				"name": "Nguyen Huu Vinh"
			}]
		}
	}
}
```

## Schema 
NextQL schema is a groups of models defined as plain Javascript object
```
const personSchema = {
	fields: {
		firstName: 1,
		lastName: 1
	},

	computed: {
		fullName(source, params, context){
			return source.firstName + source.lastName;
		}
	},

	methods: {
		get(params, context){
			return context.db.get(params.id);
		}
	}
}
```
Schema use to resolve an object into query value.

* **fields**: Any field exposed for query. Resolve direct from source, apply for primitive value only.
* **computed**: Apply for virtual field which not present in source object or you want expose source object's methods.
* **methods**: Any methods the model expose for query.

In exaggeration, schema's fields map with object's field. schema's computed map with object's instance methods, and finally schema's methods map with object's static methods.

### How to map an Object with a Model
GraphQL force you define input and output type - so it easy map a result object into specify type. NextQL let you free to return any kind of object, so how it decide which model applied?

Easily, NextQL treat all object same as an GraphQL Interface. So you must provide a global resolveType function to resolve model name from object.

NextQL ship with defaultResolveType which lookup value constructor name for model name. 

```
const defaultResolveType = (value) => value.constructor && value.constructor.name;
```

But you free to choose whatever to resolve type from object. It could be mongoose model name, __type field ...

If failed to resolve type from object, NextQL will fall back to default behavior: It only allow to query primitive field value. This behavior help you free from define unnecessary simple models.   

## Compare with GraphQL
Compare with [getDie sample from GraphQL.js] (http://graphql.org/graphql-js/object-types/)
```
var {graphql, buildSchema} = require('graphql');

var schema = buildSchema(`
  type RandomDie {
    numSides: Int!
    rollOnce: Int!
    roll(numRolls: Int!): [Int]
  }

  type Query {
    getDie(numSides: Int): RandomDie
  }
`);

// This class implements the RandomDie GraphQL type
class RandomDie {
    constructor(numSides) {
        this.numSides = numSides;
    }

    rollOnce() {
        return 1 + Math.floor(Math.random() * this.numSides);
    }

    roll({numRolls}) {
        var output = [];
        for (var i = 0; i < numRolls; i++) {
            output.push(this.rollOnce());
        }
        return output;
    }
}

// The root provides the top-level API endpoints
var root = {
    getDie: function ({numSides}) {
        return new RandomDie(numSides || 6);
    }
}

module.exports = function () {
    return graphql(schema, `{
  getDie(numSides: 6) {
    rollOnce
    roll(numRolls: 3)
  }
}`, root)
}
```

**Implement use NextQL**

```
const nextql = require('./nextql');

class RandomDie {
	constructor(numSides) {
		this.numSides = numSides;
	}

	rollOnce() {
		return 1 + Math.floor(Math.random() * this.numSides);
	}

	roll({ numRolls }) {
		var output = [];
		for (var i = 0; i < numRolls; i++) {
			output.push(this.rollOnce());
		}
		return output;
	}
}


nextql.model('RandomDie', {
	computed: {
		rollOnce(source) {
			return source.rollOnce();
		},
		roll(source, params) {
			return source.roll(params)
		},
	},
	methods: {
		getDie: function ({ numSides }) {
			return new RandomDie(numSides || 6);
		}
	}
})

module.exports = function run() {
	return nextql.execute({
		RandomDie: {
			getDie: {
				$params: {
					numSides: 6
				},
				rollOnce: 1,
				roll: {
					$params: {
						numRolls: 3
					}
				}
			}
		}
	});
}
```

Benchmark result:
```
graphql#getDie x 7,324 ops/sec ±6.93% (72 runs sampled)
nextql#getDie x 44,043 ops/sec ±5.38% (68 runs sampled)
Fastest is nextql#getDie
```

## Installing / Getting started


## Developing


### Building



### Deploying / Publishing


## Features

## Licensing

"The code in this project is licensed under MIT license."

