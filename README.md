## Async/Await in Angularjs

ES7 introduces async functions, which allow you to write asynchronous code with a synchronous syntax. Unfortunately this doesn't work well with AngularJS: awaited code will run outside of Angular's digest loop, and therefore it won't trigger watchers or view updates.

Whenever you commit to this repository, GitHub Pages will run [Jekyll](https://jekyllrb.com/) to rebuild the pages in your site, from the content in your Markdown files.

### Problem
ES7 introduces async functions that allow to write async code with a synchronous syntax.
This looks like this:

```
let vm = this;
  vm.user = {name: ""};
  
  vm.getUser = async function(userName) {
    vm.user = await UserService.getUser(userName);
    console.log(vm.user.name); // --> "foo"; 
  }
  
  vm.getUser("foo");
  ```
This works because await expects a promise as arguments, and it waits until the promise is resolved before continuing.

Unfortunately this doesn’t work well in AngularJS: awaited code is not run in the digest loop, and therefore we need to put $scope.$apply calls all over the place. 
This results in code like this:
```
vm.getUser = async function(userName) {
  vm.user = await UserService.getUser(userName);
  $scope.$apply();  // don't forget this! 
}
```
### Meet $async from angular-async-await 

https://www.npmjs.com/package/angular-async-await

Let $async take care of it for you.
```
"use strict";
 
let SampleViewCtrl = ['DummyService', '$async', function (DummyService, $async) {
  let vm = this;
  vm.user = {};
 
  vm.getUser = $async(async function (userName) {
    vm.user = await DummyService.getUser(userName);
    // vm.user is now updated in the view 
  });
  
  vm.getUser("Ben Hansen");
}];
 
export default SampleViewCtrl;
```
### How it works

Thanks to the generator functions introduced in ES6 we can write our own async engine that does work well with AngularJS. At Magnet.me we’ve created the $async module which does exactly that. 
With $async we can rewrite the initialize function like this:
```
const initialize = $async(function*() {
	$scope.user = yield users.getActiveUser();
	$scope.messages = yield conversations.getMessagesFor($scope.user);
});
```
Instead of an async function we now use a generator function passed to $async, and instead of awaiting a promise we now yield a promise.
Generator functions do not run to completion when they are invoked. Instead they return an iterator that can be iterated over manually. This iterator will stop and “return” at every call to yield.
For example, the following generator function generates all positive integers:
```
function* integersGenerator() {
	let i = 1;
	while (true) {
		yield i;
		i++;
	}
}

const integersIterator = integersGenerator();
integersIterator.next().value; //1
integersIterator.next().value; //2
integersIterator.next().value; //3
```
The $async module makes use of the ability to pause generator functions to emulate async functions. It expects a generator that will generate promises, and it will chain all these promises together correctly. The result is that each yield call will wait for its promise to be resolved before continuing.
The core of $async looks something like this:
```
//This function iterates asynchronous over an iterator
function iterateAsync(iterator) {
	//First run until the first yield. 
	//`.value` should now contain the yielded promise
	return iterator.next().value
		//Then we wait until the promise has resolved, 
		//and recursively iterate again
		.then(() => iterateAsync(iterator));
}

//usage:
iterateAsync(function*() {
	yield somePromise;
	yield someOtherPromise;
	//etc.
});

//This is effectively the same as this:
somePromise
	.then(() => someOtherPromise())
	.then(/*etc*/);
```
Now that we have our own async engine it is trivial to make it work properly with Angular: 
we can add the call to $scope.$apply in the iterateAsync function and there it is: an async function that works with AngularJS!
