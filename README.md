
# About

Comprehensive list of major components (syntax and semantics) of several programming languages.

All components have examples or annotations to demonstrate how they can be utilized.

Tip: Use `ctrl + F` to look for a specific keyword.

# Table of Contents

[Swift](#swift-table), [Golang](#go-table), [Java](#java-table)

<h4 id="swift-table"></h4>

* **[Swift](#swift)**
    <!-- * Semantics / Language Characteristics -->
    * [Concurrency](#swift-concurrency)
        * [Structured Concurrency & AsyncSequence](#swift-structured-concurrency)
        * [Actor](#swift-actor)
        * [Detached Task](#swift-detached-task)
        <!-- * (non-sendable, @sendable) -->
    * [Generics](#swift-generics)
        * [Type Constraints](#swift-type-constraints)
        * [where](#swift-generic-where)
        * [Associated Type & Protocol](#swift-generic-associated-type-protocol)
        * [some & any. Opaque & Boxed Types](#swift-generic-some-any)
    * [Protocols](#swift-protocols)
        * [Syntax](#swift-protocols-syntax)
        * [Conformance](#swift-conformance)
    * [Extensions](#swift-extensions)
    * [Closures](#swift-closures)
        * [Trailing closure](#swift-closure-trailing)
        * [@escaping](#swift-closure-escaping)
        * [Capturing value](#swift-closure-capture-value)
        * [weak unowned for ARC](#swift-closure-weak-unowned-ARC)
    * [Functional Programming](#swift-functional-programming)
    * [Optional](#swift-optional)
    * [Control Flow](#swift-control-flow)
        * [guard](#swift-guard)
        * [if](#swift-if)
        * [for](#swift-for)
        * [while](#swift-while)
        * [switch](#swift-switch)
    <!-- * [Collections](#swift-collections)  -->
    <!-- * [Functions](#swift-functions)-->
    * [Error](#swift-error)
    * Macros
    * Property Wrappers

<h4 id="go-table"></h4>

* **[Go](#go)**
    * [Concurrency](#go-concurrency)
        * [Channel & goroutine](#go-channel-goroutine)
        * [select](#go-select)
        * [Mutex, Atomic & Waitgroup](#go-mutex-atomic-waitgroup)
        <!-- * Stateful goroutine -->
    * [Generics](#go-generics)
    * [Control Flow](#go-control-flow)
        * [if](#go-if)
        * [for](#go-for)
        * [switch](#go-switch) 
        * [defer](#go-defer)
    <!-- * Type. Struct & Interface -->
    * [Pointers](#go-pointers)
    * [Error](#go-error)
<!-- * [Collections](#go-collections)  -->

<h4 id="java-table"></h4>

* **[Java](#java)**
    * [Concurrency](#java-concurrency)
        * [Runnable & Thread](#java-runnable-thread)
        * [ExecutorService & Thread pool](#java-executorService)
        * [CompletableFuture](#java-completableFuture)
    * [Generics](#java-generics)
    * [Control Flow](#java-control-flow)
        * [if](#java-if)
        * [for](#java-for)
        * [while](#java-while)
        * [switch](#java-switch)
    <!-- * [Collections](#java-collections) -->
    * [Classes](#java-classes)
        * [static, getter, setter](#java-static-getter-setter)
        * [Inheritance](#java-inheritance)
        * [Abstract](#java-abstract)
        * [Lambda](#java-lambda)
        * [Enum](#java-enum)
    * [Interface](#java-interface)
    * [Error](#java-error)
<!-- * [Python](#python)
    <!-- * [Collections](#python-collections)
    * [Error](#python-error) -->


## Swift

<h3 id="swift-concurrency">Concurrency</h3>

<h4 id="swift-structured-concurrency">Structured Concurrency & AsyncSequence</h4>

* Two ways to create child tasks. `async let` or `TaskGroup`
* Child tasks inherit parent task's priority & get cancellation propagate.
* When canceled, parent task propagates cancellation to child tasks.

```swift
Task {
    await staticChildTasks()
    let imgs = await dynamicChildTasks(numOfTasks: 10)
    print(imgs.count)  // 10
}

func staticChildTasks() async -> [Image] {
    async let img1 = imgDownload(id: 1)
    async let img2 = imgDownload(id: 2)
    async let img3 = imgDownload(id: 3)
    
    return await [img1, img2, img3]
}

func dynamicChildTasks(numOfTasks: Int) async -> [Image] {
    await withTaskGroup(of: Image.self) { group in
        var images: [Image] = []
        
        for i in 1...numOfTasks {
            group.addTask { await imgDownload(id: i) }
        }
        
        for await image in group {      // AsyncSequence
            images.append(image)
        }
        
        return images
    }
}

func imgDownload(id: Int) async -> Image {
    let randomDuration = UInt64(Int.random(in: 1...3) * 1_000_000_000)
    try? await Task.sleep(nanoseconds: randomDuration)
    return Image()
}

class Image {
}
```

<h4 id="swift-actor">Actor</h4>

* Similar to classes, but allow only one task to access mutable state at a time to prevent data corruption from data race.
* Need `await` to access mutable data outside of actor.

```swift
actor Counter {
    let threshold = 100
    private(set) var count = 0
    
    func addCount() {
        count += 1
    }
    
    private func doSomething() {
        addCount()          // Within the actor, don't need await.
    }
    
    nonisolated func printThreshold() {
        print(threshold)    // As constant can't be mutated, can be nonisolated.
    }
}

let myCounter = Counter()
print(myCounter.threshold)

Task {
    await myCounter.addCount()
    await print(myCounter.count)
}
```

<h4 id="swift-detached-task">Detached Task</h4>

Totally independent from parent task. Does not inherit anything from parent task.

```swift
let outerTask = Task {
    await largeDownload()

    Task.detached(priority: .background) {
        await backgroundCleanUp()
    }
}

// The outer task will cancel, but the inner detached task will not cancel.
// Task.isCancelled is False inside backgroundCleanUp.
outerTask.cancel()
```

[Back to top](#table-of-contents)



<h3 id="swift-generics">Generics</h3>

<h4 id="swift-type-constraints">Type Constraints</h4>

```swift
struct MyStack<Element: Equatable> {
    var items: [Element] = []

    mutating func push(_ item: Element) {
        items.append(item)
    }
}
```

<h4 id="swift-generic-where">where</h4>

```swift
extension MyStack {
    // Add func for a specific concrete type
    func sum() -> Int where Element == Int {
        items.reduce(0, +)
    }

    // Add func only when Element type is comparable
    func isTopLargerThan(_ item: Element) -> Bool where Element: Comparable {
        items.last! > item
    }
}

let intStack = MyStack<Int>(items: [1, 2, 3, 4])
let doubleStack = MyStack<Double>(items: [1.0, 2.0, 3.0, 4.0])

intStack.sum()
// doubleStack.sum()    Compile error as .sum() not available for Double
intStack.isTopLargerThan(5)
doubleStack.isTopLargerThan(5.0)
```



<h4 id="swift-generic-associated-type-protocol">Associated Type & Protocol</h4>

```swift
protocol Box {
    associatedtype SomeType: Equatable
    mutating func push(_ item: SomeType)
}

extension MyStack: Box {
}
```

<h4 id="swift-generic-some-any">some & any. Opaque & Boxed Types</h4>

```swift
func useKeywordSome(_ boxes: [some Box]) -> [some Box] {
    //some indicates the exact type known to the compiler
    return boxes
}

func useKeywordAny(_ boxes: [any Box]) -> [any Box] {
    //any indicates the exact type NOT known to the compiler
    return boxes
}

let intStack = MyStack(items: [1, 2, 3])
let doubleStack = MyStack(items: [1.0, 2.0, 3.0])

useKeywordSome([intStack, intStack])
// Compile error because the types of stack are not the same
// useKeywordSome([intStack, doubleStack])

useKeywordAny([intStack, doubleStack])      // Works for different types of stack
```
[Back to top](#table-of-contents)



<h3 id="swift-protocols">Protocols</h3>

<h4 id="swift-protocols-syntax">Syntax</h4>

```swift
protocol ParentI {
}

protocol ChildI: ParentI {
}

protocol ClassOnlyI: AnyObject {
    // Can't use on Structs & Enums
}

// MARK: - Init

protocol ProtocolWithInit {
    init(text: String)
}

class MyClass: ProtocolWithInit {
    required init(text: String) {
        // Need keyword required
    }
}

struct MyStruct: ProtocolWithInit {
    init(text: String) {
        // Don't need keyword required
    }
}

// MARK: - Mutating func

protocol HasMutatingFunc {
    // Conforming value type needs to use the keyword mutating
    // Conforming class does not need to use the keyword mutating
    mutating func changeValue()
}

class MyClass: HasMutatingFunc {
    var num = 0
    
    func changeValue() {
        num += 1    // Don't need keyword mutating
    }
}

struct MyStruct: HasMutatingFunc {
    var num = 0

    mutating func changeValue() {
        num += 1    // Need keyword mutating
    }
}

// MARK: - Checking

let myObjects: [AnyObject] = []

for obj in myObjects {
    if let hashedObj = obj as? any Hashable {
        print(hashedObj.hashValue)
    }

    switch obj {
        case is any Hashable:
            print("Hashable")
        default:
            print("Not hashable")
    }
}

// MARK: - Optional requirements

// To work with Objective-C classes
@objc protocol BackwardCompatible {
    @objc optional var optionalGetter: Int { get }
}
```

<h4 id="swift-conformance">Conformance</h4>

```swift
protocol FirstI {
}
protocol SecondI {
}

class ParentC {
}
class ChildC: Parent, FirstI, SecondI {
}

protocol MyProtocol {
    static var clsConstant: Int { get }
    static var clsVariable: Int { get set }
    
    var mustBeSettable: Int { get set }
    var doesNotNeedToBeSettable: Int { get }
}

class BasicImplementation: MyProtocol {
    static let clsConstant = 99
    static var clsVariable = 0

    var mustBeSettable = 0
    var doesNotNeedToBeSettable: Int { 99 }
}

// MARK: - Default implementation

protocol HasDefaultImplementation { 
    func saySomething()
}

extension HasDefaultImplementation {
    func saySomething() {   // Provide default implementation
        print("Default")
    }
}

struct PrintDefault: HasDefaultImplementation {
}

struct PrintCustom: HasDefaultImplementation {
    func saySomething() {
        print("Custom")
    }
}

PrintDefault().saySomething()   // "Default"
PrintCustom().saySomething()    // "Custom"

// MARK: - Synthesized

struct Point: Hashable {
    var x = 0
    var y = 0
}

// Possible because Point automatically conforms to Equtable
// Gets synthesized implementation of hash(into:) & ==
let points: Set<Point> = []
let isEqual = Point(x: 1, y: 1) == Point(x: 1, y: 1)
```
[Back to top](#table-of-contents)



<h3 id="swift-extensions">Extensions</h3>

* Objective-C category

```swift
extension Int {
    enum NumType {
        case even, odd
    }

    var numType: NumType {
        self % 2 == 0 ? .even : .odd
    }
    
    var squared: Int {
        self * self
    }
    
    mutating func increaseBy(_ num: Int) {
        self += num
    }
}

extension Array where Element == Int {
    func squared() -> [Int] {
        map { $0.squared }
    }
}

let nums = [1, 2, 3, 4]
nums.squared()            // [1, 4, 9, 16] returned
```
[Back to top](#table-of-contents)



<h3 id="swift-closures">Closures</h3>

<h4 id="swift-closure-trailing">Trailing closure</h4>

See [Functional Programming](#swift-functional-programming)

<h4 id="swift-closure-escaping">@escaping</h4>

```swift
var savedClosure: (() -> Void)!
func storeClosure(closure calledAfterReturn: @escaping () -> ()) {
    savedClosure = calledAfterReturn
}

storeClosure {
    print("Hello")
}
savedClosure()      // Hello
```

<h4 id="swift-closure-capture-value">Capturing value</h4>

```swift
func performOperation(op: @escaping (Int) -> Int) -> () -> Int {
    var curVal = 1

    func calcNextValue() -> Int {
        curVal = op(curVal)
        return curVal
    }
    return calcNextValue
}


let increment = performOperation {
    $0 + 1
}

let multiplyByTen = performOperation {
    $0 * 10
}

print(increment())  // 2
print(increment())  // 3
print(increment())  // 4

print(multiplyByTen())  // 10
print(multiplyByTen())  // 100
print(multiplyByTen())  // 1000
```

<h4 id="swift-closure-weak-unowned-ARC">weak unowned for ARC</h4>

```swift
class MyClass {
    let num = 1

    // The closures have type of () -> ()
    lazy var unownedClosure = { [unowned self] in
        print(self.num)
    }

    lazy var weakClosure = { [weak self] in
        print(self?.num as Any)
    }

    lazy var createMemoryLeak = {
        print(self.num)
    }

    deinit {
        print("Deinit")
    }
}

var refCycleTest: MyClass?
refCycleTest = MyClass()
refCycleTest?.unownedClosure()
refCycleTest = nil              // Print Deinit

refCycleTest = MyClass()
refCycleTest?.weakClosure()
refCycleTest = nil              // Print Deinit

refCycleTest = MyClass()
refCycleTest?.createMemoryLeak()
refCycleTest = nil              // DOES NOT Print Deinit
```
[Back to top](#table-of-contents)



<h3 id="swift-functional-programming">Functional Programming</h3>

```swift
let total = [1, 2, 3].reduce(0, +)
// 6

let evenNums = (0...10).filter { $0 % 2 == 0 }
// [0, 2, 4, 6, 8, 10]

let squared = [2, 3, 4].map { $0 * $0 }
// [4, 9, 16]

let noOptional = [1, nil, 3].compactMap { $0 }
// [1, 3]

let twoD = [[1, 3], [2, 4]]
let oneD = twoD.flatMap { $0 }
// [1, 3, 2, 4]

let printAll = [1, 2, 3].forEach { print($0) }
```
[Back to top](#table-of-contents)



<h3 id="swift-optional">Optional</h3>

```swift
// Optional chaining & Nil coalescing
let defaultValue = 0 // Use defaultValue if left is nil
let intTypeData = optionalData?.optionalProperty?.anotherOptional ?? defaultValue

let forceUnwrapText: String! = "Implicitly unwrapped optional"
let crashIfNil: String = forceUnwrapText
```
[Back to top](#table-of-contents)



<h3 id="swift-control-flow">Control Flow</h3>

```swift
let condition = true
let optionalData: Int? = 1
```

<h4 id="swift-guard">guard</h4>

```swift
func useGuardForEarlyExit(num: Int?) -> Int {
    guard let num else { return -1 }
    return num
}
```

<h4 id="swift-if">if</h4>

```swift
let five = condition ? 5 : -1  // Ternary operator
let sameAsAbove = if condition { 5 } else { -1 }

if let optionalData {

}

if let optionalData = optionalData { // Same as above

}

if condition {

} else if !condition {

} else {

}
```

<h4 id="swift-for">for</h4>

```swift
for _ in 0...9 {
    // Loop 10 times
}

for i in [1, 2, 3] {
    print(i) // 1, 2, 3
}

for i in [1, 2, 3] where i > 1 {
    print(i) // 2, 3
}

for i in stride(from: 1, to: 10, by: 2) {
    print(i) // 1, 3, 5, 7, 9
}

for (index, i) in [1, 2, 3].enumerated() {
    print((index, i)) // (0, 1), (1, 2), (2, 3)
}
```

<h4 id="swift-while">while</h4>

```swift
while condition {
    break
}

repeat {
    break
} while condition
```

<h4 id="swift-switch">switch</h4>

```swift
// No need for break.
// Add fallthrough for multiple cases to be checked
let one = 1
switch one {
    case 1...:
        print("positive")
    case ...(-1):
        print("negative")
    default:
        print("zero")
}

enum Numbers {
    case positive, negative, zero
}

let num: Numbers = .zero
switch num {
    case .positive:
        print("positive")
    case .negative:
        print("negative")
    case .zero:     // Exhaustive so dont need default.
        print("zero")
}

let boxDimensions = (10, 2, 1)
switch boxDimensions {
    case let (w, h, d) where [w, h, d].contains { $0 < 0 }:
        print("Dimensinon can't be negative")
    case let (_, h, _) where h > 10:
        print("Box too tall")
    case let (w, h, d) where w * h * d > 100:
        print("Box too large")
    default:
        print("Good box size")
}

let optionalVal: Int? = 5
switch optionalVal {
    case let unwrappedVal?:
        print(unwrappedVal)
    case .none:
        print("404")
}
```

[Back to top](#table-of-contents)



<!-- <h3 id="swift-collections">Collections</h3>

```swift
// Dictionary
// Array
// Set
```
[Back to top](#table-of-contents)





<h3 id="swift-functions">Functions</h3>

```swift

```
[Back to top](#table-of-contents) -->



<h3 id="swift-error">Error</h3>

```swift
enum MyError: Error {
    case divisionByZero
}

let nilIfError = try? divideTen(by: 0)

do {
    let _ = try divideTen(by: 0)
} catch MyError.divisionByZero {
    print("Can't divide by 0")
} catch {
    print("Unknown error")
}

func divideTen(by num: Int) throws -> Int {
    guard num != 0 else { throw MyError.divisionByZero }
    return 10 / num
}
```
[Back to top](#table-of-contents)



## Go

<h3 id="go-concurrency">Concurrency</h3>

<h4 id="go-channel-goroutine">Channel & goroutine</h4>

```go
func sendNum(n int, ch chan int) {
	ch <- n
}

ch := make(chan int)
go sendNum(1, ch)
go sendNum(2, ch)

val1, val2 := <-ch, <-ch	// Wait for 2 values

bufferedCh := make(chan int, 2)
bufferedCh <- 1
bufferedCh <- 2     // Send upto 2 values without waiting

val1, val2 = <-bufferedCh, <-bufferedCh

c := make(chan int)
go sendNumsAndClose(c)
for i := range c {      // Receive until channel closes
    fmt.Println(i)
}

func sendNumsAndClose(c chan int) {
	for i := 0; i < 10; i++ {
		c <- i
	}
	close(c)
}

func sendOnlyChannel(ch chan<- int) {
	ch <- 1
    // <- ch			Can't receive
}

func receiveOnlyChannel(ch <-chan int) {
	// ch <- 1			Can't send
	fmt.Println(<- ch)
}
```

<h4 id="go-select">select</h4>

```go
var(
	ch = make(chan int)
	done = make(chan bool)
)

func main() {
	go func() {
		ch <- 1
    	done <- true
	}()

	waitForMsgs()
}

func waitForMsgs() {
    for i:= 0; i < 2; i++ {
        select {
        case receivedNum := <-ch:
            fmt.Println(receivedNum)
        case <-done:
            fmt.Println("DONE")
        // default:    // To not wait for anything
        //     fmt.Println("Didn't wait")
        }
    }
}
```

<h4 id="go-mutex-atomic-waitgroup">Mutex, Atomic & Waitgroup</h4>

```go
var atomicCounter int32
var mutexCounter int
var counterMutex sync.Mutex

func increment() {
    counterMutex.Lock()
    defer counterMutex.Unlock()
    mutexCounter++
}

func main() {
    var wg sync.WaitGroup
    for i := 0; i < 10000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
			atomic.AddInt32(&atomicCounter, 1)
            increment()
        }()
    }
    wg.Wait()

    // Both values 10000
    fmt.Printf("Final values: %d, %d", atomiatomic.LoadInt32(&atomicCounter)cCounter, mutexCounter)
}
```
[Back to top](#table-of-contents)



<h3 id="go-generics">Generics</h3>

```go
type Box[Number Addable] struct {
	value Number
}

func (b Box[T]) useBox() {
	fmt.Printf("Using box: %T\n", b)
}

intBox := Box[int]{}
intBox.useBox()      // Using box: main.Box[int]


type Addable interface {
    float64 | int
}

func addIntOrFloat64[Number Addable] (a, b Number) Number {
    return a+b
}

fmt.Println(addNumbers(1, 2))   // 3
```
[Back to top](#table-of-contents)



<!-- <h3 id="go-collections">Collections</h3>

```go
// Dictionary, Array, Set
```
[Back to top](#table-of-contents) -->



<h3 id="go-control-flow">Control flow</h3>

<h4 id="go-if">if</h4>

```go
if num := rand.IntN(10); num > 5 {
    fmt.Println("Large val %d", num)
} else {
    fmt.Println("Small val %d", num)    // Can use num in else
}
```

<h4 id="go-for">for</h4>

```go
for i := 0; i < 10; i++ {
    // Loop 10 times
}

for i < 10 {    // While loop version doing the same thing as above
	i += 1
}

for {           // Infinite loop
    break       // To break out of the loop
}
```

<h4 id="go-switch">switch</h4>

```go
i := 1

switch i {
case 1:
    fmt.Println("One")
default:
    fmt.Println("Will not reach default. Dont need break")
}
```

<h4 id="go-defer">defer</h4>

```go
func deferFunctionCall() {
	defer fmt.Println("Last - LIFO")
	defer fmt.Println("Second - CleanUp work")
	fmt.Println("First")
}
```
[Back to top](#table-of-contents)



<h3 id="go-functions">Functions</h3>

```go
func multipleReturnVals(in1, in2 int) (out1, out2 int) {
	out1 += in1 // out1 default val = 0
	out2 += in2
	return
}
```
[Back to top](#table-of-contents)



<h3 id="go-pointers">Pointers</h3>

```go
func swap(x *int, y *int) {
	temp := *x
	*x = *y
	*y = temp
}

x := 1
y := 2
swap(&x, &y)
fmt.Println(x, y)	// Print 2, 1
```
[Back to top](#table-of-contents)



<h3 id="go-error">Error</h3>

```go
// No exceptions thrown like other languages.
// Return error as the last parameter.
var ErrGotPositiveNum = fmt.Errorf("Already positive")

func makeItPositive(n int) (int, error) {
	if n > 0 {
		return -1, ErrGotPositiveNum
	}
	
	return n * -1, nil
}

// Print output:
// Did no work. Already positive
// Something went wrong
if num, err := makeItPositive(9); err != nil {
    if errors.Is(err, ErrGotPositiveNum) {
        fmt.Println("Did no work.", err)
    }

    fmt.Println("Something went wrong") 
} else {
    fmt.Printf("Got positive: %d \n", num)
}


type MyError struct {
	CustomMsg string
}

// Add this func to implement the error interface
func (e *MyError) Error() string {
	return "Printing: " + e.CustomMsg
}

func doSomething() error {
	return &MyError{
		CustomMsg: "This is custom error message",
	}
}

err := doSomething()

// Print output:
// Got MyError. Msg: This is custom error message
var myErr *MyError
if errors.As(err, &myErr) {
    fmt.Println("Got MyError. Msg: " + myErr.CustomMsg)
}
```
[Back to top](#table-of-contents)



## Java

<h3 id="java-concurrency">Concurrency</h3>

<h4 id="java-runnable-thread">Runnable & Thread</h4>

```java
class MyRunnable implements Runnable {
    public void run() {
        System.out.println("New thread");
    }
}

Thread myThread = new Thread(new MyRunnable());
myThread.start();
```

<h4 id="java-executorService">ExecutorService & Thread pool</h4>

```java
ExecutorService executor = Executors.newFixedThreadPool(4);
executor.submit(new MyRunnable());
executor.shutdown();
```

<h4 id="java-completableFuture">CompletableFuture</h4>

```java
CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
    // Some long-running operation
    return "Future 1";
});

CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
    return "Future 2";
});

CompletableFuture<String> future3 = CompletableFuture.supplyAsync(() -> {
    return "Future 3";
});

CompletableFuture<Void> allFutures = CompletableFuture.allOf(future1, future2, future3);

allFutures.thenRun(() -> {
    String result1 = future1.join();
    String result2 = future2.join();
    String result3 = future3.join();
    System.out.println("All completed: " + result1 + ", " + result2 + ", " + result3);
});
```
[Back to top](#table-of-contents)



<h3 id="java-generics">Generics</h3>

```java
class MyStack<Element extends Number> {
    private LinkedList<Element> list = new LinkedList<Element>();

    public void push(Element e) {
        list.add(e);
    }

    public Element pop() {
        return list.removeLast();
    }

    public int firstIntVal() {
        // Can get int because Element type upper bound is Number
        return list.getFirst().intValue();
    }
}

MyStack<Double> stack = new MyStack<>();
stack.push(3.14);
System.out.println(stack.firstIntVal());    // Print 3


public <E> void checkList(List<E> list, int capacity) {
    if (list.size() > capacity) list.clear();
}

// Wildcard for simpler equivalent of above.
// Possible as not concerned with the type inside the list
public void checkList(List<?> list, int capacity) {
    if (list.size() > capacity) list.clear();
}
```
[Back to top](#table-of-contents)



<h3 id="java-control-flow">Control Flow</h3>

<h4 id="java-if">if</h4>

```java
int num = 1;
String one = (num == 1) ? "one" : "Not one";

if (num > 0)
    System.out.println("positive");
else if (num < 0)
    System.out.println("negative");
else
    System.out.println("zero");
```

<h4 id="java-for">for</h4>

```java
for(int i = 0; i < 10; i++) {
    // Loop 10 times
}

int[] numbers = {2, 3, 4, 5};
for (int number: numbers) {
    System.out.println(number); // 2, 3, 4, 5
}
```

<h4 id="java-while">while</h4>

```java
while (true) {
    break;
}

do {
    break;
} while (true);
```

<h4 id="java-switch">switch</h4>

```java
switch (num) {
    case 1:
        System.out.println("Need break to prevent fallthrough");
        break;
    default:
        System.out.println("Default");
}
```
[Back to top](#table-of-contents)



<h3 id="java-classes">Classes</h3>

<h4 id="java-static-getter-setter">static, getter, setter</h4>

```java
class MyClass {
    static String sharedName;

    private int readOnlyData;

    static class StaticNestedClass {  // Using outer class as a namespace

    }

    class NestedClass { // Inner class tied to instance of outer class
        public void accessReadOnlyData() {
            System.out.println(MyClass.this.readOnlyData);
        }
    }

    static
    {
        sharedName = "SharedName"; // Block gets called only once.
    }

    public static void classMethod() {
        
    }

    public MyClass(int readOnlyData) {
        this.readOnlyData = readOnlyData;
    }

    public int getReadOnlyData() {
        return this.readOnlyData;
    }
}

System.out.println(MyClass.sharedName);

MyClass outerObj = new MyClass(5);
MyClass.NestedClass innerObj = outerObj.new NestedClass();
innerObj.accessReadOnlyData();
```

<h4 id="java-inheritance">Inheritance</h4>

```java
class Parent {
    public void sayHi() {
        // print Hi
    }

    public final void noOverRide() {
        // Child class can't override this method
    }
}

class Child extends Parent {

    public Child() {    // Constructor
        // super();
        // Above line is inside a constructor implicitly,
        // so parent constructor gets called by default.
    }

    @Override   // Annotation for compiler warnings.
    public void sayHi() {
        // print Hi
    }
}

final class NoChild {   // Can't be subclassed
    int count = 5; // Constant
}
```

<h4 id="java-abstract">Abstract</h4>

```java
abstract class MyAbstractClass {    // Must be subclassed
    public abstract void sayHi();
}

class MyConcretClass extends MyAbstractClass {  // Must implement all abstract methods
    public void sayHi() {
        // Print Hi
    }
}

MyAbstractClass obj = new MyConcretClass();
```

<h4 id="java-lambda">Lambda</h4>

```java
@FunctionalInterface
interface MyInterface {
    public int performCalculation(int a, int b);
}

MyInterface multiplier = (a, b) -> a * b;
int result = multiplier.performCalculation(3, 5);   // 15

List<Integer> nums = List.of(1, 2, 3);
nums.forEach( (n) -> { System.out.println(n); } );
```

<h4 id="java-enum">Enum</h4>

```java
enum Number {
    Positive(1), Negative(-1), Zero(0);

    private int val;

    private Number(int val) {
        this.val = val;
    }

    public void setVal(int val) {
        this.val = val;
    }

    public int getVal() {
        return this.val;
    }
}

System.out.println(Number.Negative.ordinal());    // Print 1
Number num = Number.Positive;
num.setVal(100);
System.out.println(num.getVal());

switch (num) {
    case Positive:
        // Print positive
        break;
    case Negative:
        // Print negative
        break;
    default:
        // Print zero
        break;
}
```
[Back to top](#table-of-contents)



<h3 id="java-interface">Interface</h3>

```java
interface A {
    void needImplementation();
}

interface B extends A {

}

class C implements B {
    public void needImplementation() {
        // Print Hi
    }
}

// Anonymous inner class
A a = new A() {
    public void needImplementation() {
        // Print Hello
    }
};
a.needImplementation(); // Print Hello instead of Hi
```
[Back to top](#table-of-contents)



<!-- <h3 id="java-collections">Collections</h3>

<h4 id="java-array">Array</h4>

```java

// String (immutable)
String immutable = "Immutable";
StringBuffer sb = new StringBuffer(str: "Mutable");

// Array
int zeroes[] = new int[4]; // Fixed size
int nums[] = {1, 2, 3, 4};

// ArrayList

// Dict
// Set

```
[Back to top](#table-of-contents) -->


<h3 id="java-error">Error</h3>

```java
class MyException extends Exception {
    public MyException(String msg) {
        super(msg);
    }
}

int num = 4;

try {
    if (num > 0)
        throw new MyException("Only want negative");
    else {
        int divideByZero = 5/num;
    }
} catch(Exception e) {
    System.out.println(e);
} finally {
    // Close resource
}

// try with resources. Auto closing resource with AutoCloseable interface.
// Handles scanner.close();
try (Scanner scanner = new Scanner(new File("test.txt"))) {
   
} catch (FileNotFoundException e) {
    // Error handling
}
```
[Back to top](#table-of-contents)



<!-- ## Python

<h3 id="python-collections">Collections</h3>

```python
# Dictionary

# Array

# Set
```
[Back to top](#table-of-contents) -->
