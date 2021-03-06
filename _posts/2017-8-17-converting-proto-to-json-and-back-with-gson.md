---
layout: "post"
title: "converting-proto-to-json-and-back-with-gson"
date: "2017-07-17 20:56"
---

# Converting Protocol Buffers data to Json and back with Gson Type Adapters :D

## Introduction
In this article, I will demonstrate to the reader a method to write a GSON Type Adapter for converting protobuf objects to and from JSON.

## Why write an adapter ?
GSON does not deserialize protobug objects out-of-the-box. There are no classes or utilities in GSON that convert data represent as protobuf data to JSON and vice versa.

However, we do have the option of using [JsonFormat](https://github.com/google/protobuf/blob/master/java/util/src/main/java/com/google/protobuf/util/JsonFormat.java) class which can be used for achieving our purpose.

## Why not use JsonFormat directly ?
In my case, my application was already reading data in JSON and deserializing it to the respective POJO's. However, a change in the direction of the project meant that new classes would have to be modeled using [Google's Protocol Buffers](https://developers.google.com/protocol-buffers/). Protobuf was great but it had the following shortcomings:
* Maintenance: Editing protobuf was difficult. There is a lack of editors that support plugins to read, display and edit protobuf data. In contrast, Editing JSON data was really simple. A lot of editors such as Atom provide plugins which made it easy to read and update JSON data.
* Program Design: Our app used a single GSON instance for our data. We needed a way to use that same GSON instance to read data represented as protocol buffers.


## Let the Example Begin :)

*Note this example uses Protobuf version 3.3.1 though the concepts are applicable for lower protobuf versions.*
*The source code for the example is available at [PersonProtoTutorial](https://github.com/moijes12/PersonProtoTutorial)*

Take the proto file defined below:

**Person.proto**
```
syntax = "proto2";

option java_package = "com.example.tutorial.proto";
option java_outer_classname = "PersonProtos";

package tutorial;

message Person {
    required string name = 1;
    optional string email = 2;
    optional int32 age = 3;
}
```

Running the *protoc* compiler on this proto would generate a class of the below form,

```java
// Generated by the protocol buffer compiler.  DO NOT EDIT!
// source: proto/Person.proto

package com.example.tutorial.proto;

public final class PersonProtos {
  ...
  /**
   * Protobuf type {@code tutorial.Person}
   */
  public static final class Person extends
      com.google.protobuf.GeneratedMessage implements
      // @@protoc_insertion_point(message_implements:tutorial.Person)
      PersonOrBuilder {
        ...
      }
}
```


This is the java definition of the proto and we will create a Person message in our demo class.

### Demo Class
Now, assume we use the proto in our demo class as below:

**PersonDemo.java**
```java
public class PersonDemo {
	public static void main(String[] args) {

    // Create a Person object
    Person.Builder personBuilder = Person.newBuilder();
    personBuilder.setName("John");
    personBuilder.setEmail("john@doe.com");
    personBuilder.setAge(24);
    Person p = personBuilder.build();

    // Print the Person proto string
    System.out.println(p);    // Output:
		              // name: "John"
			      // email: "john@doe.com"
			      // age: 24
    ...
  }
}
```

Now lets use the below Gson call in the demo class to print the Json form of the proto
```java
Gson gson = new Gson();
String personJson = gson.toJson(p);
```

Running the above line would cause the below error as Gson doesn't have a way to convert the proto to json.
```
Exception in thread "main" java.lang.IllegalArgumentException: class com.example.tutorial.proto.PersonProtos$Person declares multiple JSON fields named unknownFields
	at com.google.gson.internal.bind.ReflectiveTypeAdapterFactory.getBoundFields(ReflectiveTypeAdapterFactory.java:170)
	at com.google.gson.internal.bind.ReflectiveTypeAdapterFactory.create(ReflectiveTypeAdapterFactory.java:100)
	at com.google.gson.Gson.getAdapter(Gson.java:423)
	at com.google.gson.Gson.toJson(Gson.java:661)
	at com.google.gson.Gson.toJson(Gson.java:648)
	at com.google.gson.Gson.toJson(Gson.java:603)
	at com.google.gson.Gson.toJson(Gson.java:583)
	at com.example.tutorial.PersonDemo.main(PersonDemo.java:35)
```

Let us look at how a Type Adapter helps us here

### Type Adapter to convert from Proto to Json and vice versa

*A good explanation of Adapters can be found [here](http://www.javacreed.com/gson-typeadapter-example/)*

```java
package com.example.tutorial.adapter;
import java.io.IOException;

import com.example.tutorial.proto.PersonProtos.Person;
import com.google.gson.JsonParser;
import com.google.gson.TypeAdapter;
import com.google.gson.stream.JsonReader;
import com.google.gson.stream.JsonWriter;
import com.google.protobuf.util.JsonFormat;

/**
 * The PersonAdapter class which will provide a {@TypeAdapter} to convert Person proto
 * to a well defined, concise Json and vice-versa.
 *
 * @author moijes12
 *
 */
public class PersonAdapter extends TypeAdapter<Person> {

	/**
	 * Override the read method to return a {@Person} object from it's json representation.
	 */
	@Override
	public Person read(JsonReader jsonReader) throws IOException {
	        // Create a builder for the Person message
		Person.Builder personBuilder = Person.newBuilder();
		// Use the JsonFormat class to parse the json string into the builder object
		// The Json string will be parsed fromm the JsonReader object
		JsonParser jsonParser = new JsonParser();
		JsonFormat.parser().merge(jsonParser.parse(jsonReader).toString(), personBuilder);
		// Return the built Person message
		return personBuilder.build();
	}

	/**
	 * Override the write method and set the json value of the Person message.
	 */
	@Override
	public void write(JsonWriter jsonWriter, Person person) throws IOException {
		// Call the printer of the JsonFormat class to convert the Person proto message to Json
		jsonWriter.jsonValue(JsonFormat.printer().print(person));
	}
}
```

Using this adapter in our demo requires the Gson object to have the adapter registered as below

```java
GsonBuilder gsonBuilder = new GsonBuilder();
Gson gson = gsonBuilder.registerTypeAdapter(Person.class, new PersonAdapter()).create();
```    

So the below code in our demo class
```java
// Create a Person object
Person.Builder personBuilder = Person.newBuilder();
personBuilder.setName("John");
personBuilder.setEmail("john@doe.com");
personBuilder.setAge(24);
Person p = personBuilder.build();

// Get the Json representation
String personJson = gson.toJson(p);
System.out.println(personJson);

// Convert Json back to Protocol
Person personReconvertedFromJson = gson.fromJson(personJson, Person.class);
System.out.println(personReconvertedFromJson.toString());
```

Would produce as Output
```
{
  "name": "John",
  "email": "john@doe.com",
  "age": 24
}

name: "John"
email: "john@doe.com"
age: 24
```
