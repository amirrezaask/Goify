# Goify


## App struct
Always avoid using global state for any thing. use a type like server or app to hold your global state, connections and every thing you are going too need in requests, and always try to use abstract implementation(interface) instead of concrete types, If you need something in only for example 2 of your handlers, don't put them in App struct, and initialize them in controllers or create a new struct for those 2 handlers, don't make a mess in your main app struct.
```go
type App struct {
    db someDbInterface
    logger someLogger
    emailer someEmailer
    notificationService someNotificationService
}
```
Remember some times your app struct becomes too fat, you can break it into simpler and smaller structs
```go
type PeopleServer struct {
    db peopleDbInterface
    logger somelogger
}
type OrderServer struct {
    db orderDbInterface
    logger somelogger
    bankService bankInterface
}
```
## Handlers are better when they are closures
in Go http handlers (in stdlib, but third party libs are not that different) should be of a specific type, forexample: func(w http.ResponseWriter, req \*http.Request), so with this function signature you can't pass any external dependencies to it like database connections or logger or any thing else, on way to do such thing is to wrap this handler in a parent function which accepts what ever you want.

```go
func requestHandler(someDB *sql.DB) func(w http.ResponseWriter, r *http.Request) {
    return func(w http.ResponseWriter, r *http.Request) {
        // do handler logic
    }
}

```
benefit of this strategy in comparison with above solution (App struct) is that app struct of course handles all kind of dependency but has the overhead of having all deps in a handler whether it's going to need it or not, and probably in some edge cases like stateful application it may cause data race, but this pattern only copies the deps we actually going too need in handler because we said so, it's more explicit, and it's more concurrency friendly.

## Middlewares
There are libraries that implement middlewares, or even routers like [gorrila/mux](http://github.com/gorrila/mux) that support middlewares outside the box, but I like to look at middlewares even simpler, middlewares are just functions that run some logic before/after request being handled by router so let's do exactly this in our code.
```go
func (b *BookServer) IsAdmin(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        //query for user and check permission
        if !userIsAdmin {
            w.WriteHeader(401)
            fmt.Fprintln(w, "Access Denied")
        }
        next(w,r)
    }
}
```

## Context is a great data container
contexts are a great a way of having propagation of goroutines but they have another great ability, they can hold data in them as well.
imagin you have a middleware that checks for user identity, so it will query database for user, and you are going too need that data later, you can simply change request and put it in request context as below:
```go
func (b *BookServer) CheckUserIdentity(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        //get user identity from db
        // changing request context to a new context with user identity
        r = r.WithContext(context.WithValue(r.Context(), "ident", /*user identity*/))
        next(w, r)
    }
}
```

## DRY is good but not always
creating abstraction over a repititive code that you write most of the times is good but in my experience some times it's easier to have a code that is copied few times, to have a abstraction that is either complex and unreadable or has overhead.

## Always stick to left
When you are reading code, it's nice to see all logic sequentialy following each other, program flow should not be indented into conditions, unless you are handling an edge case or error scenario.
```go
// hard to follow
func someErrorProneFunction(x, y int) (int,error) {
    r, err := redis()
    if err == nil {
        err = r.Set(x, y)
        if err == nil {
            z, err := r.Get(x)
            if err == nil {
                return z, nil
            }
            return 0, err
        }
        return err
    }
    return err
}
// more simple version
func someErrorProneFunction(x, y int) (int, error) {
    r, err := redis()
    if err != nil {
      return 0, err  
    }
    err = r.Set(x, y)
    if err != nil {
        return 0, err
    }
    z, err := r.Get(x)
    if err != nil {
        return 0, err
    }
    return z, nil

}


```

## Be liberal in what you accept, and be conservative in what you return
when writing a function always accept parameters that are generic and abstract like interfaces, because they are so much easier to mock
and always return concrete types again because they are much simpler to assert in tests.
```go
func WriteToFile(f *os.File) {} // Wrong: when testing this function creating a os.File is too expensive

func WriteToFile(w io.Writer) {} // Good: when testing this function it's really easy to create io.Pipe and pass pipeWriter to this function and assert on reader

func WriteToFile(rwc io.ReadWriteCloser) {} // Not Good: we don't know exactly what this function is going todo, is it going to close writer? is it going to read ? we don't know.

```

another thing about passing an interface is that an interface some how shows us what this functions is going to do, forexample in above when we are passing a file we really don't know what this function is going to do with this file, maybe it's going to read content, maybe write something, maybe append something we don't know, but in second function we exactly know that this function is going to write on the interface we are passing to it.

```go
func NewRecorder() io.Writer // not good because we don't know what is the exact type behind io.Writer, this could cause problem
func NewRecorder() *os.File // good we know exactly that this function is opening/creating a file

```

## Initialize only when you need to
golang has multiple ways of defining and initializing variables, like `:=` and `var`, don't always use them interchangably, if you want to only define a variable but you want to intialize it later, use `var` and if you want to define and initialize use `:=`, don't use `:=` all the time.
```go
//not good
model := Model{}

err := json.Unmarshall(data, &model)
if err != nil {
    return fmt.Errorf("error in unmarshalling json: %v", err)
}

//good
var model Model

err := json.Unmarshall(data, &model)
if err != nil {
    return fmt.Errorf("error in unmarshalling json: %v", err)
}

``` 
like above example when you want to forexample pass pointer of a variable to be the destination of a data, just define it and not initialize it


## const when possible, var when necessary
constatns are one of the few ways we can approach immutability in golang, they are super fast so use them in every place you can. When you define a constant you are helping go-compiler a lot, with a constant compiler knows that this value is not going to change through the entier program so it can do so many optimizations around it, another thing is when you are working in concurrent programs it's so good to avoid sharing some data that can be manipulated by multiple goroutines. 

## expected errors should be expected
if you are writing a package, let's say a bank api interface for golang, always for expected error types like connection problems or domain specific errors create error types so that user of that 
package be able to assert different error types. another thing is to always define your errors as consts and not vars because they are exported and it's really easy to mess up with them so make them const and immutable.

## table driven tests
it's good practice to find all parameters that can change a function behaviour and create a type holding all those parameters and then test the function with different values for those params.
```go
func TestSum(t *testing.T) {
    tt:=map[string]struct{
        x,y,exp int
    }{
        "simple math 2+3=5": {
            2,3,5
        },
        "simple math 2+0=2": {
            2,2,2
        }
    }
    for sc,tc:=range tt{
        res:=Sum(tc.x,tc.y)
        if res != tc.exp{
            t.Errorf("Scenario %s faild, expected:%v got:%v", sc,tc.exp, res)
        }
    }

}

```
another good practice is to name your test scenarios, it'll help you a lot when debugging.

## don't name vars after what they are, name them after what they do
Don't have a variable named forexample usersMap, don't put type of variables in their name, good name for a variable shows what that variable is used for. forexample:
```go
package user
var usersMap map[int]*User // not good, if we look at the variable name we cannot understand what it is doing

var usersRepository map[int]*User // better but we are defining this variable in user package so we already know that this is users repository

var Repository map[int]*User // probably best soloution

```

## Smaller the interface, better
make your interfaces as small as possible, interfaces meant to be a way to focus on similarities of types in functionality, when you create a fat interface, first of all it's hard to implement it, and then it complicates the code that is using that interface because from outside it's not clear that forexample a function uses which one of methods in that interface

```go

//not good
type Animal interface {
    Breath()
    Eat()
    Die()
    Sleep()
    Go()
    Stay()
} // too fat interface
func AnimalManager(a Animal) {} // we don't know exactly what this functional is going to do with the animal

//good
type Eater interface {
    Eat()
}
func AnimalManager(e Eater) {} // now we exactly know that this function has only one option of calling the Eat() method, we can mock/Patch method with out even looking at function body

```


## Don't tell exactly what you want, describe it

when writing a pkg or an exposed function instead of asking for a concrete type, ask for an abstraction that describes what you are going to do with that value.
```go
type CacheConn struct {
    // some unnecessary detail
}
func UpdateCache(c *CacheConn) {} // in this function outside world needs to create a cache connection and then pass it to us, and in case of unit tests we need some cache connection only for test and that is not good


type Cache interface {
    Update(key string, data interface{}) error
}

func UpdateCache() {} // better, now for unit test we can easily create a stub/mock of this interface. 

```
remember when writing functions that accepts only interfaces, you should make your interfaces as small as possible

## Reflect is Evil, but a necessary one
for go developers, reflect package is like sun to vampires. but at least in my experience some times reflect package is the best way to implement a generic solution. reflect package has overhead, maybe some times it increases the complexity but as I said it's the necassary evil, an evil package that golang programs need to survive. stdlib uses reflect too, forexample json marshal/unmarshal uses reflect package to read struct tags, so it's not some thing that you never should touch, but it's some thing you need to be carefull when using, because of overhead and some times complexity.



