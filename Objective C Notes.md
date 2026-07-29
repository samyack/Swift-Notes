# Objective-C for Experienced Swift Developers
### A Complete Translation Reference for Working in Legacy Production Codebases

> **Who this is for:** iOS engineers who already think fluently in Swift — MVVM, closures, protocols, Codable, async/await, Auto Layout, Core Data — and now need to read, debug, and ship changes in a real Objective-C codebase without re-learning programming from scratch.
>
> **How to use this document:** It is organized as a reference, not a tutorial. Jump to any section. Every concept is anchored to the Swift equivalent you already know, then shown in Objective-C, then explained at the "why does it work this way" level so the syntax stops feeling arbitrary.

---

## Table of Contents

0. [Quick Conversion Cheat Sheet](#0-quick-conversion-cheat-sheet)
1. [Language Basics](#section-1-language-basics)
2. [Methods (Functions)](#section-2-methods-functions)
3. [Classes](#section-3-classes)
4. [Properties](#section-4-properties)
5. [Blocks (Closures)](#section-5-blocks-closures)
6. [Protocols & Delegates](#section-6-protocols--delegates)
7. [Control Flow](#section-7-control-flow)
8. [Collections](#section-8-collections)
9. [Memory Management](#section-9-memory-management)
10. [UIKit Development](#section-10-uikit-development)
11. [Networking](#section-11-networking)
12. [Concurrency](#section-12-concurrency)
13. [Notifications](#section-13-notifications)
14. [Core Data](#section-14-core-data)
15. [Error Handling](#section-15-error-handling)
16. [Categories & Extensions](#section-16-categories--extensions)
17. [The Objective-C Runtime](#section-17-the-objective-c-runtime)
18. [Common Cocoa Patterns](#section-18-common-cocoa-patterns)
19. [Swift ↔ Objective-C Interoperability](#section-19-swift--objective-c-interoperability)
20. [Daily Development Examples](#section-20-daily-development-examples)
21. [Debugging Objective-C](#section-21-debugging-objective-c)
22. [Reading Real Production Code](#section-22-reading-real-production-code)
23. [Common Mistakes Swift Developers Make](#section-23-common-mistakes-swift-developers-make)
24. [Mental Model: How to Think in Objective-C](#section-24-mental-model-how-to-think-in-objective-c)
25. [Cheat Sheets](#section-25-cheat-sheets)

---

## 0. Quick Conversion Cheat Sheet

Keep this tab open. Everything below is expanded later in the document.

| Swift | Objective-C | Notes |
|---|---|---|
| `let x = 5` | `const NSInteger x = 5;` or just `NSInteger const x = 5;` | ObjC `const` is rarer; most "constants" are just variables by convention |
| `var x = 5` | `NSInteger x = 5;` | no `var`/`let` distinction at the language level for locals |
| `struct` | *(no direct equivalent)* | ObjC has C `struct` (value type, no methods) but no first-class value-type objects |
| `class Foo {}` | `@interface Foo : NSObject @end` / `@implementation Foo @end` | declaration and implementation are separate files/blocks |
| `init()` | `- (instancetype)init` | must call `[super init]` and check for `nil` manually |
| `deinit` | `- (void)dealloc` | called by ARC when retain count hits zero |
| `extension Foo {}` | `@interface Foo (CategoryName) @end` | called a **Category** |
| `protocol Foo {}` | `@protocol Foo @end` | supports `@required`/`@optional` |
| `{ (x) in ... }` closure | `^(NSInteger x) { ... }` | called a **Block** |
| `enum Foo { case a, b }` | `typedef NS_ENUM(NSInteger, Foo) { FooA, FooB };` | no associated values, no methods |
| `switch` | `switch` | ObjC switch only works on integers/enums, not strings or arbitrary values |
| `guard let x = y else { return }` | `if (!y) { return; } id x = y;` | no `guard` keyword; early-return by hand |
| `nil` | `nil` (objects) / `NULL` (C pointers) / `Nil` (classes) | three different "nil"s |
| `String` | `NSString *` | mutable variant is `NSMutableString *` |
| `Array<T>` | `NSArray *` / `NSMutableArray *` | not generic by default (lightweight generics exist but are erased at runtime) |
| `Dictionary<K,V>` | `NSDictionary *` / `NSMutableDictionary *` | |
| `Set<T>` | `NSSet *` / `NSMutableSet *` | |
| `Bool` | `BOOL` | actually a signed char (`YES`/`NO`, values 1/0) |
| `Int` | `NSInteger` | platform-dependent width (32/64-bit), like `Int` |
| `Double` | `double` | plain C type |
| `Float` | `float` / `CGFloat` | `CGFloat` matches pointer width |
| `Any` | `id` | dynamically typed pointer to any Objective-C object |
| `AnyObject` | `id` | same as above; ObjC doesn't separate them |
| `Codable` | `NSCoding` / `NSSecureCoding` (or manual `NSJSONSerialization`) | no automatic synthesis |
| `async`/`await` | completion blocks (or `NSOperation`) | pre-Swift-concurrency style; iOS 13+ ObjC code can also use `dispatch_async` |
| `Task { }` | GCD (`dispatch_async`) / `NSOperationQueue` | |
| `throws` / `try` | `NSError **` out-parameter | errors are returned by reference, not thrown |
| `as?` / `as!` | `isKindOfClass:` + cast | no compiler-enforced downcasting |
| `??` (nil-coalescing) | `x ?: y` (ternary shorthand) | ObjC's `?:` omits the middle operand |
| `self` | `self` | same concept |
| `Self` / static dispatch | `[self class]` / `[ClassName class]` | dynamic dispatch is the default in ObjC |
| `super.method()` | `[super method]` | |
| `@objc` attribute | *(implicit — everything is "objc")* | Swift needs to opt in; ObjC classes are exposed by default |
| Optionals (`Int?`) | no optionals for primitives; objects are implicitly nullable via `nil` | `nullable`/`nonnull` annotations added later for Swift interop |
| Trailing closure syntax | last block parameter, still needs full syntax | no syntactic sugar |
| `if let` | manual nil-check | |
| Tuples | *(no equivalent)* | use a small struct, `NSDictionary`, or multiple out-parameters |
| Generics (`<T>`) | "lightweight generics" (`NSArray<NSString *> *`) | compile-time only, erased at runtime, mostly for Swift bridging clarity |
| `#selector(foo)` | `@selector(foo)` | ObjC selectors are a first-class runtime type (`SEL`) |
| `String(format:)` | `[NSString stringWithFormat:@"..."]` | uses C `printf`-style format specifiers |
| Memberwise init | none automatic | must write initializers by hand |
| `defer` | *(no equivalent)* | use `@try`/`@finally` or manual cleanup at each return path |

---
## Section 1: Language Basics

### What is it?
Objective-C is a strict superset of C, with an Smalltalk-style object system bolted on top. This is the single fact that explains almost every syntax oddity you'll hit: anything that looks "weird" is usually plain C, and anything that looks "object-oriented but different" is the Smalltalk messaging layer.

### Why Objective-C works this way
Swift was designed from scratch as a modern, safe, value-oriented language. Objective-C was designed in the early 1980s as *C + objects*, so it inherited C's manual memory model, C's primitive types, and C's preprocessor — then added `@`-prefixed keywords for anything object-related, so the compiler can tell "this is C" from "this is an object" at a glance.

### Core primitive types

| Concept | Swift | Objective-C | Notes |
|---|---|---|---|
| Signed integer | `Int` | `NSInteger` | typedef'd to `long` (64-bit) or `int` (32-bit) |
| Unsigned integer | `UInt` | `NSUInteger` | same width rules |
| Floating point | `Double` | `double` | plain C |
| Floating point | `Float` | `float` | plain C |
| Geometry float | `CGFloat` | `CGFloat` | matches pointer width, used throughout UIKit |
| Boolean | `Bool` | `BOOL` | typedef'd to `signed char`; values are `YES` (1) / `NO` (0) |
| Boxed number | *(bridging only)* | `NSNumber *` | an **object** wrapper around any scalar, needed when a scalar must go into an `NSArray`/`NSDictionary` |
| Character | `Character` | `unichar` / `NSString` (single char) | ObjC has no rich grapheme-cluster type |

```objc
NSInteger age = 27;
CGFloat width = 320.0;
BOOL isLoggedIn = YES;
double precise = 3.14159;
```

### Strings

```swift
let name: String = "Sam"
let greeting = "Hello, \(name)!"
```

```objc
NSString *name = @"Sam";
NSString *greeting = [NSString stringWithFormat:@"Hello, %@!", name];
```

- `@"..."` is an **object literal** — it creates an `NSString *`, not a C string (`char *`).
- `%@` is the format specifier for "any Objective-C object" (calls `-description` on it).
- String interpolation doesn't exist; you build strings with `stringWithFormat:` (which is just `printf` with object support) or by concatenating with `stringByAppendingString:`.

**Mutability is a separate class**, not a `var`/`let` distinction:

```objc
NSString *immutable = @"Sam";                       // can't be changed in place
NSMutableString *mutable = [NSMutableString stringWithString:@"Sam"];
[mutable appendString:@" Patil"];                    // mutable = "Sam Patil"
```

### Collections at a glance (full detail in Section 8)

```swift
let names: [String] = ["Sam", "Riya"]
var mutableNames: [String] = names
mutableNames.append("Dev")

let ages: [String: Int] = ["Sam": 27]
```

```objc
NSArray<NSString *> *names = @[@"Sam", @"Riya"];
NSMutableArray<NSString *> *mutableNames = [names mutableCopy];
[mutableNames addObject:@"Dev"];

NSDictionary<NSString *, NSNumber *> *ages = @{@"Sam": @27};
```

- `@[...]` and `@{...}` are **literal syntax**, a modern (2012+) shorthand for `[NSArray arrayWithObjects:...]` etc.
- The `<NSString *>` generic annotations are "lightweight generics" — they exist purely for compile-time checking and Swift-bridging clarity. At runtime, the array doesn't actually enforce or know its element type (type erasure).

### `nil`, `Nil`, `NULL`, and `id`

| Token | Means | Swift equivalent |
|---|---|---|
| `nil` | no object (pointer to an Objective-C instance) | `nil` on an optional object |
| `Nil` | no *class* (pointer to a `Class`) | rarely surfaces in Swift |
| `NULL` | no C pointer at all | rarely surfaces in Swift |
| `NSNull` | an actual **object** representing "null" — used inside collections, since `nil` can't be inserted into an `NSArray`/`NSDictionary` | closest thing is `Optional<Any>.none` boxed, rarely needed |

`id` is the ObjC equivalent of `Any`/`AnyObject`: a pointer to *any* Objective-C object, with no compile-time type information. Sending any message to an `id` compiles; it only fails at runtime if the object can't respond.

```objc
id thing = @"hello";     // could be anything
[thing someRandomMethod]; // compiles, crashes at runtime if not implemented
```

### `instancetype`, `self`, `super`

- `instancetype` — return type meaning "whatever concrete class this was called on," used almost exclusively as the return type of initializers and factory methods (Swift's equivalent is returning `Self` implicitly from `init`).
- `self` — same concept as Swift's `self`, the current instance, but also usable inside class methods to mean the current class.
- `super` — same concept as Swift's `super`.

### `typedef`, `NS_ENUM`, `NS_OPTIONS`

Swift enums have no ObjC equivalent for associated values or methods, but plain enumeration mapping is common:

```swift
enum Status: Int {
    case pending, active, closed
}
```

```objc
typedef NS_ENUM(NSInteger, Status) {
    StatusPending,
    StatusActive,
    StatusClosed
};
```

- `typedef` is plain C: "give this type a new name."
- `NS_ENUM(UnderlyingType, Name)` is an Apple macro that expands to a `typedef enum` plus some compiler attributes so Swift can bridge it cleanly as `Status.pending` etc.
- `NS_OPTIONS` is the same idea for **bitmask** enums (Swift's `OptionSet`):

```objc
typedef NS_OPTIONS(NSUInteger, UIViewAnimationOptions) {
    UIViewAnimationOptionLayoutSubviews   = 1 << 0,
    UIViewAnimationOptionAllowUserInteraction = 1 << 1,
    // ...
};
```

### `static`, `extern`, `const`, `#define`

| Keyword | Meaning | Swift equivalent |
|---|---|---|
| `static` (inside a function/file) | variable persists across calls / is private to the file | `static` stored property, or a private global |
| `extern` | "this variable/function is defined elsewhere, just declare its existence here" | not needed — Swift modules handle this automatically |
| `const` | value cannot be changed after initialization | `let` |
| `#define FOO 5` | preprocessor text substitution, happens **before compilation** | no equivalent — Swift has no preprocessor; use `let` or a compile-time flag |

```objc
static NSString *const kAPIBaseURL = @"https://api.example.com"; // file-private constant
extern NSString *const kSharedNotificationName;                   // declared in a header, defined in a .m
#define MAX_RETRY_COUNT 3                                         // avoid in modern code; prefer `static const`
```

> **Tip:** Modern Objective-C style guides (and Apple's own frameworks) discourage `#define` for constants because it has no type, no scope, and can't be debugged — prefer `static const`.

### Beginner mistakes
- Forgetting the `*` on object pointers (`NSString name` instead of `NSString *name`) — everything that's an object is *always* a pointer in ObjC.
- Using `==` to compare `NSString`/`NSArray` contents — like Swift's `===` (identity), `==` compares pointers, not values. Use `isEqualToString:`/`isEqual:`.
- Treating `BOOL` as a real boolean type in bitwise contexts — it's a `signed char`, so `BOOL x = 2;` is legal and `if (x)` is still true, but `x == YES` can silently fail if `x` isn't exactly `1`. Always test `if (x)` / `if (!x)`, never `if (x == YES)`.

### Simple explanation
Objective-C is "C, plus an `@`-prefixed vocabulary for talking to objects." Every scalar you already know from C/Swift is still there under a slightly different name (`NSInteger` instead of `Int`); everything object-related gets an `NS`-prefixed class and an `@` symbol somewhere nearby.

### Key takeaways
- Object types are always pointers (`NSString *`, never `NSString`).
- Mutability is expressed via separate classes (`NSArray` vs `NSMutableArray`), not `let`/`var`.
- `nil` for objects, `NULL` for C pointers, `Nil` for classes — three different nothings.
- `id` is Objective-C's `Any` — dynamically typed, resolved at runtime.

---
## Section 2: Methods (Functions)

### What is it?
Objective-C doesn't call functions on objects — it **sends messages** to them. This distinction matters: `[obj doThing]` doesn't compile to a direct call the way `obj.doThing()` does in Swift; it compiles to `objc_msgSend(obj, @selector(doThing))`, a dynamic dispatch through the runtime. This is *why* Objective-C method syntax looks so different — it's message-passing syntax borrowed from Smalltalk, not a "function call with weird brackets."

### Swift version
```swift
func greet(name: String) -> String {
    return "Hello, \(name)"
}

func greet(name: String, from city: String) -> String { ... }

greet(name: "Sam")
greet(name: "Sam", from: "Pune")
```

### Objective-C version
```objc
- (NSString *)greetWithName:(NSString *)name;
- (NSString *)greetWithName:(NSString *)name fromCity:(NSString *)city;

[self greetWithName:@"Sam"];
[self greetWithName:@"Sam" fromCity:@"Pune"];
```

### Side-by-side: what every symbol means

| Symbol | Meaning |
|---|---|
| `-` | this is an **instance method** (called on an object) |
| `+` | this is a **class method** (called on the class itself, like `static func` in Swift) |
| `(NSString *)` right after `-`/`+` | the **return type** |
| `greetWithName:` | the first **selector piece** — note it already includes the first parameter's label baked into the method name |
| `(NSString *)name` | the **parameter type** and local variable name for that piece |
| `fromCity:` | second selector piece — ObjC method names are literally built by concatenating every labeled piece |
| `;` | ends the declaration, same as C |

The **full method name** (its *selector*) is `greetWithName:fromCity:` — the colons are part of the name. This is why you can't have two ObjC methods with the same first piece but different parameter labels the way you can overload in Swift; the whole label sequence *is* the identity of the method.

### Why the syntax looks the way it does
- **Why `-` and `+`?** Borrowed directly from Smalltalk to visually distinguish instance vs. class-level behavior at a glance, before you even read the rest of the line.
- **Why colons?** Each colon marks "a parameter follows here," and the label before it reads like natural language: `insertObject:atIndex:` reads almost like an English sentence once you have both arguments filled in: `[array insertObject:obj atIndex:0]`.
- **Why the `*`?** Every Objective-C object is manipulated through a pointer — there's no "value inline" object type the way Swift structs work in memory. Primitives (`NSInteger`, `BOOL`, `CGFloat`) don't get a `*` because they aren't objects.
- **Why parentheses around the return type and parameter types?** Plain C cast/type syntax — Objective-C reused C's existing type-in-parens grammar rather than inventing a new one.

### Method naming conventions
Objective-C method names are meant to be read as a sentence, and Apple's naming guidelines are stricter than Swift's:

```objc
- (void)insertObject:(id)object atIndex:(NSUInteger)index;
- (BOOL)isEqual:(id)object;
- (instancetype)initWithName:(NSString *)name age:(NSInteger)age;
```

- Prefixes like `is` signal a boolean-returning method.
– `init...` always starts an initializer.
- No parameter → no colon: `- (void)reloadData;` (Swift: `func reloadData()`).

### Selectors as values
A selector (`SEL`) is a first-class runtime value representing "the name of a method," independent of any implementation — used heavily for target-action and `respondsToSelector:`:

```swift
button.addTarget(self, action: #selector(buttonTapped), for: .touchUpInside)
```

```objc
[button addTarget:self action:@selector(buttonTapped) forControlEvents:UIControlEventTouchUpInside];
```

`@selector(buttonTapped)` produces a `SEL` — a lightweight, interned identifier the runtime can look up later. This is conceptually similar to Swift's `#selector`, which is literally implemented as a bridge to this same mechanism.

### "Method overloading" differences
Swift lets you overload by parameter type:
```swift
func process(_ value: Int) { }
func process(_ value: String) { }
```
Objective-C **cannot overload by type** with the same selector — the compiler picks a method by *name* (selector), not by argument types, since dispatch is resolved at runtime, not compile time. You must give overloads different names:
```objc
- (void)processInteger:(NSInteger)value;
- (void)processString:(NSString *)value;
```

### Factory methods vs. initializers
```swift
let date = Date()
let url = URL(string: "https://example.com")
```
```objc
NSDate *date = [NSDate date];                              // class (factory) method
NSURL *url = [NSURL URLWithString:@"https://example.com"];  // class (factory) method
NSURL *url2 = [[NSURL alloc] initWithString:@"https://example.com"]; // alloc/init pattern
```
Factory methods (`+ (instancetype)urlWithString:`) return an already-`autorelease`d, ready-to-use instance without you calling `alloc` yourself — convenient shorthand over the more verbose `alloc]/[init...]` two-step.

### Common mistakes
- Trying to call a method with the wrong argument order — in ObjC, argument order is baked into the selector, so `insertObject:atIndex:` and a hypothetical `atIndex:insertObject:` are two entirely different methods, not the same one reordered.
- Forgetting that a method with zero parameters still needs to be sent as a message: `[view reloadData]`, not `view.reloadData()` — although modern Objective-C **does** support dot syntax for properties (see Section 4), not for arbitrary method calls with side effects/parameters.
- Assuming compiler-time overload resolution exists — it doesn't; give differently-typed operations distinct names.

### Interview questions
- "What is a selector, and how does it differ from a function pointer?"
- "Why can't Objective-C overload methods purely by argument type?"
- "What does `objc_msgSend` do, and when would dispatch bypass it?" (see Section 17)

### Simple explanation
A Swift function call is "call this code directly." An Objective-C method call is "send this named request to this object and let it figure out at runtime what to do." The square-bracket syntax `[receiver message:arg]` is just "send `message:arg` to `receiver`."

### Key takeaways
- `-` = instance method, `+` = class/static method.
- The full method name includes every colon-terminated label — that's the selector.
- No compile-time overloading by type; give differently-typed methods differently-named selectors.
- `@selector(name)` produces a runtime handle to a method name, used for target-action and reflection-like APIs.

---
## Section 3: Classes

### What is it?
An Objective-C class is split into two pieces: an **interface** (`@interface`, usually in a `.h` header) declaring what the class exposes, and an **implementation** (`@implementation`, in a `.m` file) containing the actual code. Swift merges both into a single `class { }` block — Objective-C's split exists because C compiles file-by-file, and the header is how other files learn what your class can do without seeing its internals.

### Swift version
```swift
class Person {
    var name: String
    var age: Int

    init(name: String, age: Int) {
        self.name = name
        self.age = age
    }

    func greet() -> String {
        return "Hi, I'm \(name)"
    }
}
```

### Objective-C version

**Person.h**
```objc
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@interface Person : NSObject

@property (nonatomic, copy) NSString *name;
@property (nonatomic, assign) NSInteger age;

- (instancetype)initWithName:(NSString *)name age:(NSInteger)age;
- (NSString *)greet;

@end

NS_ASSUME_NONNULL_END
```

**Person.m**
```objc
#import "Person.h"

@implementation Person

- (instancetype)initWithName:(NSString *)name age:(NSInteger)age {
    self = [super init];
    if (self) {
        _name = [name copy];
        _age = age;
    }
    return self;
}

- (NSString *)greet {
    return [NSString stringWithFormat:@"Hi, I'm %@", self.name];
}

@end
```

### Side-by-side comparison

| Concept | Swift | Objective-C |
|---|---|---|
| Class declaration | `class Person { ... }` (single block) | `@interface Person : NSObject ... @end` (header) + `@implementation Person ... @end` (.m) |
| Superclass | `: Superclass` | `: Superclass` (Objective-C classes almost always ultimately inherit from `NSObject`) |
| Stored property | `var name: String` | `@property (nonatomic, copy) NSString *name;` declares it; backing ivar `_name` is auto-synthesized |
| Init | `init(...) { self.x = x }` | `- (instancetype)initWithX:(...) { self = [super init]; if (self) { _x = x; } return self; }` |
| Deinit | `deinit { }` | `- (void)dealloc { }` |

### Why Objective-C works this way
- **Why split header/implementation?** C compiles one file at a time; a header lets other `.m` files know a class's *shape* without recompiling or even seeing its source.
- **Why `self = [super init]; if (self) { ... }`?** `[super init]` can theoretically return a *different* object than the one you started with (rare, but part of the contract — e.g., class clusters like `NSString` return a different concrete subclass). The `if (self)` check guards against `super`'s initializer returning `nil` on failure, so you don't try to configure a null object.
- **Why `return self` at the end of init?** Objective-C initializers are still ordinary methods; they must explicitly return the constructed instance (or `nil` on failure) because there's no compiler-generated implicit return the way Swift's `init` has.

### Designated vs. convenience initializers
```swift
class Vehicle {
    let wheels: Int
    init(wheels: Int) { self.wheels = wheels }         // designated
    convenience init() { self.init(wheels: 4) }        // convenience
}
```
```objc
@interface Vehicle : NSObject
@property (nonatomic, readonly) NSInteger wheels;
- (instancetype)initWithWheels:(NSInteger)wheels NS_DESIGNATED_INITIALIZER; // designated
- (instancetype)init;                                                       // convenience
@end

@implementation Vehicle
- (instancetype)initWithWheels:(NSInteger)wheels {
    self = [super init];
    if (self) { _wheels = wheels; }
    return self;
}
- (instancetype)init {
    return [self initWithWheels:4];   // convenience calls designated
}
@end
```
`NS_DESIGNATED_INITIALIZER` is a compiler annotation (not a hard language rule the way Swift enforces it) that tells the compiler to warn you if some other initializer doesn't eventually funnel into it — the same "designated vs. convenience" contract Swift enforces natively, just opt-in and advisory in ObjC.

### `dealloc`
```objc
- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
    // no need to nil-out properties manually under ARC — it does that for you
}
```
This is Objective-C's `deinit`. Under ARC it's mostly used to unregister observers/delegates and stop timers — memory itself is released automatically.

### Categories, class extensions, forward declarations
```swift
extension Person {
    func isAdult() -> Bool { age >= 18 }
}
```
```objc
// Category — adds methods, declared/implemented anywhere, visible to all importers
@interface Person (Adulthood)
- (BOOL)isAdult;
@end

@implementation Person (Adulthood)
- (BOOL)isAdult { return self.age >= 18; }
@end

// Class extension — an *anonymous* category, used to declare "private" members
// only visible inside the .m file that implements the class
@interface Person ()
@property (nonatomic, strong) NSDate *lastLoginDate; // private property
@end
```
A **forward declaration** (`@class Person;`) tells the compiler "this type exists, I'll fill in details later" — used in headers to avoid `#import`ing a full class definition just to reference a pointer to it, which speeds up compilation and avoids circular-import problems.

```objc
@class Address; // forward declaration — no need to #import Address.h here

@interface Person : NSObject
@property (nonatomic, strong) Address *homeAddress;
@end
```

### `#import` vs `@class`
- `#import "Person.h"` pulls in the **entire** header (like Swift's implicit module visibility, but explicit and file-based).
- `@class Person;` only tells the compiler "a class named `Person` exists" — enough to declare a pointer (`Person *`), not enough to call its methods or access its properties. The full `#import` goes in the `.m` file where you actually use the class.

### Common mistakes
- Forgetting `self = [super init]` and just writing `[super init]` — you lose the (possibly different) object super constructs.
- Declaring a property in `@interface` in the header (public) when it should be private — use a class extension in the `.m` instead.
- Circular `#import`s between two headers — fix with `@class` forward declarations.

### Interview questions
- "Why does an Objective-C initializer reassign `self`?"
- "What's the difference between a category and a class extension?"
- "When would you use `@class` instead of `#import`?"

### Simple explanation
Think of `@interface` as the Swift class's "public signature card" and `@implementation` as its actual body — two files that together add up to one Swift `class` block.

### Key takeaways
- Header (`.h`) = what the class exposes; implementation (`.m`) = how it works.
- Initializers must manually reassign and check `self`, then return it.
- Categories = retroactive extensions visible everywhere; class extensions (`@interface ClassName ()`) = private additions visible only within the `.m`.
- `@class` = lightweight forward declaration to avoid unnecessary full imports.

---
## Section 4: Properties

### What is it?
`@property` is a compiler directive that auto-generates a getter, a setter, and a backing instance variable (ivar) for you — conceptually like Swift's stored properties, except every property must declare its **memory-management behavior** and **thread-access behavior** explicitly, because Objective-C has no compiler-enforced value semantics to fall back on.

### Swift version
```swift
class Profile {
    var name: String              // strong reference by default
    weak var delegate: ProfileDelegate?
    let id: String                // immutable
}
```

### Objective-C version
```objc
@interface Profile : NSObject
@property (nonatomic, copy) NSString *name;
@property (nonatomic, weak) id<ProfileDelegate> delegate;
@property (nonatomic, readonly) NSString *identifier;
@end
```

### Property attributes, explained one by one

| Attribute | Meaning | Swift equivalent |
|---|---|---|
| `strong` | keeps a strong (owning) reference; default for object properties | plain `var`/`let` of a class type |
| `weak` | non-owning reference, automatically set to `nil` when the referent deallocates | `weak var` |
| `assign` | non-owning reference with **no** automatic nil-ing (used for primitives and, historically, unsafe object refs) | closest to `unowned(unsafe)`, but normally just used for scalars like `NSInteger`/`BOOL` |
| `copy` | stores an **immutable copy** of the assigned value rather than the original object — critical for `NSString`/blocks/collections, since a caller could hand you a *mutable* subclass and mutate it out from under you later | Swift value types copy automatically; `copy` reproduces that safety for reference types |
| `retain` | old (pre-ARC / manual reference-counting era) name for `strong` — you'll still see it in legacy code | `strong` |
| `atomic` | getter/setter are wrapped in a lock, making individual reads/writes thread-safe (but **not** compound operations) — this is the **default** if you don't specify | no direct Swift equivalent — Swift properties aren't atomic by default |
| `nonatomic` | no locking around the getter/setter — faster, but not inherently thread-safe; the overwhelming convention in real iOS code | closest to Swift's default (no locking) |
| `readonly` | only a getter is synthesized; no setter | `let`, or `private(set) var` |
| `readwrite` | both getter and setter (default if omitted) | plain `var` |

### `atomic` vs `nonatomic`, precisely
`atomic` guarantees that a **single** get or set won't be corrupted by a concurrent get/set on another thread. It does **not** make compound operations (`array.count`, then separately `[array addObject:]`) safe, and does **not** prevent race conditions across multiple property accesses. Because of this limited guarantee and its real performance cost, `nonatomic` is used almost everywhere in practice, with actual thread-safety handled by explicit synchronization (locks, serial queues) when needed.

### `strong`/`weak`/`assign`/`copy` — when to use each

```objc
@property (nonatomic, strong) UIView *containerView;     // owns the view
@property (nonatomic, weak) IBOutlet UILabel *titleLabel; // outlets: weak, view hierarchy already owns it
@property (nonatomic, weak) id<SomeDelegate> delegate;    // delegates: always weak, avoids retain cycles
@property (nonatomic, copy) NSString *name;               // strings: always copy
@property (nonatomic, copy) void (^completion)(BOOL);     // blocks: always copy (see Section 5)
@property (nonatomic, assign) NSInteger count;             // scalars: assign (no ownership concept applies)
@property (nonatomic, assign) CGRect frame;                 // structs: assign
```

- **Delegates are `weak`** for the same reason Swift delegate properties are `weak`: the delegate (often a parent view controller) typically owns *you*, so a strong back-reference would create a retain cycle.
- **Strings and blocks are `copy`**, not `strong`, because both can have hidden **mutable** subclasses (`NSMutableString`, or a block referencing mutable captured state); copying locks in the value/behavior at assignment time so it can't change under you later.
- **IBOutlets are `weak`** by Apple convention since Storyboard/XIB view hierarchies already retain their subviews strongly; the outlet is just a convenient reference.

### `@synthesize` and instance variables
```objc
@interface Person : NSObject
@property (nonatomic, copy) NSString *name;
@end

@implementation Person
@synthesize name = _name;   // explicit — rarely needed today, auto-generated since Xcode 4.4+
- (void)printName {
    NSLog(@"%@", _name);    // direct ivar access — bypasses the getter (no atomic lock, no KVO notification)
    NSLog(@"%@", self.name); // property access — goes through the synthesized getter
}
@end
```
Modern Objective-C **auto-synthesizes** both the ivar (`_name`) and the accessor methods just from the `@property` line — `@synthesize` is legacy boilerplate you'll mostly see in older codebases or when a custom ivar name is wanted.

### Getter/setter customization
```swift
var isValid: Bool {
    return !name.isEmpty
}
```
```objc
@property (nonatomic, readonly, getter=isValid) BOOL valid;

- (BOOL)isValid {
    return self.name.length > 0;
}
```
`getter=isValid` renames the accessor (idiomatic for `BOOL` properties, which read more naturally as `isValid` than `valid`). You can similarly override `setter=` for custom setter names, though this is rarer.

### Memory behavior / ARC interaction
Every `strong` property retains its object (increments the retain count); every `weak` property holds a special zeroing reference that the runtime automatically nils out when the object deallocates. This is exactly the guarantee Swift's `weak var` gives you — Objective-C just requires you to spell out `strong`/`weak`/`assign`/`copy` per-property instead of inferring it from context.

### Common mistakes
- Declaring a delegate as `strong` instead of `weak` → retain cycle (view controller ↔ its child never deallocate).
- Declaring an `NSString` property as `strong` instead of `copy` → a caller can pass a mutable string and change it behind your back later.
- Forgetting `nonatomic` and taking the atomic-by-default performance hit with no actual safety benefit, since compound operations still aren't protected.

### Interview questions
- "What's the difference between `strong` and `copy` for an `NSString` property, and why does it matter?"
- "Why is `atomic` the default, and why do most codebases override it with `nonatomic`?"
- "What happens when a `weak` property's referent is deallocated?"

### Simple explanation
`@property` = Swift's `var`, but you must state out loud whether you own the value (`strong`), just observe it (`weak`), copy it (`copy`), or treat it as a plain scalar (`assign`) — Swift makes these choices for you based on whether it's a class or a struct; Objective-C makes you say it every time.

### Key takeaways
- `strong`/`weak`/`assign`/`copy` = explicit ownership, since ObjC has no value-type/reference-type distinction to infer it from.
- `nonatomic` is the near-universal real-world default; `atomic` is a weaker guarantee than people assume.
- Delegates: `weak`. Strings & blocks: `copy`. Scalars/structs: `assign`. Owned objects: `strong`.
- Direct ivar access (`_name`) skips the getter/setter — used inside `init`/`dealloc`, avoided elsewhere.

---
## Section 5: Blocks (Closures)

This is one of the biggest and most important chapters — blocks are everywhere in legacy Objective-C code (networking, animations, GCD, collection enumeration), and their syntax is the single thing that trips up Swift developers the most.

### What is it?
A block is Objective-C's closure: an inline, anonymous chunk of code that captures variables from its surrounding scope, can be stored, passed around, and invoked later — exactly like a Swift closure. The syntax, however, is famously unreadable until you learn the pattern once.

### The core syntax pattern

```
return_type (^block_name)(parameter_types) = ^return_type(parameters) { body };
```

The `^` is the "this is a block" marker — think of it as ObjC's version of Swift's `{ }` closure delimiters.

### Swift → Block mapping

| Swift | Objective-C |
|---|---|
| `{ () -> Void in print("hi") }` | `^{ NSLog(@"hi"); }` |
| `{ (x: Int) -> Void in print(x) }` | `^(NSInteger x) { NSLog(@"%ld", (long)x); }` |
| `let block: () -> Void = { print("hi") }` | `void (^block)(void) = ^{ NSLog(@"hi"); };` |
| `let adder: (Int, Int) -> Int = { $0 + $1 }` | `NSInteger (^adder)(NSInteger, NSInteger) = ^(NSInteger a, NSInteger b) { return a + b; };` |
| calling: `block()` | calling: `block();` |
| `@escaping` closure param | block param with no special marker needed *(all blocks can escape unless marked otherwise)* |
| Trailing closure: `foo() { ... }` | no trailing syntax — block is just the last argument, written in full |

### Declaring a block type with `typedef` (very common in real code)

```swift
typealias CompletionHandler = (Bool, Error?) -> Void

func fetchData(completion: @escaping CompletionHandler) { ... }
```

```objc
typedef void (^CompletionHandler)(BOOL success, NSError * _Nullable error);

- (void)fetchDataWithCompletion:(CompletionHandler)completion;
```

Reading this: `CompletionHandler` is now a **type name** for "a block that takes a `BOOL` and an `NSError *` and returns nothing" — exactly like declaring the Swift `typealias` above it.

### A full example — URLSession-style network call

```swift
func fetchUser(completion: @escaping (User?, Error?) -> Void) {
    URLSession.shared.dataTask(with: url) { data, response, error in
        if let error = error {
            completion(nil, error)
            return
        }
        let user = try? JSONDecoder().decode(User.self, from: data!)
        completion(user, nil)
    }.resume()
}
```

```objc
- (void)fetchUserWithCompletion:(void (^)(User * _Nullable user, NSError * _Nullable error))completion {
    NSURLSessionDataTask *task = [[NSURLSession sharedSession]
        dataTaskWithURL:url
        completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
            if (error) {
                completion(nil, error);
                return;
            }
            User *user = [User userFromData:data];
            completion(user, nil);
        }];
    [task resume];
}
```

### Capturing variables — `__block`, `__weak`, `__strong`

By default, a block **captures variables by value** (a snapshot at block-creation time) for primitives, and **captures a strong reference** for objects — similar to how a Swift closure captures.

```swift
var counter = 0
let increment = { counter += 1 }   // Swift closures capture by reference automatically for vars
```

Objective-C's default capture of a primitive is a **read-only snapshot** — you can't mutate a captured local scalar inside a block unless you mark it `__block`:

```objc
__block NSInteger counter = 0;
void (^increment)(void) = ^{
    counter++;    // only legal because of __block
};
increment();
NSLog(@"%ld", (long)counter); // 1
```

`__block` is the storage-class modifier that says "this variable is shared between the block and the enclosing scope," roughly matching Swift's default reference-style capture of `var` locals.

### Retain cycles and `__weak self`

The single most common Objective-C memory bug: a block stored as a property (or held onto past the current scope) that captures `self` strongly, while `self` also strongly holds the block — a retain cycle, exactly like Swift's `self` capture problem that `[weak self]` solves.

```swift
someObject.completion = { [weak self] result in
    guard let self = self else { return }
    self.handle(result)
}
```

```objc
__weak typeof(self) weakSelf = self;
someObject.completion = ^(id result) {
    __strong typeof(self) strongSelf = weakSelf;   // re-promote to strong for the duration of the block
    if (!strongSelf) { return; }
    [strongSelf handleResult:result];
};
```

- `__weak typeof(self) weakSelf = self;` — the exact equivalent of Swift's `[weak self]` capture list entry.
- `__strong typeof(self) strongSelf = weakSelf;` — the exact equivalent of Swift's `guard let self = self`: re-promoting the weak reference to a strong local for the duration of the block body, so `self` can't deallocate out from under you mid-execution.
- Skipping the `__strong` re-promotion and using `weakSelf` directly throughout the block risks `self` deallocating between two uses inside the same block — subtle, intermittent crashes.

### Blocks as properties — always `copy`

```objc
@property (nonatomic, copy) void (^completionHandler)(BOOL success);
```
Blocks are technically allocated on the **stack** when first created; `copy` forces Objective-C to move the block (and everything it captured) to the **heap**, so it survives past the scope where it was defined. This was historically critical pre-ARC and is still the required attribute by convention — using `strong` instead of `copy` often works today (ARC copies blocks automatically in most cases) but `copy` remains the documented, correct, and expected attribute.

### Nested blocks and dispatch blocks

```objc
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    // background work
    UIImage *processed = [self processImage:image];
    dispatch_async(dispatch_get_main_queue(), ^{
        // back on main thread
        self.imageView.image = processed;
    });
});
```
This is the direct ancestor of Swift's:
```swift
DispatchQueue.global().async {
    let processed = self.processImage(image)
    DispatchQueue.main.async {
        self.imageView.image = processed
    }
}
```

### UIView animation blocks

```swift
UIView.animate(withDuration: 0.3) {
    self.view.alpha = 0
}
```
```objc
[UIView animateWithDuration:0.3 animations:^{
    self.view.alpha = 0;
}];
```

### Common mistakes
- Forgetting `__block` on a captured local you intend to mutate — code fails to compile with "variable is not assignable."
- Storing a block as a property with `strong`/`assign` instead of `copy` in legacy (pre-ARC-aware) code, risking a dangling stack block.
- Capturing `self` strongly inside a block stored on a property owned (directly or indirectly) by `self` — the classic retain cycle.
- Using `weakSelf` directly across multiple statements inside a block instead of re-promoting to `strongSelf` once at the top — risk of `self` disappearing mid-block.

### Interview questions
- "Why does Objective-C need `__block`, when Swift doesn't need an equivalent keyword for closures capturing `var`?"
- "Walk through why a block stored as a property needs `copy`."
- "How would you avoid a retain cycle when a block captures `self`?"

### Simple explanation
A block is just a Swift closure wearing a `^` instead of curly braces at the front. `__block` = "let me mutate this captured variable." `__weak self` + `__strong self` inside the block = exactly Swift's `[weak self]` + `guard let self`.

### Key takeaways
- `^` marks a block, the same conceptual role as Swift's closure `{ }`.
- Capture is by-value for scalars unless marked `__block`; objects are captured strongly by default.
- Block properties are declared `copy` to force heap allocation.
- `__weak self` / `__strong self` inside a block is the direct equivalent of Swift's `[weak self]` / `guard let self`.

---
## Section 6: Protocols & Delegates

### What is it?
Objective-C protocols map almost one-to-one onto Swift protocols — the main new concept is that ObjC protocols can mark individual methods `@required` or `@optional`, since Objective-C predates Swift's protocol extensions and default implementations.

### Swift version
```swift
protocol ProfileDelegate: AnyObject {
    func profileDidUpdate(_ profile: Profile)
    func profileDidFail(_ profile: Profile, error: Error)
}

class ViewController: ProfileDelegate {
    func profileDidUpdate(_ profile: Profile) { ... }
    func profileDidFail(_ profile: Profile, error: Error) { ... }
}
```

### Objective-C version
```objc
@protocol ProfileDelegate <NSObject>

@required
- (void)profileDidUpdate:(Profile *)profile;

@optional
- (void)profile:(Profile *)profile didFailWithError:(NSError *)error;

@end

@interface Profile : NSObject
@property (nonatomic, weak) id<ProfileDelegate> delegate;
@end

@interface ViewController : UIViewController <ProfileDelegate>
@end

@implementation ViewController
- (void)profileDidUpdate:(Profile *)profile { ... }
@end
```

### Reading `id<ProfileDelegate>`
`id<ProfileDelegate>` means "a pointer to any object, as long as it conforms to `ProfileDelegate`" — the direct equivalent of Swift's `any ProfileDelegate` (or historically, just `ProfileDelegate` as an existential type). The `<...>` after a type is protocol conformance syntax, distinct from generics' `<...>` — context (whether it follows `id`/a class name vs. a collection) tells them apart.

### `@required` vs `@optional`
- `@required` (the default if unspecified) — conforming classes **must** implement it, or the compiler warns.
- `@optional` — conforming classes may skip it; **callers must check** `respondsToSelector:` before calling, since there's no default implementation mechanism the way Swift protocol extensions provide.

```objc
if ([self.delegate respondsToSelector:@selector(profile:didFailWithError:)]) {
    [self.delegate profile:self didFailWithError:error];
}
```
This `respondsToSelector:` check is the ObjC-era equivalent of Swift's optional protocol requirements combined with `?` calling syntax (`delegate?.profileDidFail?(...)` in the `@objc` protocol case).

### `UITableViewDelegate` / `UITableViewDataSource` — the classic example

```objc
@interface MyViewController : UIViewController <UITableViewDataSource, UITableViewDelegate>
@end

@implementation MyViewController

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return self.items.count;
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"Cell" forIndexPath:indexPath];
    cell.textLabel.text = self.items[indexPath.row];
    return cell;
}

- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    [tableView deselectRowAtIndexPath:indexPath animated:YES];
}

@end
```
This is exactly the same conceptual contract as Swift's `UITableViewDataSource`/`UITableViewDelegate` conformance — same method names underneath (they're the same Objective-C runtime methods Swift is bridging to), just with ObjC's colon-based selector spelling.

### Custom delegate pattern, end to end

```objc
// Delegate protocol declared alongside the class that uses it
@protocol DownloadManagerDelegate <NSObject>
- (void)downloadManager:(DownloadManager *)manager didFinishWithData:(NSData *)data;
- (void)downloadManager:(DownloadManager *)manager didFailWithError:(NSError *)error;
@end

@interface DownloadManager : NSObject
@property (nonatomic, weak) id<DownloadManagerDelegate> delegate;
- (void)startDownload;
@end

@implementation DownloadManager
- (void)startDownload {
    // ... on success:
    [self.delegate downloadManager:self didFinishWithData:data];
    // ... on failure:
    [self.delegate downloadManager:self didFailWithError:error];
}
@end
```

### Why delegates are always `weak`
The pattern is almost always: parent (e.g., a view controller) owns child (e.g., `DownloadManager`), and child needs to talk back to parent. If child held `delegate` as `strong`, you'd get parent → strong → child → strong → parent, a retain cycle neither can escape. `weak` breaks the cycle — identical reasoning to Swift's `weak var delegate`.

### Common mistakes
- Forgetting to check `respondsToSelector:` before calling an `@optional` method — crashes with "unrecognized selector" if the delegate hasn't implemented it.
- Declaring `delegate` as `strong` — retain cycle.
- Confusing protocol conformance syntax `<Protocol>` with generics syntax `<Type>` — they look identical but mean different things depending on context.

### Interview questions
- "How does an Objective-C `@optional` protocol method's calling convention differ from Swift's protocol extensions with default implementations?"
- "Why must delegate properties be declared `weak`?"

### Simple explanation
`@protocol` is Swift's `protocol`, `@required`/`@optional` is Objective-C's (more manual) version of Swift's protocol extensions with defaults, and `id<Protocol>` is Swift's `any Protocol`.

### Key takeaways
- Protocol methods default to `@required`; opt into `@optional` explicitly.
- Always guard `@optional` calls with `respondsToSelector:`.
- `id<ProtocolName>` = "any object conforming to `ProtocolName`."
- Delegates are `weak` properties, always.

---
## Section 7: Control Flow

### What is it?
Objective-C's control flow is almost entirely inherited from plain C — this is the section with the *fewest* surprises for someone coming from Swift, aside from a handful of missing conveniences (`guard`, `for-in` over arbitrary sequences with pattern matching, `switch` over strings).

### Side-by-side

| Swift | Objective-C | Notes |
|---|---|---|
| `if x { }` | `if (x) { }` | parens required around condition in ObjC |
| `if let x = y { }` | `if (y != nil) { id x = y; ... }` | no optional-binding sugar |
| `guard let x = y else { return }` | `if (!y) { return; }` (then use `y` directly) | no `guard` keyword |
| `switch x { case 1: ... }` | `switch (x) { case 1: ... break; }` | ObjC's `switch` only accepts integers/enums, **not** strings or objects, and needs explicit `break` (no implicit fallthrough protection) |
| `for item in items { }` | `for (id item in items) { }` | "fast enumeration," same semantics |
| `for i in 0..<10 { }` | `for (NSInteger i = 0; i < 10; i++) { }` | plain C-style loop, no ranges |
| `while x { }` | `while (x) { }` | identical |
| `repeat { } while x` | `do { } while (x);` | identical concept, different keyword |
| `continue` / `break` | `continue;` / `break;` | identical |
| `x ? a : b` | `x ? a : b` | identical |
| `a ?? b` | `a ?: b` | ObjC's shorthand ternary omits the middle operand |
| `goto` | *(not used in Swift)* | `goto label;` still legal C, rare but occasionally seen for error-cleanup jumps |

### `switch` on strings — the real gotcha
```swift
switch status {
case "active": ...
case "closed": ...
default: ...
}
```
This **does not translate directly** — ObjC `switch` only works on integral types. The idiomatic translation is an `if`/`else if` chain using `isEqualToString:`:
```objc
if ([status isEqualToString:@"active"]) {
    // ...
} else if ([status isEqualToString:@"closed"]) {
    // ...
} else {
    // ...
}
```
(Enums, by contrast, switch exactly like Swift's enum switches.)

### `guard` — simulating early return
```swift
guard let user = currentUser else { return }
process(user)
```
```objc
User *user = self.currentUser;
if (!user) {
    return;
}
[self process:user];
```
There's no dedicated keyword — you just write the inverted `if` and return manually. This is one of the most common "muscle memory" adjustments for a Swift developer: you'll instinctively reach for `guard` and need to remember to flip the condition.

### Fast enumeration (`for...in`)
```objc
NSArray<NSString *> *names = @[@"Sam", @"Riya", @"Dev"];
for (NSString *name in names) {
    NSLog(@"%@", name);
}
```
This is literally the same feature as Swift's `for item in collection` — "fast enumeration" is Objective-C's name for the same iterator-protocol-driven loop.

### Common mistakes
- Forgetting `break;` at the end of each `switch` case — ObjC, like C, falls through by default.
- Trying to `switch` on an `NSString *` directly — compiles only if you're actually switching on an integer-backed value; strings need `if`/`else if`.
- Missing parentheses around conditions (`if x`) — required in ObjC, optional in Swift.

### Simple explanation
Objective-C's control flow is C's control flow. If you've ever written C, Java, or JavaScript, you already know it — the only "new" idea is that `switch` can't handle strings/objects, and there's no `guard`, so early-return is spelled out by hand.

### Key takeaways
- Parens are mandatory around `if`/`while`/`switch` conditions.
- `switch` works only on integers/enums — use `if`/`else if` + `isEqualToString:`/`isEqual:` for strings/objects.
- No `guard` — write the inverted condition and return manually.
- `for (Type *item in collection)` is fast enumeration, functionally identical to Swift's `for-in`.

---

## Section 8: Collections

### What is it?
Objective-C's core collection types (`NSArray`, `NSDictionary`, `NSSet`) map directly onto Swift's `Array`, `Dictionary`, `Set` — the big difference is that mutability is a **separate subclass** (`NSMutableArray` etc.), not a `let`/`var` distinction, and pre-Swift-generics collections held `id` (i.e., `Any`) rather than a statically-typed element.

### Arrays
```swift
var names = ["Sam", "Riya"]
names.append("Dev")
let first = names[0]
let count = names.count
let filtered = names.filter { $0.hasPrefix("S") }
let sorted = names.sorted()
```
```objc
NSMutableArray<NSString *> *names = [@[@"Sam", @"Riya"] mutableCopy];
[names addObject:@"Dev"];
NSString *first = names[0];                 // modern subscript syntax
NSInteger count = names.count;
NSArray<NSString *> *filtered = [names filteredArrayUsingPredicate:
    [NSPredicate predicateWithBlock:^BOOL(NSString *name, NSDictionary *bindings) {
        return [name hasPrefix:@"S"];
    }]];
NSArray<NSString *> *sorted = [names sortedArrayUsingSelector:@selector(compare:)];
```

### Dictionaries
```swift
var ages: [String: Int] = ["Sam": 27]
ages["Riya"] = 24
for (name, age) in ages { ... }
```
```objc
NSMutableDictionary<NSString *, NSNumber *> *ages = [@{@"Sam": @27} mutableCopy];
ages[@"Riya"] = @24;
for (NSString *name in ages) {
    NSNumber *age = ages[name];
    // ...
}
```

### Sets
```swift
var tags: Set<String> = ["swift", "ios"]
tags.insert("mobile")
```
```objc
NSMutableSet<NSString *> *tags = [[NSMutableSet alloc] initWithArray:@[@"swift", @"ios"]];
[tags addObject:@"mobile"];
```

### Sorting, filtering, searching

| Swift | Objective-C |
|---|---|
| `array.sorted { $0 < $1 }` | `[array sortedArrayUsingComparator:^NSComparisonResult(id a, id b) { return [a compare:b]; }]` |
| `array.filter { $0.age > 18 }` | `[array filteredArrayUsingPredicate:[NSPredicate predicateWithBlock:^BOOL(id obj, NSDictionary *bindings) { return [obj age] > 18; }]]` |
| `array.first(where:)` | manual loop, or `indexOfObjectPassingTest:` |
| `array.contains(x)` | `[array containsObject:x]` |
| `array.map { ... }` | manual loop building a new `NSMutableArray` (no native `map` on `NSArray`) |

### Enumerating with blocks (an ObjC-only idiom, no direct Swift parallel)
```objc
[names enumerateObjectsUsingBlock:^(NSString *name, NSUInteger idx, BOOL *stop) {
    NSLog(@"%lu: %@", (unsigned long)idx, name);
    if ([name isEqualToString:@"Riya"]) {
        *stop = YES;   // early-exit, equivalent to `break` inside a `forEach`
    }
}];
```
The `BOOL *stop` out-parameter is how you break out of a block-based enumeration early — since `break`/`return` inside the block only exits *the block*, not the enclosing loop machinery, `*stop = YES;` is the mechanism the enumeration method itself checks after each iteration.

### `NSPredicate` — Objective-C's filter DSL
`NSPredicate` predates Swift closures and is still common in Core Data fetch requests (Section 14) and collection filtering. It can be built from a format string (SQL-like) or a block:
```objc
NSPredicate *predicate = [NSPredicate predicateWithFormat:@"age > %d", 18];
NSArray *adults = [people filteredArrayUsingPredicate:predicate];
```

### Boxing and unboxing
Objective-C collections can only hold **objects**, not raw scalars — so an `NSInteger`/`BOOL`/`CGFloat` must be "boxed" into an `NSNumber` before insertion:
```swift
let scores: [Int] = [10, 20, 30]   // Swift Array<Int> holds raw Ints directly
```
```objc
NSArray<NSNumber *> *scores = @[@10, @20, @30];   // each @10 boxes an NSInteger into NSNumber
NSInteger first = scores[0].integerValue;          // unboxing back to a scalar
```
`@10` is literal-syntax shorthand for `[NSNumber numberWithInteger:10]` — the boxing happens automatically at the literal, but conceptually it's still creating an object wrapper around a scalar, unlike Swift where `Int` is already a first-class collection element.

### Common mistakes
- Trying to put a raw `NSInteger`/`BOOL` directly into an `NSArray`/`NSDictionary` literal without boxing (`@[5]` works because `5` auto-boxes via `@5` sugar, but a plain C `int` variable must be wrapped: `@(myInt)`).
- Forgetting that `NSArray` (unlike `Array` in Swift) is a reference type — copying a mutable array to hand it to another owner requires an explicit `copy`/`mutableCopy`, or the caller can mutate your array.
- Assuming `map`/`reduce`/`compactMap` exist natively on `NSArray` — they don't; you either use blocks-based APIs, `NSPredicate`, or write a manual loop.

### Interview questions
- "Why does inserting a scalar into an `NSArray` require boxing into `NSNumber`?"
- "How do you break out of `enumerateObjectsUsingBlock:` early?"

### Simple explanation
`NSArray`/`NSDictionary`/`NSSet` are Swift's `Array`/`Dictionary`/`Set`, but reference types, always holding boxed objects, with mutability as a separate class rather than `var`/`let`.

### Key takeaways
- Mutable variants have their own class name (`NSMutableArray`, etc.) — no `let`/`var` distinction.
- Everything inside a collection must be an object — scalars get boxed into `NSNumber`.
- No native `map`/`filter`/`reduce` — use block-based APIs (`enumerateObjectsUsingBlock:`, `NSPredicate`, `sortedArrayUsingComparator:`) or hand-rolled loops.
- `*stop = YES;` inside an enumeration block is how you break out early.

---
## Section 9: Memory Management

This is one of the largest and most important chapters for maintaining legacy code, because a huge share of "weird" old Objective-C (manual `retain`/`release` calls, `@autoreleasepool` blocks, oddly-named methods) exists purely to serve memory management, and Swift developers have never had to think about it directly.

### Why ARC exists
Both Swift and modern Objective-C use **ARC (Automatic Reference Counting)**: the compiler inserts retain/release calls for you at compile time based on static analysis of ownership. This is *not* garbage collection — there's no runtime scanning/pausing; it's deterministic, compiler-inserted bookkeeping. Swift's memory model **is** ARC; Objective-C simply had it bolted on in 2011 after a decade of purely manual reference counting, so legacy code predating that still contains manual calls.

### Reference counting fundamentals

| Operation | What it does | Swift equivalent |
|---|---|---|
| `retain` | increments an object's retain count (claims ownership) | implicit, via a `strong` reference existing |
| `release` | decrements the retain count; deallocates at zero | implicit, when a `strong` reference goes out of scope |
| `autorelease` | schedules a `release` to happen later, at the end of the current autorelease pool | rarely surfaces in Swift; ARC largely eliminates the need |
| `dealloc` | called once retain count hits zero | `deinit` |

Manual reference counting (pre-ARC, "MRC" — you'll still see this in genuinely old files):
```objc
Person *p = [[Person alloc] init]; // retain count = 1
[p retain];                        // retain count = 2
[p release];                       // retain count = 1
[p release];                       // retain count = 0 → dealloc called
```
Under ARC, you **never** call `retain`/`release`/`autorelease` yourself — the compiler inserts them based on `strong`/`weak`/`assign` annotations. If you see explicit `retain`/`release` calls in a codebase, that file (or the whole project) predates ARC, or is intentionally using `-fno-objc-arc` for a specific reason (rare, usually a bridging/legacy C interop concern).

### `strong`, `weak`, `assign`, `copy` — memory semantics recap
See Section 4 for the property-declaration syntax; the underlying memory behavior:

- **`strong`**: the standard owning reference. Object stays alive as long as at least one strong reference to it exists — exactly Swift's default class reference behavior.
- **`weak`**: non-owning; **automatically zeroed** to `nil` the instant the referenced object deallocates. Same semantics as Swift's `weak var`.
- **`assign`**: non-owning, **not** zeroed automatically — if the underlying object deallocates, the pointer becomes "dangling" (points at garbage memory). This is the historical cause of Objective-C's infamous crash class: **the zombie object** (see below). Modern code uses `assign` almost exclusively for scalars (`NSInteger`, `CGRect`) where the concept of dangling doesn't apply, not for objects.
- **`copy`**: makes an independent copy at assignment time — used for `NSString`, blocks, and mutable collection types being stored as immutable snapshots.

### Autorelease pools
```objc
@autoreleasepool {
    // any object sent `autorelease` inside this block is released
    // when the block exits, rather than immediately
}
```
Autorelease pools batch up deferred releases (historically needed so a factory method like `+ (instancetype)stringWithFormat:` could return an object without the caller needing to `retain` it immediately — the object survives just long enough to be used, then gets cleaned up at the next pool drain). Under ARC you rarely write `@autoreleasepool` yourself except around tight loops that create many temporary objects (e.g., processing a huge array), to prevent memory from ballooning until the *outer* pool (main run loop) eventually drains.

### Retain cycles
A retain cycle is two (or more) objects holding **strong** references to each other, so neither's retain count ever reaches zero — memory leaks forever. This is exactly the same failure mode Swift can hit with `class` types and closures; Objective-C's most common instances:
```objc
@interface Parent : NSObject
@property (nonatomic, strong) Child *child;
@end

@interface Child : NSObject
@property (nonatomic, strong) Parent *parent; // BUG: should be `weak`
@end
```
Fix: make the back-reference `weak`. Same principle for delegate properties (Section 6) and blocks capturing `self` (Section 5).

### Zombie objects
A "zombie" is a deallocated object that still has a **dangling pointer** referencing it — sending a message to it doesn't just crash cleanly, it can silently corrupt memory or behave unpredictably, because the memory has been reused for something else. Xcode's **Zombie Objects** diagnostic (Product → Scheme → Edit Scheme → Diagnostics → Enable Zombie Objects) intercepts messages to deallocated objects and turns them into a clear, catchable crash with a useful stack trace instead of silent corruption — indispensable for debugging legacy `assign`-based object references or use-after-free bugs.

### Memory Graph Debugger
Xcode's Memory Graph tool (the icon that looks like three connected circles in the debug bar) visualizes every live object and its reference chain — the direct equivalent of using Instruments' Leaks tool, and invaluable for spotting retain cycles visually: it flags cycles with a purple exclamation mark.

### Comparison with Swift ARC
Objective-C's ARC and Swift's ARC are **the same underlying mechanism** — Swift didn't reinvent memory management, it inherited ARC from Objective-C and added compile-time enforcement (you can't forget `weak` on a delegate the way you can in ObjC, since Swift's compiler is stricter about certain patterns, though it still can't catch all cycles, e.g. closures capturing `self` strongly). This is why bridging between the two languages is possible at all — they share one reference-counting model.

### Common mistakes
- Making a parent→child strong reference bidirectional without marking one side `weak`.
- Manually calling `retain`/`release`/`autorelease` inside ARC code — this is now a **compile error**, not just bad practice, since ARC forbids explicit calls.
- Using `assign` for an object property (rather than a scalar) — creates a dangling-pointer risk once the referenced object deallocates.
- Forgetting to remove a `NSNotificationCenter` observer in `dealloc` in codebases targeting old OS versions (modern iOS auto-removes on dealloc, but very old code/patterns may still need it explicitly).

### Interview questions
- "Explain the difference between `weak` and `assign` for an object property."
- "What is a zombie object, and how would you diagnose one?"
- "Why can't you call `retain`/`release` directly in ARC code?"

### Simple explanation
ARC in Objective-C is the exact same "compiler counts your references and cleans up automatically" system that powers Swift's memory management — you're just occasionally seeing its manual, pre-2011 ancestor in old files, or configuring its edge cases (`weak`, `autoreleasepool`) by hand where Swift's compiler does more of that inference for you.

### Key takeaways
- ARC is shared between Swift and Objective-C — not a separate system.
- `weak` auto-nils on deallocation; `assign` (on objects) can dangle — always prefer `weak` for object back-references.
- Retain cycles: same failure mode as Swift, same fix (`weak` on the back-reference, or `[weak self]` in blocks).
- Zombie Objects + Memory Graph Debugger are the primary tools for hunting memory bugs in legacy code.

---
## Section 10: UIKit Development

### What is it?
UIKit itself is the *same framework* whether you write Swift or Objective-C — a `UIViewController` in ObjC is the identical class you subclass in Swift, just accessed through message-passing syntax. Almost everything in this section is "same API, different calling convention," not new concepts.

### View controller lifecycle
```swift
override func viewDidLoad() {
    super.viewDidLoad()
    setupUI()
}
```
```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupUI];
}
```
Same lifecycle methods (`viewDidLoad`, `viewWillAppear:`, `viewDidAppear:`, `viewWillDisappear:`, `viewDidDisappear:`) — literally the same methods on the same base class, so if you know the Swift lifecycle, you already know the Objective-C one.

### IBOutlet / IBAction
```swift
@IBOutlet weak var titleLabel: UILabel!
@IBAction func buttonTapped(_ sender: UIButton) { }
```
```objc
@property (nonatomic, weak) IBOutlet UILabel *titleLabel;
- (IBAction)buttonTapped:(UIButton *)sender;
```
`IBOutlet` and `IBAction` are just markers (they don't change runtime behavior) that Interface Builder scans for to know which properties/methods can be wired up in a Storyboard/XIB — same purpose in both languages.

### Target-action
```swift
button.addTarget(self, action: #selector(buttonTapped), for: .touchUpInside)
```
```objc
[button addTarget:self action:@selector(buttonTapped:) forControlEvents:UIControlEventTouchUpInside];
```

### Programmatic UI & Auto Layout
```swift
let label = UILabel()
label.translatesAutoresizingMaskIntoConstraints = false
view.addSubview(label)
NSLayoutConstraint.activate([
    label.centerXAnchor.constraint(equalTo: view.centerXAnchor),
    label.topAnchor.constraint(equalTo: view.topAnchor, constant: 20)
])
```
```objc
UILabel *label = [[UILabel alloc] init];
label.translatesAutoresizingMaskIntoConstraints = NO;
[self.view addSubview:label];
[NSLayoutConstraint activateConstraints:@[
    [label.centerXAnchor constraintEqualToAnchor:self.view.centerXAnchor],
    [label.topAnchor constraintEqualToAnchor:self.view.topAnchor constant:20]
]];
```
Note the **dot syntax** here (`label.centerXAnchor`) — Objective-C added dot syntax for **property access** (not arbitrary method calls) in the 2.0 era, specifically so chains like this stay readable. It's sugar for `[label centerXAnchor]` underneath — don't confuse it with Swift's dot syntax, which works for both properties *and* methods.

### UITableView / UICollectionView
Already shown in Section 6 (delegate/dataSource pattern) — same lifecycle: register a cell class/nib, implement `numberOfRowsInSection:`, `cellForRowAtIndexPath:`, dequeue with a reuse identifier, exactly as in Swift.

```objc
[self.tableView registerClass:[UITableViewCell class] forCellReuseIdentifier:@"Cell"];
```

### Navigation
```swift
navigationController?.pushViewController(detailVC, animated: true)
present(modalVC, animated: true)
dismiss(animated: true)
```
```objc
[self.navigationController pushViewController:detailVC animated:YES];
[self presentViewController:modalVC animated:YES completion:nil];
[self dismissViewControllerAnimated:YES completion:nil];
```

### Passing data between view controllers
```swift
let detail = DetailViewController()
detail.item = selectedItem
navigationController?.pushViewController(detail, animated: true)
```
```objc
DetailViewController *detail = [[DetailViewController alloc] init];
detail.item = selectedItem;
[self.navigationController pushViewController:detail animated:YES];
```

### Storyboard segues
```objc
- (void)prepareForSegue:(UIStoryboardSegue *)segue sender:(id)sender {
    if ([segue.identifier isEqualToString:@"ShowDetail"]) {
        DetailViewController *detail = segue.destinationViewController;
        detail.item = self.selectedItem;
    }
}
```

### Common mistakes
- Declaring `IBOutlet`s as `strong` instead of `weak` (usually harmless with modern Auto Layout-managed view hierarchies, but goes against convention and can cause double-ownership confusion).
- Forgetting `[super viewDidLoad]`/`[super viewWillAppear:animated]` — same bug as forgetting `super.viewDidLoad()` in Swift, but without a compiler warning to catch it.
- Confusing property dot-syntax with arbitrary method calls — `view.subviews.count` works (chained properties), but you still can't do `view.addSubview(x)` with dot syntax; that stays bracketed: `[view addSubview:x]`.

### Simple explanation
UIKit in Objective-C is UIKit — the exact same classes, the exact same lifecycle, the exact same delegate/dataSource pattern you already know from Swift, communicated through bracket-message syntax instead of dot-call syntax.

### Key takeaways
- Lifecycle methods, delegate/dataSource protocols, and Auto Layout concepts transfer 1:1 — the syntax is the only new thing.
- Dot syntax works for *properties* only, not for method calls with arguments — those stay in bracket form.
- `IBOutlet`/`IBAction` are compiler-transparent markers for Interface Builder, not behavior-changing keywords.

---
## Section 11: Networking

### What is it?
`NSURLSession` (bridged to Swift as `URLSession`) is literally the same class in both languages — there's no separate "Objective-C networking stack." The difference is entirely completion-block syntax vs. Swift's closures/async-await, and manual JSON parsing vs. `Codable`.

### A complete GET request, side by side

```swift
let url = URL(string: "https://api.example.com/users")!
var request = URLRequest(url: url)
request.httpMethod = "GET"

URLSession.shared.dataTask(with: request) { data, response, error in
    if let error = error {
        print("Error: \(error)")
        return
    }
    guard let data = data else { return }
    do {
        let users = try JSONDecoder().decode([User].self, from: data)
        DispatchQueue.main.async {
            self.updateUI(with: users)
        }
    } catch {
        print("Decode error: \(error)")
    }
}.resume()
```

```objc
NSURL *url = [NSURL URLWithString:@"https://api.example.com/users"];
NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
request.HTTPMethod = @"GET";

NSURLSessionDataTask *task = [[NSURLSession sharedSession]
    dataTaskWithRequest:request
    completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
        if (error) {
            NSLog(@"Error: %@", error);
            return;
        }
        if (!data) { return; }

        NSError *jsonError;
        NSArray *jsonArray = [NSJSONSerialization JSONObjectWithData:data options:0 error:&jsonError];
        if (jsonError) {
            NSLog(@"Decode error: %@", jsonError);
            return;
        }

        NSMutableArray<User *> *users = [NSMutableArray array];
        for (NSDictionary *dict in jsonArray) {
            [users addObject:[User userFromDictionary:dict]];
        }

        dispatch_async(dispatch_get_main_queue(), ^{
            [self updateUIWithUsers:users];
        });
    }];
[task resume];
```

### Line-by-line explanation
- `NSMutableURLRequest` — mutable because you're setting `HTTPMethod` after creation; Swift's `URLRequest` is a struct so `var request` alone gives you mutability.
- `dataTaskWithRequest:completionHandler:` — the block runs on a **background** queue by default (same as Swift's `URLSession` completion closures), which is why UI updates are dispatched back to `dispatch_get_main_queue()`.
- `NSJSONSerialization JSONObjectWithData:options:error:` — the ObjC-era JSON parser, predating `Codable` entirely. It returns `id` (either an `NSDictionary` or `NSArray` at the top level, matching the JSON's actual shape), which you then manually walk and convert into your model objects.
- `&jsonError` — passing the **address of** a local `NSError *` variable, so the method can set it if parsing fails (see Section 15 for the general pattern).

### `NSError **` parsing pattern
```objc
NSError *error;
id result = [NSJSONSerialization JSONObjectWithData:data options:0 error:&error];
if (!result) {
    // error is now populated
}
```
This `&error` out-parameter idiom is Objective-C's substitute for `try`/`catch`/`throws` — see Section 15 for full treatment.

### Manual `Codable` equivalent — `userFromDictionary:`
```objc
+ (instancetype)userFromDictionary:(NSDictionary *)dict {
    User *user = [[User alloc] init];
    user.name = dict[@"name"];
    user.age = [dict[@"age"] integerValue];
    return user;
}
```
There's no automatic `Decodable` synthesis — every field is pulled out of the dictionary by key and cast/coerced by hand. `NSCoding`/`NSSecureCoding` exist for archiving objects to disk (closer to `NSKeyedArchiver`/old-style persistence) but don't help with arbitrary JSON the way `Codable` does.

### Uploads/downloads
```objc
NSURLSessionUploadTask *uploadTask = [[NSURLSession sharedSession]
    uploadTaskWithRequest:request
    fromData:bodyData
    completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) { ... }];
[uploadTask resume];

NSURLSessionDownloadTask *downloadTask = [[NSURLSession sharedSession]
    downloadTaskWithURL:fileURL
    completionHandler:^(NSURL *location, NSURLResponse *response, NSError *error) {
        // `location` is a temporary file — move it before the completion handler returns
    }];
[downloadTask resume];
```

### Common mistakes
- Forgetting `[task resume]` — like forgetting to call anything on a Swift `URLSessionTask`, the task simply never starts.
- Not dispatching UI updates back to the main queue inside a completion handler.
- Assuming `NSJSONSerialization` gives you typed models — it only gives you `NSDictionary`/`NSArray`/`NSNumber`/`NSString`/`NSNull`; all typed mapping is manual.

### Simple explanation
Networking code is the same `NSURLSession` you use in Swift, called with block syntax instead of closures, and parsed with manual dictionary-walking instead of `Codable`.

### Key takeaways
- `NSURLSession` is shared infrastructure — no separate ObjC networking stack.
- `NSJSONSerialization` returns loosely-typed `NSDictionary`/`NSArray`; you map fields to model objects by hand.
- Completion handlers run off the main thread; dispatch UI work back with `dispatch_async(dispatch_get_main_queue(), ...)`.

---
## Section 12: Concurrency

### What is it?
Grand Central Dispatch (GCD) and `NSOperationQueue` are the pre-`async`/`await` concurrency tools — and they're not "legacy" so much as "still the underlying mechanism" (Swift's `async`/`await` on Apple platforms is layered on top of, and interoperates with, the same dispatch queues).

### Dispatch queues

| Swift | Objective-C |
|---|---|
| `DispatchQueue.main.async { }` | `dispatch_async(dispatch_get_main_queue(), ^{ ... });` |
| `DispatchQueue.global().async { }` | `dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{ ... });` |
| `DispatchQueue(label: "com.app.myQueue").async { }` | `dispatch_queue_t queue = dispatch_queue_create("com.app.myQueue", DISPATCH_QUEUE_SERIAL); dispatch_async(queue, ^{ ... });` |
| `DispatchQueue.main.sync { }` | `dispatch_sync(dispatch_get_main_queue(), ^{ ... });` *(dangerous if already on main — deadlocks)* |

### Async vs sync
```objc
dispatch_async(queue, ^{
    // runs later, doesn't block the calling thread
});

dispatch_sync(queue, ^{
    // blocks the calling thread until this finishes — use sparingly
});
```
Same semantics as Swift's `.async`/`.sync` — `sync` onto the *current* queue (especially main) is a classic deadlock bug in both languages.

### Dispatch groups — waiting for multiple async tasks
```swift
let group = DispatchGroup()
group.enter()
fetchA { group.leave() }
group.enter()
fetchB { group.leave() }
group.notify(queue: .main) {
    print("both done")
}
```
```objc
dispatch_group_t group = dispatch_group_create();

dispatch_group_enter(group);
[self fetchAWithCompletion:^{ dispatch_group_leave(group); }];

dispatch_group_enter(group);
[self fetchBWithCompletion:^{ dispatch_group_leave(group); }];

dispatch_group_notify(group, dispatch_get_main_queue(), ^{
    NSLog(@"both done");
});
```

### Semaphores
```objc
dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
[self fetchDataWithCompletion:^(NSData *data) {
    self.result = data;
    dispatch_semaphore_signal(semaphore);
}];
dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER); // blocks until signaled
```
Used to force asynchronous code to behave synchronously (e.g., in tests, or bridging an async API into a synchronous call site) — Swift's structured concurrency mostly removes the need for this pattern, but you'll still see it in older utility/networking code.

### `NSOperation` / `NSOperationQueue`
A higher-level abstraction over GCD, supporting cancellation, dependencies between tasks, and priority — closer conceptually to Swift's `Task` with structured cancellation, though the API predates it by a decade.
```objc
NSOperationQueue *queue = [[NSOperationQueue alloc] init];
queue.maxConcurrentOperationCount = 2;

NSBlockOperation *op1 = [NSBlockOperation blockOperationWithBlock:^{
    // work
}];
NSBlockOperation *op2 = [NSBlockOperation blockOperationWithBlock:^{
    // depends on op1
}];
[op2 addDependency:op1];

[queue addOperation:op1];
[queue addOperation:op2];
```

### Comparison with `async`/`await`
Swift's structured concurrency (`async`/`await`, `Task`, actors) is a *language-level* abstraction that, under the hood on Apple platforms, still schedules work onto the same dispatch queues/thread pool GCD uses. There's no ObjC equivalent of `async`/`await` syntax — the closest practical translation when reading legacy code is: "wherever you'd write `await`, ObjC has a completion block; wherever you'd write `Task { }`, ObjC has `dispatch_async`."

### Common mistakes
- Calling `dispatch_sync` on the main queue from code already running on main — instant deadlock.
- Forgetting `dispatch_group_leave` on an error path — the group's `notify` block never fires.
- Using a semaphore to fake synchronous behavior on the main thread — blocks the UI just like any synchronous main-thread work would in Swift.

### Simple explanation
GCD is Swift's `DispatchQueue` API — it's literally the same framework. Read `dispatch_async(dispatch_get_main_queue(), ^{ ... })` as "DispatchQueue.main.async { ... }, spelled in C function-call syntax with a block."

### Key takeaways
- `dispatch_async`/`dispatch_sync` = `.async`/`.sync` — same queue model as Swift.
- Dispatch groups = waiting on multiple async operations to finish, same idea as Swift's `DispatchGroup` (because it's the same API).
- `NSOperationQueue` = task dependencies + cancellation, conceptually closer to structured concurrency than raw GCD.
- No `async`/`await` syntax exists in ObjC — everything is completion-block or queue-based.

---
## Section 13: Notifications

### What is it?
`NSNotificationCenter` (bridged as `NotificationCenter` in Swift) is the same publish/subscribe mechanism in both languages — a global bus objects can post to and observe, decoupling sender from receiver.

### Posting
```swift
NotificationCenter.default.post(name: .userDidLogin, object: nil, userInfo: ["userID": id])
```
```objc
[[NSNotificationCenter defaultCenter] postNotificationName:@"UserDidLoginNotification"
                                                     object:nil
                                                   userInfo:@{@"userID": userId}];
```
Objective-C notification names are plain `NSString *` constants (by convention, declared as `static NSString *const` — see Section 1) rather than Swift's `Notification.Name` wrapper type, which exists purely to give compile-time type safety over what's still, underneath, the same string-keyed mechanism.

### Observing
```swift
NotificationCenter.default.addObserver(self, selector: #selector(handleLogin), name: .userDidLogin, object: nil)
```
```objc
[[NSNotificationCenter defaultCenter] addObserver:self
                                          selector:@selector(handleLogin:)
                                              name:@"UserDidLoginNotification"
                                            object:nil];

- (void)handleLogin:(NSNotification *)notification {
    NSString *userId = notification.userInfo[@"userID"];
    // ...
}
```

### Removing observers
```objc
- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}
```
On modern iOS/macOS, `NSNotificationCenter` automatically removes all observers for an object when it deallocates — but explicitly removing in `dealloc` remains common defensive practice in legacy code (and is required if you're targeting truly ancient OS versions or removing only *specific* observer registrations before the object fully deallocates).

### `userInfo`
`userInfo` is an `NSDictionary *` (Swift's equivalent is the `userInfo: [AnyHashable: Any]?` payload) — a loosely-typed bag of extra data attached to a notification, same purpose in both languages, same lack of compile-time type safety on either side of the bridge.

### Common mistakes
- Forgetting to remove an observer on an object that gets deallocated while still registered — on older OS versions, this could crash; today it's usually just a (still worth fixing) latent bug if you're relying on manual removal timing.
- Mismatched notification name strings between poster and observer (a typo in either string silently means the notification is never received) — this is the main reason to use a single shared `static NSString *const` constant rather than typing the literal string in multiple places.

### Simple explanation
Same `NotificationCenter` you already use in Swift; notification names are plain strings instead of the `Notification.Name` wrapper, and the observer callback is a selector instead of a closure.

### Key takeaways
- Notification names: use a shared `static NSString *const` constant, never repeat literal strings.
- Observers register with `addObserver:selector:name:object:`; the selector method receives an `NSNotification *`.
- Remove observers in `dealloc` as defensive practice, even though modern OS versions handle it automatically.

---

## Section 14: Core Data

### What is it?
Core Data itself is unchanged between Swift and Objective-C — `NSManagedObject`, `NSFetchRequest`, `NSPredicate`, and `NSManagedObjectContext` are the same classes; only the calling syntax differs.

### Fetching
```swift
let request: NSFetchRequest<Person> = Person.fetchRequest()
request.predicate = NSPredicate(format: "age > %d", 18)
request.sortDescriptors = [NSSortDescriptor(key: "name", ascending: true)]
let results = try? context.fetch(request)
```
```objc
NSFetchRequest *request = [Person fetchRequest];
request.predicate = [NSPredicate predicateWithFormat:@"age > %d", 18];
request.sortDescriptors = @[[NSSortDescriptor sortDescriptorWithKey:@"name" ascending:YES]];

NSError *error;
NSArray<Person *> *results = [context executeFetchRequest:request error:&error];
if (!results) {
    NSLog(@"Fetch failed: %@", error);
}
```

### Saving
```swift
do {
    try context.save()
} catch {
    print("Save failed: \(error)")
}
```
```objc
NSError *error;
if (![context save:&error]) {
    NSLog(@"Save failed: %@", error);
}
```

### Creating / deleting / updating
```objc
Person *newPerson = [NSEntityDescription insertNewObjectForEntityForName:@"Person"
                                                   inManagedObjectContext:context];
newPerson.name = @"Sam";
newPerson.age = 27;

[context deleteObject:existingPerson];

existingPerson.name = @"Updated Name"; // updates are just property assignment
[context save:&error];
```

### Predicates & sorting
Already covered in Section 8 — `NSPredicate` and `NSSortDescriptor` are identical objects used identically in both languages; Core Data is simply the framework where you'll encounter them most heavily.

### Common mistakes
- Ignoring the `error` out-parameter from `save:`/`executeFetchRequest:error:` — silently swallowing Core Data failures.
- Forgetting that `NSManagedObject` properties are dynamically generated (via `@dynamic` under the hood) — you rarely need to write custom getters/setters for modeled attributes; the Core Data runtime synthesizes them.

### Simple explanation
Core Data is Core Data — the exact same persistence framework Swift Core Data code uses; the only change is `try`/`catch` becoming an `NSError **` out-parameter you must check by hand.

### Key takeaways
- Same classes (`NSManagedObject`, `NSFetchRequest`, `NSPredicate`, `NSManagedObjectContext`) as Swift Core Data.
- Errors come back via `NSError **`, not `throws` — always check the returned `BOOL`/`nil` alongside the error.
- Predicates and sort descriptors are shared infrastructure with plain collection filtering (Section 8).

---
## Section 15: Error Handling

### What is it?
Objective-C has no `try`/`catch`/`throws` for ordinary error propagation (it does have `@try`/`@catch`/`@finally` for **exceptions**, but those are reserved for programmer-error/fatal conditions, not everyday recoverable failures). Ordinary "this operation might fail" code uses an **`NSError **` out-parameter** — the method returns `nil`/`NO` on failure and populates the error object at the address you passed in.

### Swift version
```swift
func loadFile(at path: String) throws -> Data {
    guard let data = FileManager.default.contents(atPath: path) else {
        throw FileError.notFound
    }
    return data
}

do {
    let data = try loadFile(at: "/tmp/file.txt")
} catch {
    print("Failed: \(error)")
}
```

### Objective-C version
```objc
- (nullable NSData *)loadFileAtPath:(NSString *)path error:(NSError **)error {
    NSData *data = [[NSFileManager defaultManager] contentsAtPath:path];
    if (!data) {
        if (error) {
            *error = [NSError errorWithDomain:@"FileErrorDomain"
                                          code:404
                                      userInfo:@{NSLocalizedDescriptionKey: @"File not found"}];
        }
        return nil;
    }
    return data;
}

NSError *error;
NSData *data = [self loadFileAtPath:@"/tmp/file.txt" error:&error];
if (!data) {
    NSLog(@"Failed: %@", error);
}
```

### Reading the `NSError **` pattern
- The method signature takes `NSError **error` — a **pointer to a pointer**: the caller passes the *address* of their local `NSError *` variable (`&error`), so the callee can write a new `NSError *` value into that variable.
- Convention: the method also returns a sentinel failure value (`nil` for objects, `NO`/`0` for scalars) — **the return value is the actual failure signal**; the error object is supplementary detail. Always check the return value first; a method is allowed to return failure without populating the error in some edge cases (and is never guaranteed to populate it if you passed `NULL` for the error parameter, which is legal — hence the `if (error) { *error = ... }` guard).
- `if (error) { ... }` inside the callee guards against the caller passing `NULL` (some call sites don't care about the error detail and just check the return value).

### `NSError` structure
```objc
NSError *error = [NSError errorWithDomain:@"com.myapp.network"
                                      code:1001
                                  userInfo:@{NSLocalizedDescriptionKey: @"Request timed out"}];
```
- **domain**: a string namespacing where the error came from (like Swift's `Error` conforming type acting as an implicit "domain").
- **code**: an integer identifying the specific error within that domain (like a Swift enum's case, but not statically checked).
- **userInfo**: a dictionary of extra details, most commonly `NSLocalizedDescriptionKey` for a human-readable message.

This is a looser, stringly-typed analogue of Swift's `Error` protocol + enum-based error types — no compiler exhaustiveness checking, no pattern matching, just domain/code/dictionary conventions.

### `@try`/`@catch`/`@finally` — reserved for exceptional, not expected, failures
```objc
@try {
    [array objectAtIndex:99]; // out of bounds — programmer error
} @catch (NSException *exception) {
    NSLog(@"Caught: %@", exception.reason);
} @finally {
    NSLog(@"Cleanup");
}
```
Apple's own convention (and near-universal community practice): `NSException` is for **programmer errors** (array out-of-bounds, invalid arguments, "this should never happen") — the kind of thing that in Swift would just be an uncaught runtime trap (`fatalError`, array index crash), not something you're expected to catch and recover from in normal control flow. Ordinary recoverable failures (network errors, file-not-found, validation failures) use `NSError **`, not exceptions. Mixing these up is one of the more subtle mistakes an ObjC newcomer makes.

### Comparison with Swift's `throws`
| Swift | Objective-C |
|---|---|
| `throws` in a function signature | `NSError **error` parameter + failure-sentinel return value |
| `try` at the call site | `&error` passed in, then manually checking the return value/error afterward |
| `catch { }` | `if (!result) { /* inspect error */ }` |
| `Error` protocol | `NSError` class (domain/code/userInfo) |
| Compiler-enforced exhaustive handling | none — nothing forces you to check the error; silently ignoring it compiles fine |

### `defer`-equivalent cleanup
There's no `defer` keyword; equivalent cleanup-on-every-path code is either duplicated at each `return`, or handled via `@finally` if you're already inside a `@try` block (rare for non-exception code) or careful manual structuring.

### Common mistakes
- Ignoring the `NSError **` parameter and the failure sentinel entirely — since nothing forces you to check it, silent failures are common in sloppy legacy code.
- Using `@try`/`@catch` for ordinary recoverable errors (e.g., wrapping a network call) instead of the `NSError **` convention — works, but is against Apple's own guidance and typically signals a misunderstanding of the pattern.
- Passing `NULL` for the `error:` parameter and then being surprised there's no error detail available on failure.

### Interview questions
- "Why does Objective-C use an `NSError **` out-parameter instead of exceptions for recoverable errors?"
- "What's the actual failure signal in a method with an `NSError **` parameter — the return value or the error object?"

### Simple explanation
Objective-C's `throws` is "pass in the address of an error variable, check my return value, and I'll fill in your error variable if something went wrong." `@try`/`@catch` is reserved for the kind of failure Swift would let crash outright.

### Key takeaways
- Ordinary recoverable errors: `NSError **` out-parameter + failure-sentinel return value — this is Objective-C's `throws`.
- `@try`/`@catch`/`@finally` = for programmer errors/exceptions only, not everyday failure handling.
- Nothing enforces checking the error — unlike Swift's `try`, it's easy to silently ignore.

---
## Section 16: Categories & Extensions

### What is it?
A **category** lets you add methods to an existing class — including classes you don't own the source for, like `NSString` or `UIView` — without subclassing. This is more powerful (and more dangerous) than Swift's `extension`, because a category can be applied globally, affecting every instance of that class app-wide, including inside frameworks you don't control.

### Swift version
```swift
extension String {
    func reversedWords() -> String {
        return self.split(separator: " ").reversed().joined(separator: " ")
    }
}
```

### Objective-C version
```objc
// NSString+WordUtilities.h
@interface NSString (WordUtilities)
- (NSString *)reversedWords;
@end

// NSString+WordUtilities.m
@implementation NSString (WordUtilities)
- (NSString *)reversedWords {
    NSArray<NSString *> *words = [self componentsSeparatedByString:@" "];
    NSArray<NSString *> *reversed = [[words reverseObjectEnumerator] allObjects];
    return [reversed componentsJoinedByString:@" "];
}
@end
```
Naming convention: `ClassName+CategoryName.h/.m` — e.g. `NSString+WordUtilities`. Any file that `#import`s this header gets the new method available on **every** `NSString` instance in the whole program.

### Class extensions (the "anonymous category")
```objc
@interface Person ()
@property (nonatomic, strong) NSDate *lastLoginDate; // private, only visible within Person.m
@end
```
A class extension is a category with no name, declared in the same `.m` file as the class's implementation — the idiomatic way to add "private" properties/methods, closest to Swift's `private extension` on the same type, or simply keeping something out of the public interface.

### Associated objects
Objective-C categories **cannot add stored properties** the normal way (no ivar storage slot exists on a class you didn't originally define) — `@property` in a category only generates a getter/setter declaration, with no backing storage, unless you use the **associated objects** runtime API to simulate one:
```objc
#import <objc/runtime.h>

static void *kMyAssociatedKey = &kMyAssociatedKey;

@implementation UIView (Tagging)

- (void)setCustomTag:(NSString *)customTag {
    objc_setAssociatedObject(self, kMyAssociatedKey, customTag, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

- (NSString *)customTag {
    return objc_getAssociatedObject(self, kMyAssociatedKey);
}

@end
```
This is a genuinely unique Objective-C capability with no clean Swift equivalent (Swift extensions similarly can't add stored properties, and don't expose an associated-objects escape hatch the same way, since Swift doesn't have the same dynamic runtime).

### Method swizzling
Swapping one method's implementation for another at runtime — used (carefully, and controversially) for cross-cutting concerns like automatic analytics on `viewDidAppear:` across every view controller, without modifying each one:
```objc
#import <objc/runtime.h>

+ (void)load {
    Method original = class_getInstanceMethod(self, @selector(viewDidAppear:));
    Method swizzled = class_getInstanceMethod(self, @selector(swizzled_viewDidAppear:));
    method_exchangeImplementations(original, swizzled);
}

- (void)swizzled_viewDidAppear:(BOOL)animated {
    [self swizzled_viewDidAppear:animated]; // now calls the ORIGINAL implementation, post-swap
    NSLog(@"Tracked appearance of %@", self);
}
```
Swift has no built-in swizzling mechanism (Swift's static/`final` dispatch defeats it for pure-Swift types; it only works reliably on `@objc`/`NSObject`-derived classes) — this remains a distinctly Objective-C-era pattern you'll encounter in analytics SDKs and older cross-cutting utility code.

### Common mistakes
- Adding a category method with the **same name** as an existing method on that class (or another loaded category) — the runtime's behavior in this collision case is undefined (last-loaded category typically wins, but this isn't a reliable contract), a notorious source of "why is this method behaving differently than I wrote it" bugs.
- Attempting to declare a `@property` in a category expecting normal stored-property behavior — it compiles, but crashes/misbehaves at runtime unless backed by associated objects.
- Swizzling without preserving/calling the original implementation, permanently losing the base behavior.

### Interview questions
- "Why can't a category add a stored property directly?"
- "What's the risk of two categories both implementing a method with the same selector name on the same class?"
- "What's method swizzling used for, and why is it risky?"

### Simple explanation
A category is Swift's `extension`, except it can be applied to *any* class (including ones you don't own), can't hold stored properties without a runtime trick, and — unlike Swift extensions — can even **replace** existing methods via swizzling.

### Key takeaways
- Categories extend any class's method surface, even framework classes; class extensions are the private, same-file variant.
- No stored properties in categories without associated objects (`objc_setAssociatedObject`/`objc_getAssociatedObject`).
- Method swizzling swaps implementations at runtime — powerful, but easy to misuse; always call through to the original implementation.

---
## Section 17: The Objective-C Runtime

### What is it?
Everything that makes Objective-C feel "dynamic" compared to Swift — messaging instead of direct calls, introspection, swizzling, `respondsToSelector:` — is powered by a genuine runtime library (`libobjc`) that ships as part of the OS, with real C functions you can call directly. Understanding this is what makes the rest of the language's quirks click into place.

### `objc_msgSend` and dynamic dispatch
When you write:
```objc
[person greet];
```
the compiler translates it (roughly) to:
```c
objc_msgSend(person, @selector(greet));
```
`objc_msgSend` looks up, **at runtime**, which actual implementation to run for the selector `greet` on `person`'s class — following the class's method list, then its superclass's, and so on up the chain. This is fundamentally different from Swift's default **static/vtable dispatch** for most types (only `@objc dynamic` Swift members opt into the same `objc_msgSend` path). It's *why* Objective-C method calls have historically been somewhat slower than Swift's default dispatch, and *why* swizzling/associated objects/KVO are all possible — the lookup step is a real, interceptable function call, not a compiled-in jump.

### The `isa` pointer
Every Objective-C object's first field is a hidden `isa` pointer, referencing its `Class` — the runtime's way of answering "what class is this instance, so I know which method list to search?" `Class` objects themselves have their own `isa` pointing to their **metaclass**, which is how class (`+`) methods are looked up the same way instance (`-`) methods are, just one level up the chain.

### Selectors and method lookup, step by step
1. `[person greet]` compiles to `objc_msgSend(person, @selector(greet))`.
2. The runtime reads `person`'s `isa` to find `Person`'s `Class` structure.
3. It searches `Person`'s method list for a method matching the selector `greet`.
4. Not found? Walk up to `Person`'s superclass, repeat.
5. Found? Call that function pointer (the "IMP" — implementation pointer) with `person` and the selector as its first two hidden arguments.
6. Not found anywhere in the chain? The runtime falls back to **message forwarding** (below) before finally raising "unrecognized selector."

### Reflection / class inspection
```objc
if ([person respondsToSelector:@selector(greet)]) {
    [person performSelector:@selector(greet)];
}

if ([person isKindOfClass:[Employee class]]) { ... }   // like Swift's `is`
Employee *employee = (Employee *)person;                // like Swift's `as!`, unchecked

NSString *className = NSStringFromClass([person class]);
```
`respondsToSelector:` is Objective-C's answer to "does this conform/have this member," used constantly for `@optional` protocol methods (Section 6). `performSelector:` invokes a method **by name at runtime**, even one not known at compile time — genuine reflection, with no Swift equivalent outside of `@objc`-exposed members via `NSObject` bridging.

### Dynamic method resolution & message forwarding
If `objc_msgSend` can't find *any* implementation for a selector anywhere up the class chain, the runtime gives the object one last chance to handle it dynamically, in order:
1. `+resolveInstanceMethod:` — "can you supply an implementation for this selector right now?"
2. `-forwardingTargetForSelector:` — "is there some *other* object that should really handle this?"
3. `-methodSignatureForSelector:` + `-forwardInvocation:` — full manual message forwarding, letting you inspect and redirect the entire call.

This machinery underpins things like `NSProxy`-based dynamic proxies and some ORM/mocking libraries — genuinely advanced runtime territory, no direct Swift parallel (Swift's `@dynamicMemberLookup`/`@dynamicCallable` are a distant, much more constrained cousin).

### Associated objects & swizzling
Covered in Section 16 — both are runtime-library features (`objc_setAssociatedObject`, `method_exchangeImplementations`) rather than language keywords, which is why they require `#import <objc/runtime.h>` explicitly.

### Common mistakes
- Assuming Swift's static dispatch and Objective-C's dynamic dispatch behave identically for performance-sensitive hot loops — they don't; ObjC method calls carry real runtime lookup overhead.
- Calling `performSelector:` with more than 2 arguments — the API only supports up to 2 object arguments directly; more complex dynamic calls need `NSInvocation`.
- Forgetting that `isKindOfClass:`/`respondsToSelector:` checks are runtime, not compiler-verified — a typo'd selector name in `@selector(...)` compiles fine and simply fails (or silently does nothing) at runtime.

### Interview questions
- "What does `objc_msgSend` actually do, step by step?"
- "What is the `isa` pointer, and what is a metaclass?"
- "Describe the three-step message forwarding chain when a selector isn't found."

### Simple explanation
Every Objective-C method call is really "ask this object, at runtime, if it knows how to do this" rather than "jump directly to this code" — which is why so much of Objective-C's dynamism (swizzling, `respondsToSelector:`, associated objects) is possible: the lookup step is a real, inspectable, interceptable function call.

### Key takeaways
- `[obj method]` → `objc_msgSend(obj, @selector(method))` under the hood — dynamic, not static, dispatch.
- `isa` links an instance to its `Class`; a `Class`'s `isa` links to its metaclass for `+` methods.
- Unresolved selectors get one last chance via dynamic resolution / message forwarding before crashing.
- This dynamism is the mechanism behind swizzling, associated objects, and KVO — not separate, unrelated features.

---
## Section 18: Common Cocoa Patterns

Cocoa (Apple's original ObjC frameworks, ancestor of today's UIKit/Foundation/AppKit) established a handful of recurring design patterns you'll see over and over in legacy code — all conceptually familiar from Swift, expressed with ObjC's tools.

### Singleton
```swift
class NetworkManager {
    static let shared = NetworkManager()
    private init() {}
}
```
```objc
@implementation NetworkManager

+ (instancetype)sharedManager {
    static NetworkManager *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[NetworkManager alloc] init];
    });
    return sharedInstance;
}

@end
```
`dispatch_once` guarantees the block runs **exactly once**, thread-safely, no matter how many threads call `sharedManager` simultaneously — the ObjC-era mechanism for what Swift's `static let` gives you for free (Swift's global/static `let` initialization is itself thread-safe by language guarantee, making `dispatch_once` unnecessary).

### Factory
```objc
+ (instancetype)buttonWithStyle:(ButtonStyle)style {
    switch (style) {
        case ButtonStylePrimary: return [[PrimaryButton alloc] init];
        case ButtonStyleSecondary: return [[SecondaryButton alloc] init];
    }
}
```
Same concept as a Swift static factory method — a class method that returns different concrete instances based on input, hiding the exact subclass from the caller.

### Delegate & Observer
Both already covered in depth (Sections 6 and 13) — delegate is one-to-one, synchronous, tightly-coupled communication; `NSNotificationCenter` (Observer pattern) is one-to-many, broadcast-style, loosely-coupled.

### MVC (Model-View-Controller)
Apple's original architecture for Cocoa apps, and still the default shape of most legacy Objective-C projects, predating MVVM entirely:
- **Model**: plain data objects (often `NSManagedObject` subclasses or simple value-holding classes).
- **View**: `UIView` subclasses, purely presentational.
- **Controller**: `UIViewController` subclasses, gluing model to view — in older ObjC code, this is where you'll often find *far* more logic than a Swift MVVM codebase would put in a view controller ("Massive View Controller" is the well-known criticism of this pattern in practice).

### Target-Action
Already covered in Section 10 — a control (e.g., `UIButton`) holds a target object and a selector, and invokes that selector when an event fires. Conceptually the direct ancestor of SwiftUI's action closures and UIKit's own `addTarget:action:forControlEvents:`.

### Responder Chain
```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [super touchesBegan:touches withEvent:event]; // pass up the chain if unhandled
}
```
UIKit's event-handling hierarchy — a view that doesn't handle an event can pass it to its next responder (typically its superview, eventually the view controller, then the window, then the app) — same mechanism Swift UIKit code relies on; no ObjC-specific behavior here beyond the syntax.

### KVO (Key-Value Observing)
```objc
[person addObserver:self
         forKeyPath:@"name"
            options:NSKeyValueObservingOptionNew
            context:nil];

- (void)observeValueForKeyPath:(NSString *)keyPath
                       ofObject:(id)object
                         change:(NSDictionary *)change
                        context:(void *)context {
    if ([keyPath isEqualToString:@"name"]) {
        NSLog(@"New name: %@", change[NSKeyValueChangeNewKey]);
    }
}

- (void)dealloc {
    [person removeObserver:self forKeyPath:@"name"];
}
```
KVO lets you observe changes to any `@property` on any `NSObject` subclass by its **string key path**, powered directly by the runtime (Section 17) — Apple actually swizzles the observed object's class under the hood to intercept its setters. Swift's closest native parallel is `@Published`/`Combine`, or manually observing via `NSObject.observe(\.keyPath)` (Swift's typed KVO wrapper over this exact same mechanism).

### KVC (Key-Value Coding)
```objc
NSString *name = [person valueForKey:@"name"];
[person setValue:@"Sam" forKey:@"name"];
[person valueForKeyPath:@"address.city"]; // nested key path
```
Accessing properties **by string name** rather than by direct property syntax — used heavily for dynamic data binding (older Cocoa Bindings on macOS), Core Data, and generic collection operations (`@sum`, `@avg` KVC collection operators). No compile-time safety at all — a typo'd key string simply fails/crashes at runtime.

### Dependency Injection
```objc
@interface UserService : NSObject
- (instancetype)initWithNetworkClient:(id<NetworkClientProtocol>)client;
@end
```
Same principle as Swift DI — pass dependencies in through an initializer (or property) rather than having a class construct its own dependencies internally — nothing ObjC-specific here beyond syntax.

### Common mistakes
- Forgetting `[person removeObserver:self forKeyPath:@"name"]` before `person` deallocates while still being observed — a classic legacy-code crash ("An instance ... was deallocated while key value observers were still registered").
- Typo'ing a KVC key path string — compiles fine, fails/crashes only when actually executed.
- Not calling `dispatch_once`/using a lazy-but-not-thread-safe singleton pattern in genuinely old (pre-`dispatch_once`) code.

### Interview questions
- "How does KVO actually intercept property changes under the hood?"
- "Why is `dispatch_once` used for singletons in Objective-C, when Swift's `static let` doesn't need it?"

### Simple explanation
These are the same design patterns you already use in Swift-based iOS apps (singleton, delegate, observer, MVC, DI) — Cocoa just gives them specific, decades-old idioms (`dispatch_once`, `NSNotificationCenter`, KVO/KVC by string key) that predate Swift's more type-safe equivalents.

### Key takeaways
- Singletons use `dispatch_once` for thread-safe one-time initialization — Swift's `static let` gets this for free from the language.
- KVO/KVC are runtime-powered, string-keyed property observation/access — powerful, but with zero compile-time safety.
- MVC (not MVVM) is the default architecture in most legacy Objective-C, which is why view controllers in old code tend to be much larger.

---
## Section 19: Swift ↔ Objective-C Interoperability

### What is it?
Most real-world "legacy Objective-C" work today happens in **mixed** projects — new features written in Swift, sitting alongside an older Objective-C codebase. Interoperability is what makes that possible, and understanding its mechanics explains a lot of otherwise-mysterious annotations you'll see in ObjC headers.

### The bridging header
When a Swift target needs to call Objective-C code, Xcode generates (or you manually create) a **bridging header** — `YourProject-Bridging-Header.h` — that `#import`s every Objective-C header you want visible to Swift:
```objc
// MyApp-Bridging-Header.h
#import "Person.h"
#import "NetworkManager.h"
```
Set in Build Settings → "Objective-C Bridging Header." This is a **one-way** door: it exposes Objective-C to Swift, not the reverse.

### Exposing Swift to Objective-C
The reverse direction uses an **auto-generated header**, `YourModuleName-Swift.h`, which Xcode creates automatically for every Swift file — `#import` it from an Objective-C file to call Swift code:
```objc
#import "MyApp-Swift.h"

SwiftHelper *helper = [[SwiftHelper alloc] init];
[helper doSomething];
```
Only Swift declarations marked (explicitly or implicitly) as visible to Objective-C actually appear in this generated header.

### `@objc` and `dynamic`
```swift
class SwiftHelper: NSObject {
    @objc func doSomething() { ... }
    @objc dynamic var name: String = ""
}
```
- **`@objc`** — exposes this Swift member to the Objective-C runtime (required for anything you want callable from ObjC, used for `#selector`, target-action, KVO, and Core Data). Only classes inheriting from `NSObject` (directly or indirectly) can use `@objc` freely; plain Swift classes/structs generally can't.
- **`dynamic`** — forces this member through `objc_msgSend`-style **dynamic dispatch** rather than Swift's default static dispatch, which is required for KVO (Section 18) to work, and for runtime techniques like swizzling to apply to it.

### `NS_SWIFT_NAME`
```objc
- (void)fetchUserWithID:(NSString *)userID
              completion:(void (^)(User * _Nullable, NSError * _Nullable))completion
    NS_SWIFT_NAME(fetchUser(id:completion:));
```
Lets an Objective-C API author control exactly how their method is renamed/reshaped when it's imported into Swift — Objective-C's often-verbose selector-style names get translated into more Swift-idiomatic signatures automatically (`fetchUserWithID:completion:` → `fetchUser(id:completion:)` by default convention; `NS_SWIFT_NAME` lets you override that when the automatic translation isn't ideal).

### Nullability annotations (`nullable`/`nonnull`)
```objc
NS_ASSUME_NONNULL_BEGIN

@interface UserService : NSObject
- (nullable User *)cachedUserWithID:(NSString *)userID;
- (void)fetchUserWithID:(NSString *)userID completion:(void (^)(User * _Nullable user))completion;
@end

NS_ASSUME_NONNULL_END
```
Objective-C object pointers have no compile-time optional/non-optional distinction on their own — `nullable`/`nonnull` (and the block `NS_ASSUME_NONNULL_BEGIN`/`END` pair, which defaults everything in between to `nonnull` unless marked otherwise) exist **purely to give Swift accurate optional types** when importing this header. Without these annotations, every Objective-C object type bridges into Swift as an implicitly-unwrapped optional (`User!`), which is both less safe and less idiomatic — this is why well-maintained legacy ObjC headers you'll be asked to touch are typically full of these annotations.

### Generics limitations
```objc
@interface Box<T> : NSObject
- (T)value;
@end
```
Objective-C's "lightweight generics" (Section 1/8) are a much thinner feature than Swift generics — there's no real generic *code specialization*; `T` is erased at compile time to `id` internally, and the annotation exists mainly so Swift sees a properly-typed API (`Box<NSString *> *` bridges to `Box<String>` in Swift) rather than raw `id`/`Any`. You cannot write ObjC generic code that behaves differently per concrete type the way Swift generics with protocol constraints can.

### Mixed-project practicalities & migration tips
- New Swift code that needs to be called from old Objective-C must inherit from `NSObject` and mark members `@objc` as needed.
- Structs, enums with associated values, and other pure-Swift-only features **cannot** cross the bridge at all — only classes, `@objc`-compatible enums, and basic types (`Int`, `String`, `Bool`, arrays/dictionaries of bridgeable types) are visible to Objective-C.
- A common migration strategy: introduce new features in Swift, wrap existing Objective-C singletons/managers behind a thin Swift-facing interface, and gradually port outward from leaf classes (ones with few dependents) rather than attempting a risky top-down rewrite.
- Adding `nullable`/`nonnull` annotations to an old, un-annotated Objective-C header you're actively working in is a low-risk, high-value cleanup that immediately improves the Swift-side experience of calling that code.

### Common mistakes
- Forgetting `@objc` on a Swift method you intend to reference via `#selector` from either Swift or Objective-C code — fails silently at the `#selector` compile step or with an "unrecognized selector" at runtime.
- Assuming an un-annotated Objective-C API will bridge as a safely-optional Swift type — it bridges as implicitly-unwrapped (`Type!`), a latent crash risk if actually `nil`.
- Trying to expose a Swift `struct` or associated-value `enum` directly to Objective-C — not supported; needs a class-based or plain-integer-`enum` wrapper instead.

### Interview questions
- "What's the difference between the bridging header and the `-Swift.h` generated header?"
- "Why does `dynamic` matter for KVO on a Swift property?"
- "What happens to a Swift `Int?` when it bridges to an un-annotated Objective-C `nullable`-free header?"

### Simple explanation
The bridging header lets Swift see Objective-C; the auto-generated `-Swift.h` header lets Objective-C see Swift. `@objc`/`dynamic` opt Swift members into the ObjC runtime; `nullable`/`nonnull` teach Swift which ObjC pointers can actually be `nil`.

### Key takeaways
- Bridging header: ObjC → Swift visibility. `-Swift.h`: Swift → ObjC visibility. Different files, different directions.
- `@objc` exposes a Swift member to the runtime; `dynamic` forces dynamic dispatch (required for KVO/swizzling).
- `nullable`/`nonnull` annotations are the single highest-value cleanup for improving how legacy ObjC bridges into Swift.
- Only class-based, runtime-compatible Swift features can cross into Objective-C — structs and associated-value enums cannot.

---
## Section 20: Daily Development Examples

A working reference of the conversions you'll reach for most often day to day. Each pair is deliberately terse — the deep explanations live in the sections above; this is the quick-lookup version.

### Property declaration
```swift
var title: String
weak var delegate: SomeDelegate?
let identifier: String
```
```objc
@property (nonatomic, copy) NSString *title;
@property (nonatomic, weak) id<SomeDelegate> delegate;
@property (nonatomic, readonly) NSString *identifier;
```

### TableView cell configuration
```swift
cell.textLabel?.text = item.name
cell.accessoryType = .disclosureIndicator
```
```objc
cell.textLabel.text = item.name;
cell.accessoryType = UITableViewCellAccessoryDisclosureIndicator;
```

### CollectionView cell
```swift
let cell = collectionView.dequeueReusableCell(withReuseIdentifier: "Cell", for: indexPath) as! MyCell
cell.configure(with: item)
```
```objc
MyCell *cell = [collectionView dequeueReusableCellWithReuseIdentifier:@"Cell" forIndexPath:indexPath];
[cell configureWithItem:item];
```

### API call (see Section 11 for full detail)
```swift
URLSession.shared.dataTask(with: url) { data, _, error in ... }.resume()
```
```objc
[[[NSURLSession sharedSession] dataTaskWithURL:url completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) { ... }] resume];
```

### Navigation
```swift
navigationController?.pushViewController(vc, animated: true)
```
```objc
[self.navigationController pushViewController:vc animated:YES];
```

### Passing data forward
```swift
let detail = DetailVC()
detail.item = selectedItem
```
```objc
DetailVC *detail = [[DetailVC alloc] init];
detail.item = self.selectedItem;
```

### Delegate declaration (see Section 6)
```swift
protocol CellDelegate: AnyObject {
    func cellDidTap(_ cell: MyCell)
}
```
```objc
@protocol CellDelegate <NSObject>
- (void)cellDidTap:(MyCell *)cell;
@end
```

### Block / completion handler
```swift
func load(completion: @escaping (Bool) -> Void) { }
```
```objc
- (void)loadWithCompletion:(void (^)(BOOL success))completion;
```

### NotificationCenter
```swift
NotificationCenter.default.post(name: .didUpdate, object: nil)
```
```objc
[[NSNotificationCenter defaultCenter] postNotificationName:@"DidUpdateNotification" object:nil];
```

### Singleton
```swift
static let shared = Manager()
```
```objc
+ (instancetype)sharedManager {
    static Manager *instance;
    static dispatch_once_t token;
    dispatch_once(&token, ^{ instance = [[Manager alloc] init]; });
    return instance;
}
```

### Core Data fetch
```swift
let results = try? context.fetch(request)
```
```objc
NSError *error;
NSArray *results = [context executeFetchRequest:request error:&error];
```

### FileManager
```swift
let exists = FileManager.default.fileExists(atPath: path)
```
```objc
BOOL exists = [[NSFileManager defaultManager] fileExistsAtPath:path];
```

### DateFormatter
```swift
let formatter = DateFormatter()
formatter.dateFormat = "yyyy-MM-dd"
let string = formatter.string(from: Date())
```
```objc
NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
formatter.dateFormat = @"yyyy-MM-dd";
NSString *string = [formatter stringFromDate:[NSDate date]];
```

### JSON parsing (manual, see Section 11)
```swift
let dict = try JSONSerialization.jsonObject(with: data) as? [String: Any]
```
```objc
NSDictionary *dict = [NSJSONSerialization JSONObjectWithData:data options:0 error:&error];
```

### Alert
```swift
let alert = UIAlertController(title: "Error", message: "Something went wrong", preferredStyle: .alert)
alert.addAction(UIAlertAction(title: "OK", style: .default))
present(alert, animated: true)
```
```objc
UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"Error"
                                                                 message:@"Something went wrong"
                                                          preferredStyle:UIAlertControllerStyleAlert];
[alert addAction:[UIAlertAction actionWithTitle:@"OK" style:UIAlertActionStyleDefault handler:nil]];
[self presentViewController:alert animated:YES completion:nil];
```

### Button action
```swift
button.addTarget(self, action: #selector(tapped), for: .touchUpInside)
```
```objc
[button addTarget:self action:@selector(tapped) forControlEvents:UIControlEventTouchUpInside];
```

### Constraints (see Section 10)
```swift
NSLayoutConstraint.activate([view.topAnchor.constraint(equalTo: superview.topAnchor)])
```
```objc
[NSLayoutConstraint activateConstraints:@[[view.topAnchor constraintEqualToAnchor:superview.topAnchor]]];
```

### Animation
```swift
UIView.animate(withDuration: 0.3) { view.alpha = 0 }
```
```objc
[UIView animateWithDuration:0.3 animations:^{ view.alpha = 0; }];
```

### Timer
```swift
Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { _ in tick() }
```
```objc
[NSTimer scheduledTimerWithTimeInterval:1.0
                                 repeats:YES
                                   block:^(NSTimer *timer) { [self tick]; }];
```

### UserDefaults
```swift
UserDefaults.standard.set(true, forKey: "hasOnboarded")
let value = UserDefaults.standard.bool(forKey: "hasOnboarded")
```
```objc
[[NSUserDefaults standardUserDefaults] setBool:YES forKey:@"hasOnboarded"];
BOOL value = [[NSUserDefaults standardUserDefaults] boolForKey:@"hasOnboarded"];
```

### Dependency injection
```swift
init(networkClient: NetworkClientProtocol) { self.client = networkClient }
```
```objc
- (instancetype)initWithNetworkClient:(id<NetworkClientProtocol>)client;
```

### Custom, reusable cell
```swift
class MyCell: UITableViewCell {
    func configure(with item: Item) { ... }
}
```
```objc
@interface MyCell : UITableViewCell
- (void)configureWithItem:(Item *)item;
@end
```

---
## Section 21: Debugging Objective-C

### Reading compiler errors and warnings

| Error/Warning | What it means | Typical fix |
|---|---|---|
| `Unrecognized selector sent to instance` | You called a method the object's class doesn't implement (typo'd selector, wrong class, or category not linked in) | Check spelling of the selector; confirm the object is the class you think it is (`NSLog(@"%@", [obj class])`) |
| `EXC_BAD_ACCESS` | Sent a message to (or dereferenced) a deallocated/invalid pointer — a "zombie" or garbage memory | Enable Zombie Objects (Section 9) to get a clearer crash; check `weak`/`assign` usage |
| `Property 'x' not found on object of type 'Y'` | Header not imported, or property genuinely doesn't exist on that class | `#import` the right header, or check for a typo in the property name |
| `Incompatible pointer types sending 'X *' to parameter of type 'Y *'` | Passing the wrong object type into a method | Fix the type, or add an explicit (and justified) cast |
| `Use of undeclared identifier` | Missing `#import`, missing `@class` forward declaration, or a typo | Add the right import |
| `Duplicate symbol for architecture` | Same symbol (function/global) defined in two compiled files | Usually a `#import` of a `.m` file, or the same global defined in two headers without `extern` |
| `Undefined symbols for architecture x86_64/arm64` (linker error) | You declared something but never provided (linked) an implementation, or forgot a framework | Confirm the `.m` file is part of the build target; add missing frameworks |
| `nil` passed for a `nonnull` parameter | Bridging annotation violation, usually a warning in ObjC itself but can be a hard error from Swift call sites | Handle the `nil` case, or fix the annotation if it was actually meant to be `nullable` |

### Header import problems
- **Circular imports** (`A.h` imports `B.h`, `B.h` imports `A.h`) — fix by replacing one side with a forward declaration (`@class B;`) and moving the full `#import` into the `.m` file (Section 3).
- **Missing framework import** — e.g. using `NSURLSession` without `#import <Foundation/Foundation.h>` (usually pulled in transitively via a project's prefix/umbrella header, but worth checking first when something "should exist" and doesn't).

### Linker errors
"Undefined symbols" almost always means: the header declares something, but no `.m` file that's actually part of the current build **target** provides the implementation — a very common issue when a new class isn't added to the right target's "Compile Sources" build phase, or a needed system framework (e.g., `CoreLocation.framework`) isn't linked.

### Memory issues
- **`EXC_BAD_ACCESS`** — see the Zombie Objects and Memory Graph Debugger workflow from Section 9.
- **Leaks** — use Instruments' Leaks tool, or the Memory Graph Debugger's cycle detection, to find retain cycles (delegates not `weak`, blocks capturing `self` strongly).

### "Unrecognized selector sent to instance" — the most common legacy crash
This crash means: the runtime walked the entire class hierarchy (Section 17) and genuinely could not find an implementation for the selector you sent. Common root causes:
- Calling a method on the wrong object (e.g., you meant `self.tableView` but wrote `self.view`).
- A category providing the method wasn't actually linked into the binary (categories in static libraries can be silently dropped by the linker unless `-ObjC` / `-all_load` linker flags are set — a classic, hard-to-diagnose legacy build issue).
- Simple typo in `@selector(...)` — since it's just a string under the hood, the compiler can't catch a wrong name.

### Nil messaging
Sending a message to `nil` in Objective-C is **not a crash** — it's legal and simply returns a zeroed value (`nil`/`0`/`NO`/`0.0` depending on the expected return type). This is a deliberate, load-bearing language design decision, very different from Swift's strict optional-unwrapping crash-on-`nil` philosophy:
```objc
Person *person = nil;
NSString *name = [person name]; // returns nil, does NOT crash
NSInteger age = [person age];    // returns 0, does NOT crash
```
This is a double-edged sword: it prevents a whole class of crashes Swift would otherwise force you to handle explicitly, but it can also **mask bugs** — a chain of nil-messaging can silently produce "nothing happened" instead of a clear failure, making some legacy bugs harder to trace than an equivalent Swift crash would have been.

### ARC-related compile errors
- `ARC forbids explicit message send of 'release'` — you're compiling ARC code that tries to manually manage memory; delete the manual call, ARC handles it.
- `Automatic Reference Counting Issue: Cast of C pointer type ... to Objective-C pointer type ... requires a bridged cast` — mixing Core Foundation (`CF`) types with Objective-C (`NS`) objects requires `__bridge`/`__bridge_transfer`/`__bridge_retained` casts to tell ARC who owns the memory across the C/ObjC boundary.

### Common mistakes
- Assuming a crash means a `nil` was messaged — most `nil`-messaging is silent; a real crash is more often "unrecognized selector" (wrong class) or `EXC_BAD_ACCESS` (deallocated object).
- Not enabling Zombie Objects when chasing an intermittent `EXC_BAD_ACCESS` — without it, the crash location/stack trace is often useless (it points at whatever memory happened to be reused, not the real culprit).
- Ignoring linker warnings about missing `-ObjC` flag when categories from a static library silently don't work.

### Simple explanation
Most Objective-C debugging is either "the runtime couldn't find this method" (unrecognized selector — check spelling/class), or "you're touching memory that's already gone" (`EXC_BAD_ACCESS`/zombie — check `weak`/retain cycles), or a straightforward missing-import/missing-target-membership build issue.

### Key takeaways
- `nil` messaging is silent and legal — it returns a zeroed value, it does not crash.
- "Unrecognized selector" = runtime couldn't find any implementation anywhere in the class chain.
- `EXC_BAD_ACCESS` = messaging deallocated/garbage memory — Zombie Objects + Memory Graph Debugger are the tools.
- Categories in static libraries can be silently dropped without the `-ObjC` linker flag.

---
## Section 22: Reading Real Production Code

Below is a realistic, moderately dense Objective-C class from a legacy networking layer, read line by line — the kind of file you'll actually be handed on the job.

```objc
// UserSessionManager.h
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@protocol UserSessionManagerDelegate <NSObject>
@optional
- (void)sessionManager:(id)manager didExpireWithReason:(NSString *)reason;
@end

@interface UserSessionManager : NSObject

@property (nonatomic, weak, nullable) id<UserSessionManagerDelegate> delegate;
@property (nonatomic, copy, readonly, nullable) NSString *currentToken;

+ (instancetype)sharedManager;
- (void)loginWithUsername:(NSString *)username
                  password:(NSString *)password
                completion:(void (^)(BOOL success, NSError * _Nullable error))completion;
- (void)logout;

@end

NS_ASSUME_NONNULL_END
```

```objc
// UserSessionManager.m
#import "UserSessionManager.h"

static NSString *const kTokenStorageKey = @"com.myapp.authToken";

@interface UserSessionManager ()
@property (nonatomic, copy, readwrite, nullable) NSString *currentToken;
@property (nonatomic, strong) NSTimer *expiryTimer;
@end

@implementation UserSessionManager

+ (instancetype)sharedManager {
    static UserSessionManager *instance;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[UserSessionManager alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _currentToken = [[NSUserDefaults standardUserDefaults] stringForKey:kTokenStorageKey];
    }
    return self;
}

- (void)loginWithUsername:(NSString *)username
                  password:(NSString *)password
                completion:(void (^)(BOOL success, NSError * _Nullable error))completion {

    __weak typeof(self) weakSelf = self;

    NSURL *url = [NSURL URLWithString:@"https://api.example.com/login"];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"POST";

    NSDictionary *body = @{@"username": username, @"password": password};
    NSError *jsonError;
    request.HTTPBody = [NSJSONSerialization dataWithJSONObject:body options:0 error:&jsonError];

    if (jsonError) {
        if (completion) { completion(NO, jsonError); }
        return;
    }

    [[[NSURLSession sharedSession] dataTaskWithRequest:request
        completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {

        __strong typeof(self) strongSelf = weakSelf;
        if (!strongSelf) { return; }

        if (error) {
            dispatch_async(dispatch_get_main_queue(), ^{
                if (completion) { completion(NO, error); }
            });
            return;
        }

        NSDictionary *json = [NSJSONSerialization JSONObjectWithData:data options:0 error:nil];
        NSString *token = json[@"token"];

        strongSelf.currentToken = token;
        [[NSUserDefaults standardUserDefaults] setObject:token forKey:kTokenStorageKey];

        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) { completion(YES, nil); }
        });

    }] resume];
}

- (void)logout {
    self.currentToken = nil;
    [[NSUserDefaults standardUserDefaults] removeObjectForKey:kTokenStorageKey];
    [self.expiryTimer invalidate];

    if ([self.delegate respondsToSelector:@selector(sessionManager:didExpireWithReason:)]) {
        [self.delegate sessionManager:self didExpireWithReason:@"User logged out"];
    }
}

- (void)dealloc {
    [self.expiryTimer invalidate];
}

@end
```

### Line-by-line explanation of the non-obvious parts

- **`NS_ASSUME_NONNULL_BEGIN`/`END`** — everything inside defaults to `nonnull` unless explicitly marked `nullable`; this is why individual `nonnull` annotations are largely absent — only the exceptions (`nullable`) are called out.
- **`@interface UserSessionManager ()` in the `.m`** — a class extension redeclaring `currentToken` as `readwrite` privately, while the public header exposes it as `readonly` — a very common pattern for "read-only outside, mutable inside."
- **`static NSString *const kTokenStorageKey`** — file-private constant (Section 1), avoiding a magic string literal repeated in multiple places.
- **`dispatch_once` singleton** — see Section 18.
- **`__weak typeof(self) weakSelf` / `__strong typeof(self) strongSelf`** — the retain-cycle-avoidance dance from Section 5; the completion block is held by `NSURLSession` for the duration of the network call, so capturing `self` strongly here (indirectly, through nothing else, but as a matter of hygiene/convention for any long-lived async block) would risk a longer-than-necessary retain, though the real risk pattern is when the *containing object* also stores a strong reference back to this session manager or its blocks.
- **`&jsonError`** — the `NSError **` out-parameter pattern (Section 15); note the early return with `completion(NO, jsonError)` on the failure path.
- **`dispatch_async(dispatch_get_main_queue(), ...)`** — because `NSURLSession` completion handlers run off the main thread, but `completion` here is expected to be safe to use for UI updates.
- **`respondsToSelector:` guard before calling an `@optional` delegate method** — Section 6's pattern, since `didExpireWithReason:` was declared `@optional`.
- **`dealloc` invalidating the timer** — cleanup that would be a `deinit` block in Swift; note there's no need to nil-out `_currentToken` or similar — ARC handles releasing strong references automatically.

### Why it's written this way
This class follows several idioms you'll see repeatedly: a `dispatch_once` singleton, a public read-only property backed by a private read-write redeclaration, string constants defined once at file scope, and the `__weak`/`__strong` capture dance around any block that might outlive the current scope. None of it is arbitrary — each pattern solves a specific problem (thread-safe lazy init, encapsulation, avoiding typo'd literals, avoiding retain cycles) that Swift's language features (`static let`, `private(set)`, `let` constants, and stricter default dispatch) solve more automatically.

---
## Section 23: Common Mistakes Swift Developers Make

A concentrated list of the mistakes that actually show up in code review when a Swift developer starts writing Objective-C. Grouped by category, with the fix and a memory hook for each.

### Syntax & typing
1. **Forgetting the `*` on object pointers.** `NSString name;` → should be `NSString *name;`. *Remember: objects are always pointers.*
2. **Writing `if x` instead of `if (x)`.** Parens are mandatory. *Remember: it's still plain C.*
3. **Trying to `switch` on an `NSString`.** Use `if`/`else if` + `isEqualToString:`. *Remember: switch only works on integers/enums.*
4. **Using `==` to compare object contents.** `if (str1 == str2)` compares pointers, not text. Use `isEqualToString:`/`isEqual:`. *Remember: `==` is identity, like Swift's `===`.*
5. **Assuming method overloading by type works.** It doesn't — give different-typed variants different selector names. *Remember: dispatch is by name, resolved at runtime.*

### Memory & ownership
6. **Declaring a delegate `strong` instead of `weak`.** Creates a retain cycle. *Remember: delegates are always `weak`.*
7. **Declaring an `NSString`/block property `strong` instead of `copy`.** Risks mutation-after-assignment / stack deallocation. *Remember: strings and blocks are `copy`.*
8. **Capturing `self` strongly in a long-lived block without `__weak`/`__strong`.** *Remember: it's the same fix as Swift's `[weak self]` + `guard let self`.*
9. **Forgetting `__block` on a captured local you need to mutate inside a block.** Compile error: "variable is not assignable." *Remember: default capture of scalars is a read-only snapshot.*
10. **Manually calling `retain`/`release` in ARC code.** Compile error — ARC forbids it entirely. *Remember: ARC does this for you now, same as Swift.*

### Initialization
11. **Skipping `self = [super init]`.** Just calling `[super init];` without capturing/reassigning `self`. *Remember: `super`'s init might return a different object.*
12. **Forgetting to check `if (self)` before configuring the instance.** Risks configuring a `nil` object if `super`'s init failed. *Remember: init can fail and return `nil`.*
13. **Forgetting `return self;` at the end of a custom initializer.** *Remember: init is still just a method — nothing returns implicitly.*

### Properties
14. **Forgetting `nonatomic`.** Defaults to `atomic`, paying a real performance cost for a guarantee (single-access thread safety) that rarely matches actual needs. *Remember: nearly all real-world ObjC properties are `nonatomic`.*
15. **Assuming `atomic` makes compound operations thread-safe.** It only protects a single get/set, not multi-step logic. *Remember: `atomic` ≠ "thread-safe."*

### Collections
16. **Trying to insert a raw scalar into an `NSArray`/`NSDictionary` literal.** Needs boxing: `@(myInt)`, not `myInt` directly. *Remember: only objects go in collections.*
17. **Assuming `NSArray` has `map`/`filter`/`reduce`.** It doesn't natively — use block-based APIs or a manual loop. *Remember: Foundation predates functional-style collection methods.*
18. **Forgetting collections are reference types needing explicit `copy`/`mutableCopy`** when you don't want a caller to mutate your internal array out from under you.

### Error handling
19. **Ignoring the `NSError **` out-parameter entirely.** Nothing forces you to check it — silent failures are easy. *Remember: this is ObjC's `throws`, but unenforced.*
20. **Using `@try`/`@catch` for ordinary recoverable errors instead of `NSError **`.** Against Apple's own convention — exceptions are for programmer errors only.

### Protocols & delegates
21. **Calling an `@optional` delegate method without `respondsToSelector:` first.** Crashes with "unrecognized selector" if unimplemented.
22. **Confusing protocol-conformance `<Protocol>` syntax with generics `<Type>` syntax** — context (after `id` vs. after a collection type) disambiguates them.

### Blocks
23. **Storing a block property as `strong`/`assign` instead of `copy`.** Historically risked a dangling stack block; still the documented, correct attribute.
24. **Not re-promoting `weakSelf` to a local `strongSelf`** before using it multiple times inside a block, risking `self` disappearing mid-execution.

### Runtime & categories
25. **Two categories implementing the same selector on the same class.** Undefined which one "wins" — avoid overlapping category method names.
26. **Expecting a `@property` in a category to get real stored-property backing.** It doesn't, without associated objects.
27. **Forgetting to call through to the original implementation after swizzling.** Permanently loses the base behavior.

### KVO/Notifications
28. **Forgetting to remove a KVO observer before the observed object deallocates.** Classic legacy crash: "was deallocated while key value observers were still registered."
29. **Typo'ing a notification-name string or a KVC key-path string.** Compiles fine, silently fails at runtime — always use shared `static NSString *const` constants.

### General
30. **Expecting a `nil`-messaging bug to crash the way an unwrapped-optional bug would in Swift.** It doesn't — it silently returns a zeroed value, which can mask logic errors instead of surfacing them clearly.

---
## Section 24: Mental Model — How to Think in Objective-C

### Messaging, not calling
The single biggest mental shift: stop thinking "I'm calling a function on this object" and start thinking "I'm sending a named request to this object, and it's up to the object (looked up dynamically, at runtime) to decide how to respond." This isn't just a syntax quirk — it's *why* `nil`-messaging is safe (there's simply no responder, so nothing happens), *why* swizzling/forwarding/KVO all work (the lookup step is interceptable), and *why* method names read like natural-language sentences (`insertObject:atIndex:` — Smalltalk-style messaging was designed to read conversationally).

### Naming conventions carry real meaning
Objective-C's naming isn't just style — it's semantic:
- `is`-prefixed methods return `BOOL` (`isEqual:`, `isKindOfClass:`).
- `init`-prefixed methods are initializers, part of a compiler-recognized family with specific memory-management rules (their return value is expected to potentially replace the receiver).
- `alloc`/`new`/`copy`/`mutableCopy`-prefixed or containing methods are understood (historically, under manual memory management, and still by convention) to return an object the caller owns — this "Cocoa naming convention" is precisely what let ARC be retrofitted onto existing code: the compiler could infer ownership rules from method *names* alone.
- Class names are prefixed (`NS`, `UI`, `CA`, `CG`) because Objective-C has no namespaces — the prefix is doing the job Swift's module system does automatically.

### Apple's API style, and why it's verbose
Long, explicit method names (`tableView:didSelectRowAtIndexPath:`) aren't accidental verbosity — in a language with no namespaces, no argument-label inference, and dynamic dispatch resolved purely by name-matching, an unambiguous, self-documenting name is the only thing standing between you and a silent collision or a confusing call site. Swift's argument labels are a direct, more ergonomic descendant of this same instinct.

### Historical reasons behind the "weirdness"
- **No namespaces** → class name prefixes (`NS`, `UI`).
- **C's preprocessor is still there** → `#import`, `#define`, forward declarations with `@class`.
- **No generics until 2015 (Xcode 7)** → most legacy collection code uses raw `id`; "lightweight generics" were retrofitted for Swift-bridging clarity, not deep type safety.
- **No optionals, ever** → `nil`-messaging had to be made safe by design, since there was no other way to prevent every unchecked object reference from being a potential crash.
- **Manual memory management until 2011 (ARC)** → naming conventions (`alloc`/`copy`/`new`/`init` implying ownership) exist so ARC could be introduced *without* requiring every existing method to be re-annotated — the compiler could infer the rules from names it already understood.

### The mindset shift, summarized
| Swift instinct | Objective-C reality |
|---|---|
| "The compiler will catch a lot of my mistakes" | Many mistakes only surface at runtime — typo'd selectors, wrong key paths, `nil`-messaging all compile fine |
| "This method call is a direct jump to code" | It's a dynamic lookup through the class hierarchy — interceptable, forwardable, swizzlable |
| "Optionals make `nil` explicit and safe" | `nil` is just a value any object pointer can silently hold; messaging it is legal and simply does nothing |
| "I don't think about thread-safety of a single property access" | `atomic`/`nonatomic` is an explicit per-property choice you must make |
| "Errors propagate with `throws`/`try`/`catch`" | Recoverable errors propagate through an `NSError **` out-parameter you must remember to check |
| "Extensions only add to types I can already see" | Categories can rewrite the method surface of *any* class, including framework classes, app-wide |

### The one-sentence version
Objective-C is a messaging system built on top of C, where naming conventions (not the compiler) carry most of the safety guarantees Swift now enforces structurally — once you internalize "I'm sending a message, and the runtime decides what happens," the rest of the language's syntax stops looking arbitrary and starts looking like a consistent, if verbose, set of conventions.

---
## Section 25: Cheat Sheets

### One-page syntax cheat sheet

| Element | Objective-C |
|---|---|
| Instance method | `- (ReturnType)methodName:(ArgType)arg;` |
| Class method | `+ (ReturnType)methodName;` |
| Property | `@property (attributes) Type *name;` |
| Protocol | `@protocol Name <NSObject> ... @end` |
| Category | `@interface ClassName (CategoryName) ... @end` |
| Class extension | `@interface ClassName () ... @end` (in the `.m`) |
| Block type | `ReturnType (^blockName)(ParamTypes)` |
| Literal array | `@[obj1, obj2]` |
| Literal dictionary | `@{key1: val1}` |
| Literal number | `@5`, `@YES`, `@3.14` |
| Message send | `[receiver message:arg]` |
| Nested message send | `[[receiver message1] message2]` |
| Selector | `@selector(methodName:)` |
| Class object | `[ClassName class]` |
| Forward declaration | `@class ClassName;` |

### Memory management cheat sheet

| Attribute | Use for | Risk if wrong |
|---|---|---|
| `strong` | owned objects (default) | retain cycle if reciprocated |
| `weak` | delegates, back-references, IBOutlets | none if used correctly; dangling avoided automatically |
| `assign` | scalars (`NSInteger`, `CGRect`) | dangling pointer if misused on an object |
| `copy` | `NSString`, blocks | mutation-after-assignment bugs if `strong` used instead |
| `nonatomic` | almost everything, in practice | none — just document intent |
| `atomic` | rarely; single-access thread safety only | false sense of full thread-safety |

### UIKit cheat sheet

| Task | Method |
|---|---|
| Push | `[self.navigationController pushViewController:vc animated:YES];` |
| Present | `[self presentViewController:vc animated:YES completion:nil];` |
| Dismiss | `[self dismissViewControllerAnimated:YES completion:nil];` |
| Dequeue cell | `[tableView dequeueReusableCellWithIdentifier:@"id" forIndexPath:indexPath];` |
| Reload | `[tableView reloadData];` |
| Add subview | `[view addSubview:subview];` |
| Animate | `[UIView animateWithDuration:0.3 animations:^{ ... }];` |

### Blocks cheat sheet

| Pattern | Syntax |
|---|---|
| No args, no return | `void (^block)(void) = ^{ ... };` |
| One arg | `void (^block)(NSString *) = ^(NSString *s) { ... };` |
| Returning a value | `NSInteger (^block)(NSInteger) = ^NSInteger(NSInteger x) { return x * 2; };` |
| Typedef'd block type | `typedef void (^Completion)(BOOL, NSError *);` |
| Weak self capture | `__weak typeof(self) weakSelf = self;` |
| Strong re-promotion | `__strong typeof(self) strongSelf = weakSelf;` |
| Mutable captured local | `__block NSInteger counter = 0;` |

### Property attributes cheat sheet

```
@property (nonatomic, strong) UIView *view;              // owned object
@property (nonatomic, weak) id<Delegate> delegate;        // non-owning, auto-nils
@property (nonatomic, copy) NSString *name;                // immutable snapshot
@property (nonatomic, copy) void (^completion)(BOOL);      // block, heap-copied
@property (nonatomic, assign) NSInteger count;              // scalar
@property (nonatomic, readonly) NSString *identifier;       // getter only
@property (nonatomic, assign, getter=isValid) BOOL valid;   // custom getter name
```

### Runtime cheat sheet

| Function/Concept | Purpose |
|---|---|
| `objc_msgSend` | dynamic dispatch — the actual mechanism behind `[obj method]` |
| `isa` | pointer from an instance to its `Class` |
| `@selector(name)` | produces a `SEL`, a runtime handle to a method name |
| `respondsToSelector:` | runtime check for whether an object implements a selector |
| `performSelector:` | invoke a method by name at runtime |
| `class_getInstanceMethod` | fetch a `Method` for swizzling |
| `method_exchangeImplementations` | swap two methods' implementations (swizzling) |
| `objc_setAssociatedObject`/`objc_getAssociatedObject` | simulate stored properties in a category |

### Delegate cheat sheet

```objc
// 1. Declare the protocol
@protocol ThingDelegate <NSObject>
@required
- (void)thingDidFinish:(Thing *)thing;
@optional
- (void)thing:(Thing *)thing didFailWithError:(NSError *)error;
@end

// 2. Declare a weak delegate property
@property (nonatomic, weak) id<ThingDelegate> delegate;

// 3. Call required methods directly
[self.delegate thingDidFinish:self];

// 4. Guard optional methods
if ([self.delegate respondsToSelector:@selector(thing:didFailWithError:)]) {
    [self.delegate thing:self didFailWithError:error];
}
```

### Foundation classes cheat sheet

| Foundation class | Swift bridge | Purpose |
|---|---|---|
| `NSString` | `String` | text |
| `NSArray`/`NSMutableArray` | `Array` | ordered collection |
| `NSDictionary`/`NSMutableDictionary` | `Dictionary` | key-value collection |
| `NSSet`/`NSMutableSet` | `Set` | unordered unique collection |
| `NSNumber` | `Int`/`Double`/`Bool` (boxed) | boxed scalar for use in collections |
| `NSDate` | `Date` | point in time |
| `NSError` | `Error`/`NSError` | error domain/code/userInfo |
| `NSData` | `Data` | raw bytes |
| `NSURL` | `URL` | resource locator |
| `NSNotification` | `Notification` | pub/sub payload |
| `NSPredicate` | `NSPredicate` (unbridged, used directly) | filter/query expression |

### Common interview questions (consolidated)

1. What does `objc_msgSend` do, and how does it differ from a Swift function call?
2. Why must delegate properties be `weak`?
3. Why do string/block properties use `copy` instead of `strong`?
4. What's the difference between a category and a class extension?
5. Why can't Objective-C overload methods by argument type?
6. Explain `__block`, `__weak`, and `__strong` in the context of blocks.
7. How does KVO intercept property changes under the hood?
8. Why is `dispatch_once` used for singletons in Objective-C but not needed in Swift?
9. What's the actual failure signal in a method with an `NSError **` parameter?
10. What happens when you send a message to `nil`?
11. What's the difference between `atomic` and thread-safety in general?
12. How does message forwarding work when a selector isn't found?
13. Why does an Objective-C initializer reassign `self`?
14. What's the risk of two categories implementing the same selector?
15. Why do `nullable`/`nonnull` annotations matter for Swift interop?

### Common compiler errors (consolidated)

See Section 21 for full detail. Quick index: `Unrecognized selector` (wrong/missing method), `EXC_BAD_ACCESS` (deallocated object), `Incompatible pointer types` (wrong type passed), `Duplicate symbol` (multiple definitions), `Undefined symbols` (missing implementation/framework link), `ARC forbids explicit message send` (manual retain/release in ARC code).

### Swift → Objective-C conversion table
*(See [Section 0](#0-quick-conversion-cheat-sheet) at the top of this document for the full table.)*

### Objective-C → Swift conversion table

| Objective-C | Swift |
|---|---|
| `NSString *` | `String` |
| `NSMutableArray *` | `var array: [T]` (mutability via `var`) |
| `id` | `Any` / `AnyObject` |
| `BOOL` | `Bool` |
| `NSInteger` | `Int` |
| `nil` (object) | `nil` (optional) |
| `- (void)method` | `func method()` |
| `+ (void)method` | `static func method()` |
| `@property (nonatomic, weak)` | `weak var` |
| `@property (nonatomic, copy) NSString *` | `var name: String` |
| Block `^(NSInteger x) { }` | closure `{ (x: Int) in }` |
| `@protocol` | `protocol` |
| Category | `extension` |
| `NSError **` + return sentinel | `throws` |
| `dispatch_async(dispatch_get_main_queue(), ^{})` | `DispatchQueue.main.async { }` |
| `[obj isKindOfClass:[Foo class]]` | `obj is Foo` |
| `(Foo *)obj` | `obj as! Foo` |

---

## Closing Notes

This guide maps roughly 1:1 onto real legacy iOS codebases you're likely to encounter — networking layers built on `NSURLSession` and blocks, MVC-shaped view controllers, Core Data stacks, and Cocoa-era patterns like KVO and `dispatch_once` singletons. The fastest way to internalize it is not to memorize syntax in isolation, but to keep translating: every time you hit an unfamiliar Objective-C construct in a real file, ask "what's the Swift version of this idea," find it in the relevant section above, and let the "why" explanation do the rest of the work.

**Suggested next steps for daily reference:**
- Keep Section 0 (Quick Conversion Cheat Sheet) and Section 25 (Cheat Sheets) open while reading unfamiliar files.
- When debugging a crash, start with Section 21; when reviewing a PR touching memory-sensitive code (blocks, delegates, categories), re-check Sections 5, 9, and 16.
- When writing *new* Objective-C that needs to interoperate cleanly with Swift call sites, apply Section 19's nullability and `NS_SWIFT_NAME` guidance proactively.
