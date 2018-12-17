# Golang — handling errors gracefully

Although **go** has a simple error model, at first sight, things are not as easy as they should. In this article, I want to provide a good strategy to handle errors and overcome the problems that you may encounter along the way.

First, we are going to analyze what is an error in go.

Then we are going to see the flow between error creation and error handling and analyze the possible flaws.

Finally, we are going to explore a solution that allows us to overcome those flaws without compromise the design of our application.

### What is an error in go

Looking at the builtin error type we can take some conclusions:

```
// The error built-in interface type is the conventional interface for
// representing an error condition, with the nil value representing no error.
type error interface {
	Error() string
}
```

We see that an error is an interface that implements a simple method Error returning a string.

This definition tells us that all it takes to create an error is a simple string, so if I create the following struct:

```
type MyCustomError string
func (err MyCustomError) Error() string {
  return string(err)
}
```

I came up with the simplest error definition possible.

Note: This is just to give an example. We could create an error using the go standard packages *fmt* and *errors*:

```
import (
  "errors"
  "fmt"
)
simpleError := errors.New("a simple error")
simpleError2 := fmt.Errorf("an error from a %s string", "formatted")
```

Is a simple message enough to handle the errors gracefully? Let’s answer this question in the end, exploring the solution I will provide.

### Error flow

So we already know what is an error and the next step is to visualize the flow in his lifecycle.

For the sake of simplicity and don’t repeat yourself principle, is desirable to take action on an error once at a single place.

Let’s see why giving the following example:

```
// bad example of handling and returning the error at the same time
func someFunc() (Result, error) {
 result, err := repository.Find(id)
 if err != nil {
   log.Errof(err)
   return Result{}, err
 }
  return result, nil
}
```

What is the problem with this piece of code?

Well, we are handling the error twice by logging it first and then by returning it to the caller of this function.

Probably one of your team colleagues will use this function and when an error returns, he will log the error again. Then an error nightmare occurs in the system log.

So imagine that we have 3 layer levels at our application, the **repository**, the **interactor**, and the **web server**:

```
// The repository uses an external depedency orm
func getFromRepository(id int) (Result, error) {
  result := Result{ID: id}
  err := orm.entity(&result)
  if err != nil {
    return Result{}, err
  }
  return result, nil 
}
```

By the principle, I referred before, this is the right way to handle the error by returning to the top. Later it will be logged, retrieve the right feedback to the web server, all in one place.

But there is a problem with the previous piece of code. Unfortunately, go builtin error doesn’t provide a stack trace. Besides that, the error was generated at an external dependency, and we need to know what piece of code inside our project is responsible for this error.

**github.com/pkg/errors** to the rescue.

I am going to redo the previous function by adding the stack trace and by adding the message that repository failed to get the result. I want to do this without compromising the original error:

```
import "github.com/pkg/errors"
// The repository uses an external depedency orm
func getFromRepository(id int) (Result, error) {
  result := Result{ID: id}
  err := orm.entity(&result)
  if err != nil {
    return Result{}, errors.Wrapf(err, "error getting the result with id %d", id);
  }
  return result, nil 
}
// after the error wraping the result will be 
// err.Error() -> error getting the result with id 10: whatever it comes from the orm
```

What this function does, is to wrap the error coming from the ORM, build a stack trace without compromising the original error.

So let's see how the other layers will handle the error. First the interactor:

```
func getInteractor(idString string) (Result, error) {
  id, err := strconv.Atoi(idString)
  if err != nil {
    return Result{}, errors.Wrapf(err, "interactor converting id to int")
  }
  return repository.getFromRepository(id) 
}
```

Now the top layer, the web server:

```
r := mux.NewRouter()
r.HandleFunc("/result/{id}", ResultHandler)
func ResultHandler(w http.ResponseWriter, r *http.Request) {
  vars := mux.Vars(r)
  result, err := interactor.getInteractor(vars["id"])
  if err != nil { 
    handleError(w, err) 
  }
  fmt.Fprintf(w, result)
}
func handleError(w http.ResponseWriter, err error) { 
   w.WriteHeader(http.StatusIntervalServerError)
   log.Errorf(err)
   fmt.Fprintf(w, err.Error())
}
```

As you saw we just handled the error at the top layer. Perfect? No. If you notice we are always returning 500 as HTTP response code. Besides that, we are always logging the error. Some errors like “result not found” just add noise to our log.

### My Solution

We saw in the previous topic that a string is not enough to take de decisions when handling the errors on the top layer.

We know that if we introduce something new in an error, we will somehow induce a dependency at the points where the error is created and when the error is finally handled.

#### So let’s explore a solution defining 3 goals:

- Provide a good error stack trace
- Log the error (e.g. the web infrastructure layer)
- Provide a contextual error information to the user when necessary. (e.g. The provided email is not in the right format)

First, we create an error type:

```
package errors
const(
  NoType = ErrorType(iota)
  BadRequest
  NotFound 
  //add any type you want
)
type ErrorType uint
type customError struct {
  errorType ErrorType 
  originalError error 
  contextInfo map[string]string 
}
// Error returns the mssage of a customError
func (error customError) Error() string {
   return error.originalError.Error()
}
// New creates a new customError
func (type ErrorType) New(msg string) error {
   return customError{errorType: type, originalError: errors.New(msg)}
}

// New creates a new customError with formatted message
func (type ErrorType) Newf(msg string, args ...interface{}) error {    
   err := fmt.Errof(msg, args...)
   
   return customError{errorType: type, originalError: err}
}

// Wrap creates a new wrapped error
func (type ErrorType) Wrap(err error, msg string) error {
   return type.Wrapf(err, msg)
}

// Wrap creates a new wrapped error with formatted message
func (type ErrorType) Wrapf(err error, msg string, args ...interface{}) error { 
   newErr := errors.Wrapf(err, msg, args..)   
   
   return customError{errorType: errorType, originalError: newErr}
}
```

So as you may see I only make public the ErrorType and the error types. We can create new errors and wrap existing ones.

But we are missing two things.

How can we check the error type, without exporting customError?

How can we add/get a context to the errors, even to pre-existing ones from external dependencies?

Let’s adopt the strategy of **github.com/pkg/errors.** First wrap these library methods.

```
// New creates a no type error
func New(msg string) error {
   return customError{errorType: NoType, originalError: errors.New(msg)}
}

// Newf creates a no type error with formatted message
func Newf(msg string, args ...interface{}) error {
   return customError{errorType: NoType, originalError: errors.New(fmt.Sprintf(msg, args...))}
}

// Wrap wrans an error with a string
func Wrap(err error, msg string) error {
   return Wrapf(err, msg)
}

// Cause gives the original error
func Cause(err error) error {
   return errors.Cause(err)
}

// Wrapf wraps an error with format string
func Wrapf(err error, msg string, args ...interface{}) error {
   wrappedError := errors.Wrapf(err, msg, args...)
   if customErr, ok := err.(customError); ok {
      return customError{
         errorType: customErr.errorType,
         originalError: wrappedError,
         contextInfo: customErr.contextInfo,
      }
   }

   return customError{errorType: NoType, originalError: wrappedError}
}
```

Now let's build our methods do handle context and type for any generic error:

```
// AddErrorContext adds a context to an error
func AddErrorContext(err error, field, message string) error {
   context := errorContext{Field: field, Message: message}
   if customErr, ok := err.(customError); ok {
      return customError{errorType: customErr.errorType, originalError: customErr.originalError, contextInfo: context}
   }

   return customError{errorType: NoType, originalError: err, contextInfo: context}
}

// GetErrorContext returns the error context
func GetErrorContext(err error) map[string]string {
   emptyContext := errorContext{}
   if customErr, ok := err.(customError); ok || customErr.contextInfo != emptyContext  {

      return map[string]string{"field": customErr.context.Field, "message": customErr.context.Message}
   }

   return nil
}

// GetType returns the error type
func GetType(err error) ErrorType {
   if customErr, ok := err.(customError); ok {
      return customErr.errorType
   }

   return NoType
}
```

Now getting back to our example, we are going to apply this new error package:

```
import "github.com/our_user/our_project/errors"
// The repository uses an external depedency orm
func getFromRepository(id int) (Result, error) {
  result := Result{ID: id}
  err := orm.entity(&result)
  if err != nil {    
    msg := fmt.Sprintf("error getting the  result with id %d", id)
    switch err {
    case orm.NoResult:
        err = errors.Wrapf(err, msg);
    default: 
        err = errors.NotFound(err, msg);  
    }
    return Result{}, err
  }
  return result, nil 
}
// after the error wraping the result will be 
// err.Error() -> error getting the result with id 10: whatever it comes from the orm
```

Now the interactor:

```
func getInteractor(idString string) (Result, error) {
  id, err := strconv.Atoi(idString)
  if err != nil { 
    err = errors.BadRequest.Wrapf(err, "interactor converting id to int")
    err = errors.AddContext(err, "id", "wrong id format, should be an integer)
 
    return Result{}, err
  }
  return repository.getFromRepository(id) 
}
```

And finally the web server:

```
r := mux.NewRouter()
r.HandleFunc("/result/{id}", ResultHandler)
func ResultHandler(w http.ResponseWriter, r *http.Request) {
  vars := mux.Vars(r)
  result, err := interactor.getInteractor(vars["id"])
  if err != nil { 
    handleError(w, err) 
  }
  fmt.Fprintf(w, result)
}
func handleError(w http.ResponseWriter, err error) { 
   var status int
   errorType := errors.GetType(err)
   switch errorType {
     case BadRequest: 
      status = http.StatusBadRequest
     case NotFound: 
      status = http.StatusNotFound
     default: 
      status = http.StatusInternalServerError
   }
   w.WriteHeader(status) 
   
   if errorType == errors.NoType {
     log.Errorf(err)
   }
   fmt.Fprintf(w,"error %s", err.Error()) 
   
   errorContext := errors.GetContext(err) 
   if errorContext != nil {
     fmt.Printf(w, "context %v", errorContext)
   }
}
```

As you may see with an exported type and some exported values we can make our life much easier dealing with errors. One thing that I like from this solution is that by design when creating an error we explicitly show his type.

Do you have any suggestion? Comment below.