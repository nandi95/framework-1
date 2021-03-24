# Testing

Upfront is fully tested to give as much confidence in the code as possible. To carry on this ethos in your application, upfront offers helpful tools to for testing with mock data and test outgoing requests.

## Testing service implementations
Swapping out [services](./services/readme.md) of upfront is easy as setting them in the [GlobalConfig](./helpers/global-config.md).
For example if you want to test your custom method you added to a model, you could do something similar to the following:

<code-group>
<code-block title="Javascript">
```js
import { GlobalConfig } from '@upfrontjs/framework';
import User from '@/Models'

const config = new GlobalConfig;

describe('customAjaxMethod()', () => {
    const mockFunc = jest.fn();
    const user = new User;
    config.set('api', {
        handle: mockFunc
    });
    
    it('should initiate a GET request', async () => {
        await user.customAjaxRequest();
        
        expect(mockFunc).toHaveBeenCalledWith('url', 'get', 'data', 'customHeaders')
    });
});
```
</code-block>

<code-block title="Typescript">
```ts
import { GlobalConfig } from '@upfrontjs/framework';
import type { Configuration } from '@upfrontjs/framework';
import type MyConfig from '@/MyConfig';
import User from '@/Models/User'

const config: GlobalConfig<MyConfig extends Configuration> = new GlobalConfig;

describe('customAjaxMethod()', () => {
    const mockFunc = jest.fn();
    const user = new User;
    config.set('api', {
        handle: mockFunc
    });
    
    it('should initiate a GET request', async () => {
        await user.customAjaxRequest();
        
        expect(mockFunc).toHaveBeenCalledWith('url', 'get', 'data', 'customHeaders')
    });
});
```
</code-block>
</code-group>

## Factories

To make testing your code easier with real data independent of your back end, upfront provides factories. With factories, you can construct [models](./calliope/readme.md) hydrated with data that you have defined and/or using a random data generator like [ChanceJs](https://chancejs.com/) set in your [configuration](./helpers/global-config.md#randomdatagenerator). With the  factory defined, construction of a model is easy as `User.factory().create()`

To define a factory what you need to do is extend upfront's `Factory` in you class then in you model create a `factory()` method which returns your constructed factory.

<code-group>
<code-block title="Javascript">
```js
// UserFactory.js
import { Factory } from '@upfrontjs/framework';

export default UserFactory extends Factory {
    definition(model, index) {
        return {
            name: 'user name',
            resourceUuid: String.uuid()
        };
    }
}

// User.js
import { Model } from '@upfrontjs/framework';
import UserFactory from '../../factories/UserFactory';

export default User extends Model {
    factory() {
        return new UserFactory
    }
}
```
</code-block>

<code-block title="Typescript">
```ts
// UserFactory.ts
import { Factory } from '@upfrontjs/framework';
import type { Attributes } from '@upfrontjs/framework';
import type User from './src/Models/User';

export default UserFactory<T extends User> extends Factory {
    definition(model: T, index: number): Attributes {
        return {
            name: 'user name',
            resourceUuid: String.uuid()
        };
    }
}

// User.ts
import { Model } from '@upfrontjs/framework';
import UserFactory from '../../factories/UserFactory';

export default User extends Model {
    // or the return value of `Factory<this>` is also valid
    factory(): UserFactory {
        return new UserFactory
    }
}
```
</code-block>
</code-group>

#### definition
The `definition` is the only required method on the factory. This base definition will be used by all your factory calls.
In the above example `definition()` indicates that it takes 2 arguments. the first being an empty instance of the [model](./calliope/readme.md) you're creating. The second is an index starting from `1` which you may use to add some dynamic variable to your attributes (this is especially useful when creating multiple models in the same go e.g.: `username: 'nickname_' + index'`).

Furthermore, you may define attributes as functions which will be called with the attributes resolved up to the point of calling your function. For example
```js
return {
    firstName: () => 'user',
    lastName: 'name',
    fullName: (attributes) => {
        return attributes.firstName + ' ' + attributes.lastName // 'user name'
    }
}
```

If you would like to define [relationships](./calliope/relationships.md) in your `definition` you could pass the relation's value, and the relation's foreign key.

```js
const team = Team.factory().create()
return {
    teamId: team.getKey(),
    team: team // both Model and raw attributes are acceptable here
}
```

### States
States are a way to encapsulate some changes to the return value of your [definition()](#definition) method. It is resolved the same way as the `definition`, so you may use methods, and the previously resolved attributes.

To define a state in your factory add a method with your state name:

<code-group>
<code-block title="Javascript">
```js
// UserFactory.js
import { Factory, ModelCollection } from '@upfrontjs/framework';

export default UserFactory extends Factory {
    definition(model, index) {
        return {
            name: 'user name'
        };
    }

    nameOverride(model, index) {
        return {
            name: 'new name'
        };
    }
}
```
</code-block>

<code-block title="Typescript">
```ts
// UserFactory.ts
import { Factory, ModelCollection } from '@upfrontjs/framework';
import type { Attributes } from '@upfrontjs/framework';
import type User from './src/Models/User';

export default UserFactory<T extends User> extends Factory {
    definition(model: T, index: number): Attributes {
        return {
            name: 'user name'
        };
    }
    
    nameOverride(model: T, index: number): Attributes {
        return {
            name: 'new name'
        };
    }
}
```
</code-block>

</code-group>

Then call your state like: `User.factory().state('nameOverride').create()` For more instructions on how to use the method see [state](#state).


### Factory Hooks
 Factory hooks are methods that called when creating the models. There are two available hooks: `afterMaking` and `afterCreating`.
These hooks if implemented in your [factory](#factories) will receive the result of the builder at their respective times. As these are passed by reference you, can change their values to your liking without having to return it.

<code-group>
<code-block title="Javascript">
```js
// UserFactory.js
import { Factory, ModelCollection } from '@upfrontjs/framework';

export default UserFactory extends Factory {
    definition(model, index) {
        return {
            name: 'user name'
        };
    }

    afterMaking(modelOrCollection) {
        if (ModelCollection.isModelCollection(modelOrCollection)) {
            modelOrCollection.forEach(model => model.setAttribute('name', 'name from hook'));
        } else {
            modelOrCollection.setAttribute('name', 'name from hook');
        }
    }
}
```
</code-block>

<code-block title="Typescript">
```ts
// UserFactory.ts
import { Factory, ModelCollection } from '@upfrontjs/framework';
import type { Attributes } from '@upfrontjs/framework';
import type User from './src/Models/User';

export default UserFactory<T extends User> extends Factory {
    definition(model: T, index: number): Attributes {
        return {
            name: 'user name'
        };
    }
    
    afterMaking(modelOrCollection: ModelCollection<T> | T): void {
        if (ModelCollection.isModelCollection(modelOrCollection)) {
            modelOrCollection.forEach(model => model.setAttribute('name', 'name from hook'));
        } else {
            modelOrCollection.setAttribute('name', 'name from hook');
        }
    }
}
```
</code-block>

</code-group>

#### `random`

If in the [configuration](./helpers/global-config.md) you have set the key [`randomDataGenerator`](./helpers/global-config.md#randomdatagenerator), the value will be available to use in your factories like so:
```js
import { Factory, ModelCollection } from '@upfrontjs/framework';

export default UserFactory extends Factory
{
    definition()
    {
        return {
            name: this.random.name()
        };
    }
}
```

---
With the factory defined you're ready to start creating models. by either the [factory](./helpers/readme.md#factory) method or by the static `factory` method on your model `Model.factory()`

## FactoryBuilder
 
FactoryBuilder is a class that you can access by the [factory](./helpers/readme.md#factory) helper method or by calling the `factory` method on your model constructor. This utility enables us to fluently build models. It offers a couple methods to that extent. 

#### times

The `times` method sets the amount of models that is returned by the builder. The builder by default returns `1` model. Setting this amount to more than `1` will cause the builder to return a [ModelCollection](./calliope/model-collection.md).
```js
User.factory().times(3).create() // ModelCollection
```

#### with
The `with` method is a way to add [relations](./calliope/relationships.md) in line to the model. This is an alternative to adding it to the argument at the creation, and is the preferred way as it's more concise and provides type safety. You can pass he model constructor or FactoryBuilder instance to the method. You may pass an optional second argument to specify the relation's name with or without the [relation prefix](./calliope/relationships.md#relationmethodprefix)
```js
User.factory().with(Contract).create().team // Contract
User.factory().with(Shift.factory().times(2)).create().shifts // ModelCollection
```

::: warning
 - The passed in values will be constructed with the same method ([raw](#raw), [make](#make), [create](#create)) as the builder itself.
 - As expected a [belongsTo](./calliope/relationships.md#belongsto) relationship will need the foreign key to be set on the model you're building.
:::

As with relations, it is going to respect the relation type, that is for example, if you pass a model constructor that is defined as [`hasMany`](./calliope/relationships.md#hasmany) relation, it will return a [ModelCollection](./calliope/model-collection.md) even if it's a `builder` with `times` set to 1.

::: tip
Adding relation can also be added manually in the [`definition`](#definition) method, states and in the creation's argument.
:::

#### state

The state method applies the [states](#states) defined in your factory. You may pass a string, or an array of strings as an argument.
```js
User.factory().state('admin').create()
User.factory().state(['admin', 'withTeam']).create()
```
---
The following are the last methods to be called on the builder. These methods return the result of the build. 
#### raw

The `raw` method instruct the builder to construct your model(s). It will return the attributes in an object literal or a [Collection](./helpers/collection.md) of them depending on the [`times`](#times) method call or lack there of. This will not include any primary keys and [timestamps](./calliope/timestamps.md).

#### make

The `make` method instruct the builder to construct your model(s). It will return an instance of your model or a [ModelCollection](./calliope/model-collection.md) depending on the [`times`](#times) method call or lack there of. This will not include any primary keys and [timestamps](./calliope/timestamps.md).

#### create

The `create` method instruct the builder to construct your model(s). It will return an instance of your model or a [ModelCollection](./calliope/model-collection.md) depending on the [`times`](#times) method call or lack there of. This will include primary keys and [timestamps](./calliope/timestamps.md).