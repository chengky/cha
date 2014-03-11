<p align="center">
  <img height="256" width="256" src="https://f.cloud.github.com/assets/677114/2333826/7f36d7ea-a471-11e3-99ea-fd8567069b7d.png"/>
</p>

cha ![NPM version](https://badge.fury.io/js/cha.png) ![Build Status](https://travis-ci.org/chajs/cha.png)
===
> Make task chaining.

Cha allows tasks to be connected together into a chain that makes better readability and easier to maintain.

## How to getting started?

Installing cha via NPM, this will install the latest version of cha in your project folder
and adding it to your `devDependencies` in `package.json`:
```sh
npm install cha --save-dev
```

Touch a tasks file and naming whatever you love like `build.js`:
```js
// Load cha library.
var cha = require('cha')
// Load tasks directory with an index.js that exports ./tasks/glob.js, etc.
var tasks = require('./tasks')

// Register tasks that should chaining.
cha.in('glob',     tasks.glob)
    .in('cat',     tasks.cat)
    .in('replace', tasks.replace)
    .in('write',   tasks.write)
    .in('uglifyjs',tasks.uglifyjs)
    .in('copy',    tasks.copy)
    .in('request', tasks.request)

// Define task via chaining calls.
cha()
    .glob({
        patterns: './fixtures/js/*.js',
        cwd: __dirname
    })
    .replace({
        search: 'TIMESTAMP',
        replace: +new Date
    })
    .replace({
        search: /DEBUG/g,
        replace: true
    })
    .request('http://underscorejs.org/underscore-min.js')
    .map(function(record){
        console.log(record.path)
        return record;
    })
    .cat()
    .uglifyjs()
    .write('./out/foobar.js')
    .copy('./out/foobar2.js')
```

Add a arbitrary command to the `scripts` object:
```json
{
  "name": "cha-example",
  "scripts": {
    "build": "node ./build.js"
  },
  "devDependencies": {
    "cha": "~0.1.0"
  }
}
```

To run the command we prepend our script name with run:
```sh
$ npm run build

> cha@0.0.1 build 
> node ./test/build

request http://underscorejs.org/underscore-min.js
concat ./test/fixtures/bar.js,./test/fixtures/foo.js,http://underscorejs.org/underscore-min.js
write ./out/foobar.js
copy out/foobar.js > ./out/foobar2.js
```

## How to use watch task?

```js
var cha = require('../')
var tasks = require('./tasks')

// Set a watcher.
cha.watch = require('./tasks/watch')

cha.in('read',    tasks.read)
   .in('cat',     tasks.cat)
   .in('coffee',  tasks.coffee)
   .in('write',   tasks.write)
   .in('uglifyjs',tasks.uglifyjs)

// Start watcher.
cha.watch('./fixtures/coffee/*.coffee', {
    cwd: __dirname,
    immediately: true
}, function(filepath, event, watched){

    cha().read(watched)
        .coffee()
        .cat()
        .uglifyjs()
        .write('./out/foobar3.js')
})

```

To run the command we prepend our script name with run:
```sh
$ npm run watch

> cha@0.0.1 watch 
> node ./test/watch

read /test/fixtures/coffee/bar.coffee
read /test/fixtures/coffee/foo.coffee
concat /test/fixtures/coffee/bar.coffee,/test/fixtures/coffee/foo.coffee
write ./out/foobar3.js
```

## How to enjoy cha expressions?

```js
// Load cha library.
var cha = require('cha')
var tasks = require('./tasks')

// Register tasks that should chaining.
cha.in('glob',      tasks.glob)
    .in('request',  tasks.request)
    .in('cat',      tasks.cat)
    .in('replace',  tasks.replace)
    .in('write',    tasks.write)
    .in('uglifyjs', tasks.uglifyjs)

// Start with cha expressions.
cha(['glob:./fixtures/js/*.js', 'request:http://underscorejs.org/underscore-min.js'])
    .replace({
        search: 'TIMESTAMP',
        replace: +new Date
    })
    .replace({
        search: /DEBUG/g,
        replace: true
    })
    .cat()
    .uglifyjs()
    .write('./out/foobar.js')
```

To run the command we prepend our script name with run:
```sh
$ npm run expr

> cha@0.0.1 expr 
> node ./test/expr

request http://underscorejs.org/underscore-min.js
concat http://underscorejs.org/underscore-min.js
write ./out/foobar.js
```
## How to creating custom task?

Chaining task should based on the [Task.JS](https://github.com/taskjs/spec) specification.

The following example creating a task concatenate files from the inputs. It extend from `Execution` class and define `execute` method:

```js
var _ = require('lodash');
var fs = require('fs');
var path = require('path');
var Execution = require('execution');
var Record = require('record');

var Concat = Execution.extend({
    // Naming your task.
    name: 'concat',
    // Task description.
    description: 'Concatenate files',
    // Task options.
    options: {
        banner: {
            default: '',
            description: 'The string will be prepended to the beginning of the contents'
        },
        endings: {
            default: '\n',
            description: 'Make sure each file endings with newline'
        },
        footer: {
            default: '',
            description: 'The string will be append to the end of the contents'
        }
    },
    // Override `execution` method.
    execute: function (resolve, reject) {
        var options = this.options;
        var logger = this.logger;
        var inputs = this.inputs;

        var endings = options.endings;
        var banner = options.banner;
        var footer = options.footer;

        var paths = _.pluck(inputs, 'path');
        logger.log(this.name, paths.join(','));

        var contents = this.concat(inputs, endings);

        // Return new record object.
        resolve(new Record({
            contents: banner + contents + footer
        }));
    },
    concat: function (inputs, endings) {

        return inputs.map(function (record, index, array) {
            var contents = record.contents.toString();
            if (!RegExp(endings + '$').test(contents)) {
                contents += endings;
            }
            return contents;
        }).reduce(function (previousContents, currentContents, index, array) {
                return previousContents + currentContents;
            }, '');

    }
});

module.exports = Concat
```

## Internal methods

Cha built-in some collection related methods: `map`, `filter`, `reject`, `find`, `findLast`, `uniq`, `first`, `last`, `at`, `sample`, `shuffle`.

### map(callback)
Creates an array of values by running each element in the collection through the callback.
```js
cha('glob:./**/*.*')
    .map(function(record, index, collection){
        return record;
    })
    // [<Record::Buffer path="./coffee/bar.coffee">, <Record::Buffer path="./coffee/foo.coffee">, <Record::Buffer path="./js/bar.js">, <Record::Buffer path="./js/foo.js">]
```

### filter(callback)
Iterates over elements of a collection, returning an array of all elements the callback returns truey for.
```js
cha('glob:./**/*.*')
    .filter(function(record, index, collection){
        return record.path.indexOf('.js')
    })
    // [<Record::Buffer path="./js/foo.js">, <Record::Buffer path="./js/bar.js">]
```

### reject(callback)
The opposite of `filter` this method returns the elements of a collection that the callback does `not` return truey for.
```js
cha('glob:./**/*.*')
    .filter(function(record, index, collection){
        return record.path.indexOf('.js')
    })
    // [<Record::Buffer path="./coffee/bar.coffee">, <Record::Buffer path="./coffee/foo.coffee">]
```

### find(callback)
Iterates over elements of a collection, returning the first element that the callback returns truey for.
```js
cha('glob:./**/*.*')
    .find(function(record, index, collection){
        return record.path.indexOf('.js')
    })
    // [<Record::Buffer path="./js/bar.js">]
```

### findLast(callback)
This method is like `find` except that it iterates over elements of a collection from right to left.
```js
cha('glob:./**/*.*')
    .findLast(function(record, index, collection){
        return record.path.indexOf('.js')
    })
    // [<Record::Buffer path="./js/foo.js">]
```

### uniq(callback)
Creates a duplicate-value-free version of an array using strict equality for comparisons, i.e. ===.
If a callback is provided each element of array is passed through the callback before uniqueness is computed.
```js
cha('glob:./**/*.*')
    .uniq(function(record, index, collection){
        return record.path.toLowerCase();
    })
    // [<Record::Buffer path="./coffee/bar.coffee">, <Record::Buffer path="./coffee/foo.coffee">, <Record::Buffer path="./js/bar.js">, <Record::Buffer path="./js/foo.js">]
```

### first([number])
Gets the first element or first n elements of an array.
```js
cha('glob:./**/*.*')
    .first()
    // [<Record::Buffer path="./coffee/bar.coffee">]
```
```js
cha('glob:./**/*.*')
    .first(2)
    // [<Record::Buffer path="./coffee/bar.coffee">, <Record::Buffer path="./coffee/foo.coffee">]
```

### last([number])
Gets the last element or last n elements of an array.
```js
cha('glob:./**/*.*')
    .last()
    // [<Record::Buffer path="./js/foo.js">]
```
```js
cha('glob:./**/*.*')
    .last(2)
    // [<Record::Buffer path="./js/bar.js">, <Record::Buffer path="./js/foo.js">]
```

### at([index])
Creates an array of elements from the specified indexes of the collection. Indexes may be specified as arrays of indexes.
```js
cha('glob:./**/*.*')
    .at(4)
    // [<Record::Buffer path="./js/foo.js">]
```
```js
cha('glob:./**/*.*')
    .at([3, 4])
    // [<Record::Buffer path="./js/bar.js">, <Record::Buffer path="./js/foo.js">]
```

### sample([number])
Retrieves a random element or n random elements from a collection.
```js
cha('glob:./**/*.*')
    .sample()
    // [<Record::Buffer path="./js/foo.js">]
```
```js
cha('glob:./**/*.*')
    .sample(2)
    // [<Record::Buffer path="./js/bar.js">, <Record::Buffer path="./coffee/foo.coffee">]
```

### shuffle()
Creates an array of shuffled values, using a version of the [Fisher-Yates shuffle](http://en.wikipedia.org/wiki/Fisher-Yates_shuffle).
```js
cha('glob:./**/*.*')
    .shuffle()
    // [<Record::Buffer path="./js/bar.js">, <Record::Buffer path="./coffee/bar.coffee">, <Record::Buffer path="./coffee/foo.coffee">, <Record::Buffer path="./js/foo.js">]
```

## Release History

* 2014-03-10    0.1.1    Custom tasks could override internal methods.
* 2014-03-05    0.1.0    Initial release.
