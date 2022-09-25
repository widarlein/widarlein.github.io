---
title: Clever converters in Flutter with Firebase Firestore
date: 2022-09-24
categories: ["flutter tricks"]
tags: [flutter, firebase]     # TAG names should always be lowercase
---
# Existing converters
It is no secret that the data conversion from our custom types written in Dart has long been somewhat tedious. When nothing else in configured, the methods for writing data to a document expects a `Map<String, dynamic>` and that is also the return type for the DocumentSnapshot etc. that you receive when getting data from Cloud Firestore. In order to leverage the type system, we don't want to deliver the firestore data as a random map to the rest of our application. We nned to map the maps to our domain data model types. To my knowledge, there are a couple of ways of accomplishing this.

## Example data model
Let's work with a really simple data model for this post
```dart
class Person {
    final String name;
    final int age;

    Person(this.name, this.age);
}
```

## "Naive, write your conversion"-way
I call it naive, but it doesn't necessarily have to be so in a bad way. It's simply the first, simplest approach that comes to mind. If we need a `Map` representing the class to give to firebase for creating a document, let's write the conversion ourselves.

```dart
class Person {
   ...

   Map<String, dynamic> toMap() => {
       "name": name,
       "age": age
   };
}
```
and conversely for converting from firestore to a model
```dart
class Person {
   ...

   factory Person.fromMap(Map<String, dynamic> map) => Person(
       map["name"]!,
       map["age"]!
   );
}
```
> Note the lack of error handling here. Coercing the type from the map access to a non-nullable type with `!` would generally be safe if you strictly create the documents with the same data model. But even models could change over time and if you e.g. make a previously non-nullable property nullable, then make sure to add null checks for that. Probably wrap the conversion in try-catch in order to fail gracefully.
{: .prompt-warning }

Then in our data repository we would do 

```dart
final person = Person("Sundar", 50);
final doc = await FirebaseFirestore.instance.collection("people").add(person.toMap());

final snapshot = await FirebaseFirestore.instance.collection("people").doc(doc.id).get();
final samePerson = Person.fromMap(snapshot.data());
```

## Outsource the to/fromMap methods
Maintaining these methonds gets really tedious as they grow so a simple extra layer to the basic approach is simply to use a code generation tool to automatically generate these methods, like [json_serializable](https://pub.dev/packages/json_serializable) or [freezed](https://pub.dev/packages/freezed). If we go with json_serializable we'll end up with a data class like this
```dart
@JsonSerializable()
class Person {
    final String name;
    final int age;

    Person(this.name, this.age)

    factory Person.fromMap(Map<String, dynamic> json) => _$PersonFromJson(json);
    Map<String, dynamic> toMap() => _$PersonToJson(this);
}
```
> When serializing for json, the serializer also expects a `Map<String, dynamic>` which makes it a no-brainer to also use this functionality for creating maps for Firestore.
{: .prompt-info }

You would then go about using firestore identically as in the last scenario.

## .withConverters()
This is another slight improvment layer on top of the previous ones. The firebase team acknowledged that this might be something that users do regularly and added a convenience method on `CollectionReference` called [.withConverter()](https://pub.dev/documentation/cloud_firestore/latest/cloud_firestore/CollectionReference/withConverter.html). The method takes to function parameters, `fromFirestore` and `toFirestore`. The fromFirestore expects a function whose job it is to convert from a `DocumentSnapshot<Map<String, dynamic>>` to a data model and toFirestore expects a function that converts a data model into a `Map<String, dynamic>`. A simple usage can be seen here 

```dart
final db = FirebaseFirestore.instance;
final peopleCollectionRef = db.collection("people").withConverters(
    fromFirestore: (snapshot, _) => Person.fromMap(snapshot.data()!),
    toFirestore: (model, _) => model.toMap(),
);

final person = Person("Larry", 49);

final doc = await peopleCollectionRef.add(person);
final samePerson = await peopleCollectionRef.doc(doc.id).get().then((s) => s.data());
```

As you can see, this makes both read and write operations type safe and you don't have to convert with each operation as long as you use the same CollectionReference. I however feel that saving the collectionRef as a field in a repository/data source feels a bit iffy. There are no clear guidelines on this to my knowledge, but I prefer to create a CollectionReference in each method that performs an operation in my repositories. In this case, the plain .withConverters() is a little verbose to my liking, so I tried to think of some abstraction. If you are not like me, feel free to use vanilla `.withConverters()` to your heart's content.

# Alternatives


## .withBetterConverters()
The abstraction I worked out was an extension method on CollectionReference which I like to call `.withBetterConverters()` even though it's barely a better solution at all, just different or more abstract. I wanted to hide the implementation of the function parameters `fromFirestore` and `toFirestore` and just pass the to/fromMap themselves. For `fromFirestore` that works well because `Person.fromMap()` is a statically available constructor, but `person.toMap()` is an instanstance method and only available for specific instances. This led me to create a mixin called `ToMapable` so we could always be certain that such an instance have a method called `toMap()`. Without further ado let's look at the code.

```dart
mixin ToMapable {
  Map<String, dynamic> toMap();
}

@JsonSerializable()
class Person with ToMapable{
    final String name;
    final int age;

    Person(this.name, this.age)

    factory Person.fromMap(Map<String, dynamic> json) => _$PersonFromJson(json);
    Map<String, dynamic> toMap() => _$PersonToJson(this);
}

extension BetterConverters on CollectionReference {
  CollectionReference<T> withBetterConverter<T extends ToMappable>({
    required T Function(Map<String, dynamic> map) fromFirestore,
  }) {
    return withConverter(
      fromFirestore: (snapshot, options) => fromFirestore(snapshot.data()!),
      toFirestore: (value, options) => value.toMap(),
    );
  }
}
```
This let's us hide the `toFirestore` completely within the implementation of `withBetterConverters()`, however now we have the the situation turned around. Since `Person.fromMap()` is statically available, we can't guarantee that with a mixin, so we can't completely hide that. There's possibly someway that I haven't figured out yet, but until that time, usage would look like this

```dart
final db = FirebaseFirestore.instance;
final peopleCollectionRef = db.collection("people").withBetterConverters(fromFirestore: Person.fromMap);

final person = Person("Sergey", 49);

final doc = await peopleCollectionRef.add(person);
final samePerson = await peopleCollectionRef.doc(doc.id).get().then((s) => s.data());
```
As you can see, it's not anything revolutionary, but definitely less verbose if you write a lot of collectionRefs.

## Cloud Firestore ODM
Another alternative could be the firebase official ODM package. However it's currently in alpha state, so I wouldn't touch it until it get to at least beta. It's actively developed, which is nice to see. It works by definining data schemas and provides features like data validation in both directions, type-safe queries, data binding and API code completion. I'm really looking forward to using this. You can find it [here](https://github.com/firebase/flutterfire/tree/master/packages/cloud_firestore_odm).

# Summary
While not rocket science, the BetterConverters provide a little bit of abstracion over the data conversion making each repository less verbose. The single easiest win right now however would be to incorporate code generation in your data models, in order to have an easier time maintaining the models as the the code base changes.

## Extra tip
I found this package for faking the firestore instance for ease of unit testing. I havent tested the package out myself, but I will definitely give it a spin for my next unit tests.
https://pub.dev/packages/fake_cloud_firestore

# Reading List
* [https://firebase.google.com/docs/firestore/manage-data/add-data#custom_objects](https://firebase.google.com/docs/firestore/manage-data/add-data#custom_objects)
* [https://pub.dev/packages/json_serializable](https://pub.dev/packages/json_serializable)
* [https://pub.dev/packages/freezed](https://pub.dev/packages/freezed)
* [https://pub.dev/documentation/cloud_firestore/latest/cloud_firestore/CollectionReference-class.html](https://pub.dev/documentation/cloud_firestore/latest/cloud_firestore/CollectionReference-class.html)