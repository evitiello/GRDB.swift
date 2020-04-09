Migrating From GRDB 4 to GRDB 5
===============================

**This guide aims at helping you upgrading your applications from GRDB 4 to GRDB 5.**

- [Preparing the Migration to GRDB 5](#preparing-the-migration-to-grdb-5)
- [New requirements](#new-requirements)
- [ValueObservation](#valueobservation)
- [Other Changes](#other-changes)


## Preparing the Migration to GRDB 5

If you haven't made it yet, upgrade to GRDB 4.12.1 first, and fix all deprecation warnings, prior to the GRDB 5 upgrade.

GRDB 5 ships with fix-its that will suggest simple syntactic changes, and won't require you to think much.

Your attention is required, though, in two areas: database observation, and custom SQLite builds. Please see below.


## New requirements

GRDB requirements have been bumped:

- **Swift 5.2+** (was Swift 4.2+)
- **Xcode 11.4+** (was Xcode 10.0+)
- iOS 9.0+ (unchanged)
- macOS 10.10+ (was macOS 10.9+)
- tvOS 9.0+ (unchanged)
- watchOS 2.0+ (unchanged)


## ValueObservation

[ValueObservation] is the database observation tool that tracks changes in database values. It has quite changed in GRDB 5.

**The API surface of ValueObservation was reduced**, leaving only two core methods:

```swift
// Define an observation
let observation = ValueObservation.tracking { db in
    /* fetch and return the observed value */
}

// Start observing the database
let cancellable = observation.start(
    in: dbQueue,
    onError: { error in ... },
    onChange: { value in print("fresh value: \(value)") })
```

If the tracked value is computed from several database requests that are not always the same, make sure you use the `trackingVaryingRegion` method, as below. See [Observing a Varying Database Region] for more information.

```swift
// An observation which does not always execute the same requests:
let observation = ValueObservation.trackingVaryingRegion { db -> Int in
    switch try Preference.fetchOne(db)!.selection {
        case .food: return try Food.fetchCount(db)
        case .beverage: return try Beverage.fetchCount(db)
    }
}
```

The result of the `start` method is now a DatabaseCancellable which allows you to explicitly stop an observation:

```swift
// BEFORE: GRDB 4
let observer: TransactionObserver
observer = observation.start(...)

// NEW: GRDB 5
let cancellable: DatabaseCancellable
cancellable = observation.start(...)
```

The `onError` handler of the `start` method is now mandatory:

```swift
// BEFORE: GRDB 4
do {
    try observation.start(in: dbQueue) { value in
        print("fresh value: \(value)")
    }
} catch { ... }

// NEW: GRDB 5
observation.start(
    in: dbQueue,
    onError: { error in ... },
    onChange: { value in print("fresh value: \(value)") })
```

Convenience methods that build observations have been removed:

```swift
// BEFORE: GRDB 4
let observation = request.observationForCount()
let observation = request.observationForFirst()
let observation = request.observationForAll()
let observation = ValueObservation.tracking(request, fetch: { db in ... })

// NEW: GRDB 5
let observation = ValueObservation.tracking(request.fetchCount)
let observation = ValueObservation.tracking(request.fetchOne)
let observation = ValueObservation.tracking(request.fetchAll)
let observation = ValueObservation.tracking { db in ... }
```

<details>
    <summary>RxGRDB impact</summary>

```swift
// BEFORE: GRDB 4
request.rx.observeCount(in: dbQueue)
request.rx.observeFirst(in: dbQueue)
request.rx.observeAll(in: dbQueue)

// NEW: GRDB 5
ValueObservation.tracking(request.fetchCount).rx.observe(in: dbQueue)
ValueObservation.tracking(request.fetchOne).rx.observe(in: dbQueue)
ValueObservation.tracking(request.fetchAll).rx.observe(in: dbQueue)
```

</details>

**The behavior of ValueObservation has changed**.

Those changes have been applied identically to [GRDBCombine] and [RxGRDB], so that you are granted with an identical behavior, regardless of the technique you use to observe the database (vanilla GRDB, Combine, or RxSwift).

1. ValueObservation used to notify its initial value *immediately* when the observation starts. Now, it notifies fresh values on the main thread, *asynchronously*, by default.
    
    This means that parts of your application that rely on this immediate value to, say, setup their user interface, have to be modified. Insert a `scheduling: .immediate` argument in the `start` method:
    
    ```swift
    let observation = ValueObservation.tracking(Player.fetchAll)
    let cancellable = observation.start(
        in: dbQueue,
        // Opt in for immediate notification of the initial value
        scheduling: .immediate,
        onError: { error in ... },
        onChange: { [weak self] (players: [Player]) in
            guard let self = self else { return }
            self.updateView(players)
        })
    // <- Here the view has already been updated.
    ```
    
    <details>
        <summary>GRDBCombine impact</summary>
    
    ```swift
    let observation = ValueObservation.tracking(Player.fetchAll)
    let cancellable = observation
        .publisher(in: dbQueue)
        // Opt in for immediate notification of the initial value
        .scheduling(.immediate)
        .sink(...)
    ```
    
    </details>
    
    <details>
        <summary>RxGRDB impact</summary>
    
    ```swift
    let observation = ValueObservation.tracking(Player.fetchAll)
    let disposable = observation
        .rx.observe(in: dbQueue)
        // Opt in for immediate notification of the initial value
        .scheduling(.immediate)
        .subscribe(...)
    ```
    
    </details>

2. ValueObservation used to notify one fresh value for each and every database transaction that had an impact on the tracked value. Now, it may coalesce notifications. It your application relies on exactly one notification per transaction, use [DatabaseRegionObservation] instead.

3. Some value observations used to automatically remove duplicate values. This is no longer automatic. If your application relies on distinct consecutive values, use the [removeDuplicates] operator.

**ValueObservation features have been removed**.

1. ValueObservation used to have a `compactMap` method. This method has been removed without any replacement.
    
    If your application uses GRDBCombine or RxGRDB, then use the `compactMap` method from Combine or RxSwift instead.

2. ValueObservation used to have a `combine` method. This method has been removed without any replacement.
    
    In your application, replace combined observations with a single observation:
    
    ```swift
    // BEFORE: GRDB 4
    let playerCountObservation = ValueObservation.tracking(Player.fetchCount)
    let bestPlayersObservation = ValueObservation.tracking(Player
            .limit(10)
            .order(Column("score").desc)
            .fetchAll)
    let observation = ValueObservation
        .combine(playerCountObservation, bestPlayersObservation)
        .map(HallOfFame.init)
    
    // NEW: GRDB 5
    let observation = ValueObservation.tracking { db -> HallOfFame in
        let playerCount = try Player.fetchCount(db)
        let bestPlayers = try Player
            .limit(10)
            .order(Column("score").desc)
            .fetchAll(db)
        return HallOfFame(playerCount: playerCount, bestPlayers: bestPlayers)
    }
    ```
    
    As is previous versions of GRDB, do not use the `combineLatest` operators of Combine or RxSwift in order to combine several ValueObservation. You would lose all guarantees of [data consistency](https://en.wikipedia.org/wiki/Consistency_(database_systems)).

3. ValueObservation used to let application define custom "reducers" using the ValueReducer protocol. These apis are no longer available. See the [#731](https://github.com/groue/GRDB.swift/pull/731) conversation for a solution towards a replacement.


## Other Changes

1. The `QueryInterfaceRequest` type has been renamed to `Request`.

2. [Batch updates] used to rely of the `<-` operator. This operator has been removed. Use the `set(to:)` method instead:
    
    ```swift
    // BEFORE: GRDB 4
    try Player.updateAll(db, Column("score") <- 0)
     
    // NEW: GRDB 5
    try Player.updateAll(db, Column("score").set(to: 0))
    ```

3. [Custom SQL functions] are now [callable values](https://github.com/apple/swift-evolution/blob/master/proposals/0253-callable.md):
    
    ```swift
    // BEFORE: GRDB 4
    Player.select(myFunction.call(Column("name")))
     
    // NEW: GRDB 5
    Player.select(myFunction(Column("name")))
    ```

4. If you happen to implement custom fetch requests with the `FetchRequest` protocol, you now have to define the `makePreparedRequest(_:forSingleResult:)` method:
    
    ```swift
    // BEFORE: GRDB 4
    struct MyRequest: FetchRequest {
        func prepare(_ db: Database, forSingleResult singleResult: Bool) throws -> (SelectStatement, RowAdapter?) {
            let statement: SelectStatement = ...
            let adapter: RowAdapter? = ...
            return (statement, adapter)
        }
    }
     
    // NEW: GRDB 5
    struct MyRequest: FetchRequest {
        func makePreparedRequest(_ db: Database, forSingleResult singleResult: Bool) throws -> PreparedRequest
            let statement: SelectStatement = ...
            let adapter: RowAdapter? = ...
            return PreparedRequest(statement: statement, adapter: adapter)
        }
    }
    ```

5. The technique for using GRDB with a custom SQLite build has [changed](CustomSQLiteBuilds.md).
    
    You will have to rename a few files, and import GRDB instead of GRDBCustomSQLite:
    
    ```swift
    // BEFORE: GRDB 4
    import GRDBCustomSQLite
     
    // NEW: GRDB 5
    import GRDB
    ```


[ValueObservation]: ../README.md#valueobservation
[DatabaseRegionObservation]: ../README.md#databaseregionobservation
[RxGRDB]: http://github.com/RxSwiftCommunity/RxGRDB
[GRDBCombine]: http://github.com/groue/GRDBCombine
[Observing a Varying Database Region]: ../README.md#observing-a-varying-database-region
[removeDuplicates]: ../README.md#valueobservationremoveduplicates
[Custom SQL functions]: ../README.md#custom-sql-functions
[Batch updates]: ../README.md#update-requests