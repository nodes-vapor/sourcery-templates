# Sourcery Templates ✨
[![Swift Version](https://img.shields.io/badge/Swift-3-brightgreen.svg)](http://swift.org)
[![Vapor Version](https://img.shields.io/badge/Vapor-2-F6CBCA.svg)](http://vapor.codes)
[![codebeat badge](https://codebeat.co/badges/52c2f960-625c-4a63-ae63-52a24d747da1)](https://codebeat.co/projects/github-com-nodes-vapor-sourcery-templates)
[![Readme Score](http://readme-score-api.herokuapp.com/score.svg?url=https://github.com/nodes-vapor/sourcery-templates)](http://clayallsopp.github.io/readme-score?url=https://github.com/nodes-vapor/sourcery-templates)
[![GitHub license](https://img.shields.io/badge/license-MIT-blue.svg)](https://raw.githubusercontent.com/nodes-vapor/sourcery-templates/master/LICENSE)

[Sourcery](https://github.com/krzysztofzablocki/Sourcery) stencil files for generating Vapor 2 boilerplate.

## Introduction

Writing Vapor apps usually involves a lot of boilerplate code and since Vapor doesn't have any convenience for this in the current toolbox, we decided to look at metaprogramming to fix this issue. This repository contains the templates we have found useful to have in our toolbelt when developing Vapor projects. The overall guidelines for the templates are:

- It should be easy to opt-in and out of.
- It should be easy to mix and match between Sourcery-generated files and custom-made files.
- It should be very clear what has been generated by Sourcery to avoid manual edits that could potentially be overwritten when Sourcery is being run again.
- The template files should cover the common case, but not all cases. To avoid maintenance overload in this repo, we generally try to solve the general cases. Edge cases should be solved by opting out of Sourcery in the specific cases.

## Important notices

This repo contains a Sourcery configuration file (`.sourcery.yml`) which can be copied/moved/linked into the root of your project. Using this configuration all Sourcery-generated files will be created in `Sourcers/App/Generated/`, but please note that this `Generated` folder needs to be created manually before running Sourcery.

All Vapor related templates support a `import` and `imports` annotation that allows you to declare imports in the generated source. Please not that, due to a limitation in Stencil, `import` and `imports` have different behaviour. Use the `imports` annotation if you need to import more than one module.

Example:
```swift
// sourcery: imports = JWTKeychain, imports = Storage
final class User: Model {
    ...
```

## Table of Contents

- [Introduction](#introduction)
  - [Important notice](#important-notice)
- [Models](#models)
  - [Model](#model)
  - [Preparation](#preparation)
    - [Annotations](#annotations)
  - [RowConvertible](#rowconvertible)
    - [Annotations](#annotations-1)
  - [NodeRepresentable](#noderepresentable)
    - [Annotations](#annotations-2)
  - [JSONConvertible](#jsonconvertible)
    - [Annotations](#annotations-3)
- [Controllers](#controllers)
  - [RouteCollection](#routecollection)
    - [Annotations](#annotations-4)
- [Forms](#forms)
    - [Annotations](#annotations-5)
- [Tests](#tests)
  - [LinuxMain](#linuxmain)
    - [Annotations](#annotations-6)
- [General](#general)
  - [Enums](#enums)
    - [Annotations](#annotations-7)
- [Configuration](#configuration)
  - [Options](#options)
- [Credits](#credits)
- [License](#license)

## Models
This collection of templates is related to models and automating their conversions to common Vapor types. To make Sourcery pick up your model, annotate your model with `model` :

```swift
// sourcery: model
final class User: Model {
    // ...
}
```

### Model
Automatically generates an initializer and an enum for MySQL enum types.

Example:
```swift
// sourcery: model
final class User: Model {
    var name: String
    var age: Int?
}
```

Becomes:
```swift
// sourcery: model
final class User: Model {
    var name: String
    var age: Int?

// sourcery:inline:auto:User.Models
    let storage = Storage()


    internal init(
        name: String,
        age: Int? = nil
    ) {
        self.name = name
        self.age = age
    }
// sourcery:end
}
```

### Preparation
Generates a list of database keys, automates `prepare` and `revert` functions.

Example:
```swift
// sourcery: model
final class User: Model {
    var name: String
    var age: Int?
}
```

Becomes:
```swift
import Vapor
import Fluent

extension User: Preparation {
    internal enum DatabaseKeys {
        static let id = User.idKey
        static let name = "name"
        static let age = "age"
    }

    // MARK: - Preparations (User)
    internal static func prepare(_ database: Database) throws {
        try database.create(self) {
            $0.id()
            $0.string(DatabaseKeys.name)
            $0.int(DatabaseKeys.age, optional: true)
        }

    }

    internal static func revert(_ database: Database) throws {
        try database.delete(self)
    }
}

```

#### Annotations

| Key                 | Description                              |
| ------------------- | ---------------------------------------- |
| `databaseKey`       | Set the database key (default is the name of the member). |
| `preparation`       | Set the database preparation type for the given member. For example `preparation = string` will generate `$0.string(...)` |
| `length`            | Length of the field.                     |
| `unique`            | Whether or not the field is unique.      |
| `foreignTable`      | The table to use while configuring foreign ids. This field is only valid if `preparation` is set to `foreignId`. |
| `foreignIdKey`      | The foreign key to use while configuring foreign ids. |
| `foreignKeyName`    | The foreign key's name. _note_:  this annotation should only be used when you run into Fluent issues with the auto-generated names. |
| `ignore`            | Prevents the preparation from being included in the generated code. |
| `ignorePreparation` | Prevents the preparation from being included in the generated `Preparation` code. |

### RowConvertible
Automates `init (row: Row)` and `makeRow` boilerplate.

Example:
```swift
// sourcery: model
final class User: Model {
    var name: String
    var age: Int?
}
```

Becomes:
```swift
import Vapor
import Fluent

extension User: RowConvertible {
    // MARK: - RowConvertible (User)
    convenience internal init (row: Row) throws {
        try self.init(
            name: row.get(DatabaseKeys.name),
            age: row.get(DatabaseKeys.age)
        )
    }

    internal func makeRow() throws -> Row {
        var row = Row()

        try row.set(DatabaseKeys.name, name)
        try row.set(DatabaseKeys.age, age)

        return row
    }
}
```

#### Annotations

| Key                    | Description                              |
| ---------------------- | ---------------------------------------- |
| `databaseKey`          | Set the database key (default is the name of the member). |
| `ignore`               | Prevents the property from being included in the generated code. |
| `ignoreRowConvertible` | Prevents the property from being included in the generated `RowConvertible` code. |

### NodeRepresentable
Generates a list of node keys and `makeNode(in context: Context?)`.

Example:
```swift
// sourcery: model
final class User: Model {
    var name: String
    var age: Int?
}
```

Becomes:
```swift
import Vapor
import Fluent

extension User: NodeRepresentable {
    internal enum NodeKeys: String {
        case name
        case age
    }

    // MARK: - NodeRepresentable (User)
    func makeNode(in context: Context?) throws -> Node {
        var node = Node([:])

        try node.set(User.idKey, id)
        try node.set(NodeKeys.name.rawValue, name)
        try node.set(NodeKeys.age.rawValue, age)

        return node
    }
}
```

#### Annotations

| Key                       | Description                              |
| ------------------------- | ---------------------------------------- |
| `nodeKey`                 | Set the key for node (de)serialization.  |
| `ignore`                  | Prevents the property from being included in the generated code. |
| `ignoreNodeRepresentable` | Prevents the property from being included in the generated `NodeRepresentable` code. |

### JSONConvertible
Generates a list of JSON keys, `init(json: JSON)` and `makeJSON`.

Example:
```swift
// sourcery: model
final class User: Model {
    var name: String
    var age: Int?
}
```

Becomes:
```swift
extension User {
    internal enum JSONKeys {
        static let name = "name"
        static let age = "age"
    }
}

// MARK: - JSONInitializable (User)

extension AppUser: JSONInitializable {
    internal convenience init(json: JSON) throws {
        let name: String = try json.get(JSONKeys.name)
        let age: Int? = try json.get(JSONKeys.age)

        self.init(
            name: name,
            age: age
        )
    }
}

// MARK: - JSONRepresentable (User)

extension User: JSONRepresentable {
    internal func makeJSON() throws -> JSON {
        var json = JSON()

        try json.set(User.idKey, id)
        try json.set(JSONKeys.name, name)
        try json.set(JSONKeys.age, age)

        return json
    }
}

extension User: JSONConvertible {}

extension User: ResponseRepresentable {}
```

#### Annotations

| Key                       | Description                                                  |
| ------------------------- | ------------------------------------------------------------ |
| `jsonKey`                 | Set the key for JSON serialization.                          |
| `jsonValue`               | Set the value for JSON serialization.                        |
| `ignore`                  | Prevents the property from being included in the generated code. |
| `ignoreJSONConvertible`   | **On type:** Prevents any JSON-related code to be generated. **On property:** Prevents the property from being included in the generated `JSONConvertible` code. |
| `ignoreJSONInitializable` | **On type**: Prevents the `JSONInitializable` extension to be generated. **On property:** Prevents the property from being included in the `JSONInitializable` extension. |
| `ignoreJSONRepresentable` | **On type:** Prevents the `JSONRepresentable` extension to be generated. **On property:** Prevents the property from being included in the `JSONRepresentable` extension. |

## Controllers
These templates are for controllers and route collections. To make Sourcery pick up your controller, annotate your controller with `controller`:

```swift
// sourcery: controller
final class UserController {
	// ... 
}
```

### RouteCollection

Generates route collection for controller logic.

Example:

```swift
// sourcery: controller
// sourcery: group = users
final class UserController {
    // sourcery: route, method = get, path = /
    func index(_ req: Request) throws -> ResponseRepresentable {
        return try User.all().makeJSON()
    }

    // sourcery: route, method = get, path = /:userId
    func show(_ req: Request) throws -> ResponseRepresentable {
        let user: User = try req.parameters.next()
        return user
    }
}
```

Becomes:

```swift
extension UserController: RouteCollection {
    func build(_ builder: RouteBuilder) throws {
        builder.group("users") { routes in
            // GET /users/
            routes.get("/", handler: index)
            // GET /users/:userId
            routes.get("/:userId", handler: show)
        }
    }
}
```

#### Annotations

| Key      | Description                              |
| -------- | ---------------------------------------- |
| `group`  | Set the group prefix for all routes in the controller. |
| `route`  | Set the controller method to be generated as a route. This would require `method` and `path` to be set as well. |
| `method` | The method of the route.                 |
| `path`   | The path for the route.                  |

## Forms

This template makes working with [Forms](https://github.com/nodes-vapor/forms) more convenient by annotating them with "form".

| Note: Make sure to postfix your field properties with "Field" so that the generated value accessors don't conflict with the field names.

Example:

```swift
// sourcery: form
struct UserForm: Form {
    let nameField: FormField<String>
    let ageField: FormField<Int>

    ...
}
```

Becomes:

```swift
extension UserForm {
    var fields: [FieldType] {
        return [
            nameField,
            ageField
        ]
    }
}

extension UserForm {
    var name: String? {
        return nameField.value
    }
    var age: Int? {
        return ageField.value
    }
}

extension UserForm {
    internal func assertValues(errorOnNil: Error = Abort(.internalServerError)) throws -> (
        name: String,
        age: Int
    ) {
        guard
            let name = name,
            let age = age
        else {
            throw errorOnNil
        }

        return (
            name,
            age
        )
    }
}
```

### Annotations

| Key | Description |
| --- | --- |
| `form` | Opt-in a type conforming to `Form` to use this template. |
| `ignoreField` | Opt-out a specific field from all extensions |
| `ignoreAssertValues` | Opt-out a specific field from being included in the `assertValues` function. |

## Tests
These templates are related to unit testing with XCTest.

### LinuxMain
Generates a static `allTests` for every `XCTestCase` and registers them in the `LinuxMain.swift`.

Example:

```swift
class UserControllerTests: TestCase {
    func testShowOneUser() throws {}
    func testShowAllusers() throws {}
}
```

Becomes:

```swift
// sourcery:inline:auto:LinuxMain

extension UserControllerTests {
    static var allTests = [
      ("testShowOneUser", testShowOneUser),
      ("testShowAllusers", testShowAllusers),
    ]
}

XCTMain([
    testCase(UserControllerTests.allTests)
])

// sourcery:end
```

Please note that if you're using one of the official templates ([api](https://github.com/vapor/api-template/blob/master/Tests/AppTests/Utilities.swift#L24) or [web](https://github.com/vapor/web-template/blob/master/Tests/AppTests/Utilities.swift#L23)) then you would need to make sure that the definition of `TestCase` gets excluded from `LinuxMain.swift` using the `excludeFromLinuxMain` annotation.

#### Annotations

| Key                    | Description                              |
| ---------------------- | ---------------------------------------- |
| `excludeFromLinuxMain` | Prevents the test case from being included in the generated code. |

## General

These templates are not related to any specific part of your Vapor project.

### Enums

Generates convenient accessors for getting a list of all cases. The MySQL preparation will use this as well.

Example:

```swift
// sourcery: enum
enum TestEnum: String, RawStringConvertible {
    case a, b, c
}
```

Becomes:

```swift
extension TestEnum {
    static var all: [TestEnum] = [
        .a,
        .b,
        .c,
    ]

    static let allRaw = TestEnum.all.map { $0.rawValue }
}
```

#### Annotations

| Key      | Description                              |
| -------- | ---------------------------------------- |
| `enum`   | In the situation where enums has been globally turned off, this flag can be used to opt-in on specific enum types. |
| `ignore` | Prevents the enum from being included in the generated code. |

## Configuration

Most of our templates are handled on a opt-in basis per type, however we also have some configurations which can be handled on a global level.

### Options

| Key          | Description                              |
| ------------ | ---------------------------------------- |
| `enumOptOut` | Globally turns off generation of enum accessors. To enable for only some specific enum types, use the `enum` annotation. If this is not set all enums within the module will have convenience accessors created. |

## 🏆 Credits

This package is developed and maintained by the Vapor team at [Nodes](https://www.nodesagency.com).
The package owner for this project is [Steffen](https://github.com/steffendsommer).


## 📄 License

This package is open-sourced software licensed under the [MIT license](http://opensource.org/licenses/MIT).
