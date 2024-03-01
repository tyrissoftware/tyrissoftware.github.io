---
layout: post
title: "Introducing ValueStore 📦"
date: 2024-02-26
categories: ios library swift apple
tags: ios library swift apple
---

# Introducing Value Store: a general interface for persistence

TL;DR UserDefaults has many issues, it should not be used directly. We introduce a library with a simple interface for persisting values.

## The problem

UserDefaults are very useful and easy to use, however, they have many issues that are not usually discussed. Let’s see what the issues are and what can be done about it.

### Unsafe value types

*UserDefaults* types say they can handle *Any* type, but this is not true. Try storing a custom struct value and see what happens:

![Exception thrown when trying to save an unsupported type](/assets/images/exception.jpg)

They can only save some types, but the compiler won’t catch these errors for you.

Also, try saving a value of a type and then retrieving it with a different type. The compiler won’t help you here either:

![Saving an Int and loading a String](/assets/images/types.jpg)

Depending on the types you might get an implicit conversion, a crash or whatever.

### Unsafe keys

UserDefaults use strings as keys, which is problematic. First, nothing checks for typos. You can easily save a value into one key, and then try to load it from a different one. If some shortcut goes amiss and a symbol modifies one of your keys, you won’t get any warnings or errors. There’s also nothing preventing you from reusing a key for a different purpose, with potentially catastrophic consequences.

### Too specific

Another problem with UserDefaults is that it’s way too specific. If you sprinkle your code with direct uses of it and later you find out that you need to move your data to the Keychain, good look ensuring that you changed all the places where that key was used, and that the migration is done correctly.

We have designed ValueStore to solve all these issues, while adding support for async operations that might error out.

## The solution

First we need to realize that we can’t have a single dictionary that stores different types in a type safe way at compile time. Maybe one day this will be possible (with the power of dependent types), but for now we need to separate each value into a different store. In any case, we will probably want to use a different implementation for each value, like storing sensitive data into the Keychain, and some data into the file system, so a single dictionary won’t suffice.

The solution is to introduce an interface that can load, store and remove individual values of a single type in a type safe manner:

{% highlight swift %}
struct ValueStore<Value> {
	var load: () throws -> Value
	var save: (Value) throws -> Value
	var remove: () throws -> Void
}
{% endhighlight %}

This follows the idea to use structs as more flexible interfaces compared to protocols: https://www.pointfree.co/episodes/ep33-protocol-witnesses-part-1 


Let’s see how this solves each one of the previous issues:

### Unsafe value types

A ValueStore has an explicit generic type for the value, which ensures type safety. Furthermore, the constructors for UserDefaults stores require that the value type conforms to the protocol PropertyListValue, to ensure that you can only store values of supported types.

### Too specific

ValueStore is just a generic interface, so you can have different implementations that point to the UserDefaults, the file system, memory, the Keychain, an endpoint, …



If you ever need to migrate a value from one place to another, you can use the [replacing][Utilities] utility. Every time we load data from the store, it will try to load from the new store, and if there’s no data it will move it from the old store. You can of course implement your own migration strategies if you have different needs.

### Unsafe keys

We can use an enum with a String representation to store our UserDefaults keys. This way the compiler will be able to ensure that we won’t duplicate keys:

{% highlight swift %}
enum UserDefaultsKey: String {
	case preference1
	case preference2
}
{% endhighlight %}

Also, if you accidentally modify one of these keys, the compiler will helpfully point you to all the code that is broken, because you changed a preference key name. And no need to worry about using a different key for loading, saving or removing your data.

The [Usage][#usage] section in the documentation explains this in more detail.

### Generalizing it

This interface can be used to access UserDefaults, but it’s a bit limited for other purposes. Let’s see how to make it more general.

First, we can make the functions asynchronous, so that we can use them for non synchronous operations:

{% highlight swift %}
struct ValueStore<Value> {
	var load: () async throws -> Value
	var save: (Value) async throws -> Value
	var remove: () async throws -> Void
}
{% endhighlight %}

Finally, in some cases we might need dynamic parameters in order to perform an operation, such as network tokens, so let’s add an Environment type parameter to allow this:

{% highlight swift %}
struct ValueStore<Environment, Value> {
	var load: (Environment) async throws -> Value
	var save: (Environment, Value) async throws -> Value
	var remove: (Environment) async throws -> Void
}
{% endhighlight %}

This is the final ValueStore interface, pretty much as it’s defined in the library. Helper functions are defined for the cases where the Environment parameter is not needed (it can be Void in this case).

## Usage

The following is an example of how the library might be used. Of course, you can use it in other ways, but this should provide a good starting point.

Define a struct to hold all the persisted values:

{% highlight swift %}
struct PersistenceEnvironment {
	var userPreference1: ValueStore<Void, String>
	var userPreference2: ValueStore<Void, Int>
	var networkPreference: ValueStore<NetworkEnvironment, String>
}
{% endhighlight %}

Now we should create an enum to hold all the keys to be used with UserDefaults:

{% highlight swift %}
enum UserDefaultsKey: String {
	case userPreference1
	case userPreference2
}
{% endhighlight %}

This way the compiler will ensure that all keys are unique. Now we need to create a constructor that will use this key, which will use the unsafeRawUserDefaults implementation internally:

{% highlight swift %}
extension ValueStore {
	static func userDefaults(_ key: UserDefaultsKey) -> Self {
		.unsafeRawUserDefaults(key.rawValue)
	}
}
{% endhighlight %}

You can also have other implementations that store the value using an endpoint, the keychain or whatever:

{% highlight swift %}
extension ValueStore where Environment == NetworkEnvironment {
	static func network(_ endpoint: String) -> Self {
		.init(
			load: { networkEnvironment in
				try await get(...)
			},
			save: { value, networkEnvironment in
				try await post(...)
			},
			remove: { networkEnvironment in
				try await delete(...)
			}
		)
	}
}
{% endhighlight %}

Now we can create a live version of our PersistenceEnvironment:

{% highlight swift %}
extension PersistenceEnvironment {
	static var live: Self {
		.init(
			userPreference1: .userDefaults(.userPreference1),
			userPreference2: .userDefaults(.userPreference2),
			networkPreference: .network("preference")
		)
	}
}
{% endhighlight %}

Finally you can use these stores to read and write your values:

```swift
let environment = PersistenceEnvironment.live

var storedPreference1 = try await environment.userPreference1.load()
storedPreference1 += "!"

try await environment.userPreference1.save(storedPreference1)
```

Check the documentation or the tests for a better understanding on how to use it, and all the utilities that come with ValueStore.

[UserDefaults]: https://github.com/tyrissoftware/swift-valuestore/blob/master/Documentation/UserDefaults.md#userdefaults
[Utilities]: https://github.com/tyrissoftware/swift-valuestore/blob/master/Documentation/Utilities.md