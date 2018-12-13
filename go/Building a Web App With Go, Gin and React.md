# Building a Web App With Go, Gin and React

**Gin** is a high-performance micro-framework that delivers a very minimalistic framework that carries with it only the most essential features, libraries, and functionalities needed to build web applications and microservices. It makes it simple to build a request handling pipeline from modular, reusable pieces. It does this by allowing you to write middleware that can be plugged into one or more request handlers or groups of request handlers.

## Gin Features

Gin is a fast, simple yet fully featured and very efficient web framework for Go. Check out some of the features below that makes it a worthy framework to consider for your next Golang project.

- **Speed:** Gin is built for speed. The framework offers a Radix tree based routing, small memory foot print. No reflection. Predictable API performance.
- **Crash-Free**: Gin has the capability of catching crashes or panics during runtime, and can recover from it, this way your application will be always available.
- **Routing:** Gin provides a routing interface to allow you express how your web application or API routes should look.
- **JSON Validation:** Gin can parse and validate the JSON requests easily, checking for the existence of required values.
- **Error Management:** Gin provides a convenient way to collect all the errors occurred during a HTTP request. Eventually, a middleware can write them to a log file, to a database and send them through the network.
- **Built-In Rendering:** Gin provides an easy to use API for JSON, XML, and HTML rendering.

## Prerequisites

To follow along with this tutorial, you’ll need to have Go installed on your machine, a web browser to view the app, and a command line to execute build commands.

**Go** or as its normally called; *“Golang”*, is a programming language developed by Google for building modern software. Go is a language designed to get stuff done efficiently and fast. The key benefits of Go include:

- Strongly typed and garbage collected
- Blazing fast compile times
- Concurrency built in
- Extensive standard library

Head over to the [downloads section](https://golang.org/dl/) of the Go website, to get Go running on your machine.

## Building An App With Gin

We’ll be building a simple joke listing app with **Gin**. Our app will simply list some silly dad jokes. We are going to add authentication to it, all logged-in users will have the privilege to like and view jokes.

This will allow us illustrate how **Gin** can be used to develop web applications, and/or APIs.

![golang-gin-demo-shot](/img/38371873-6ccabb50-38e5-11e8-9b67-b97cc2ce98c6.png)

We’ll be making use of the following functionalities offered by Gin:

- Middleware
- Routing
- Routes Grouping

### Ready, Set, Go

We will write our entire Go application in a `main.go` file. Since its a small application, its going to be easy to build the application with just `go run` from the terminal.

We’ll create a new directory `golang-gin` in our Go workspace, and then a `main.go` file in it:

```shell
$ mkdir -p $GOPATH/src/github.com/user/golang-gin
$ cd $GOPATH/src/github.com/user/golang-gin
$ touch main.go
```

The content of the `main.go` file:

```go
package main

import (
  "net/http"

  "github.com/gin-gonic/contrib/static"
  "github.com/gin-gonic/gin"
)

func main() {
  // Set the router as the default one shipped with Gin
  router := gin.Default()

  // Serve frontend static files
  router.Use(static.Serve("/", static.LocalFile("./views", true)))

  // Setup route group for the API
  api := router.Group("/api")
  {
    api.GET("/", func(c *gin.Context) {
      c.JSON(http.StatusOK, gin.H {
        "message": "pong",
      })
    })
  }

  // Start and run the server
  router.Run(":3000")
}
```

We’ll need to create some more directories for our static files. In the same directory as the `main.go` file, let’s create a `views` folder. In the `views` folder, create a `js` folder and an `index.html` file in it.

The `index.html` file will be very simple for now:

```html
<!DOCTYPE html>
<html>
<head>
  <title>Jokeish App</title>
</head>

<body>
  <h1>Welcome to the Jokeish App</h1>
</body>
</html>
```

Before we test what we have to far, let’s install the added dependencies:

```shell
$ go get -u github.com/gin-gonic/gin
$ go get -u github.com/gin-gonic/contrib/static
```

To see what’s working, we’ll need to start our server by running `go run main.go`.

![go-gin-gorun-1](/img/38358547-700bc822-38bd-11e8-88a6-246559fbe57f.png)

Once the application is running, navigate to `http://localhost:3000` in your browser. If all went well, you should see level 1 header text **Welcome to the Jokeish App** displayed.

![golang-welcome-de](img/38385072-cdaa2168-3908-11e8-9889-e9c1e713ce0a.png)

### Defining The API

Let’s add some more code in our `main.go` file, for our API definitions. We’ll update our `main` function with two routes `/jokes/` and `/jokes/like/:jokeID`, to the route group `/api/`.

```go
func main() {
  // ... leave the code above untouched...

  // Our API will consit of just two routes
  // /jokes - which will retrieve a list of jokes a user can see
  // /jokes/like/:jokeID - which will capture likes sent to a particular joke
  api.GET("/jokes", JokeHandler)
  api.POST("/jokes/like/:jokeID", LikeJoke)
}

// JokeHandler retrieves a list of available jokes
func JokeHandler(c *gin.Context) {
  c.Header("Content-Type", "application/json")
  c.JSON(http.StatusOK, gin.H {
    "message":"Jokes handler not implemented yet",
  })
}

// LikeJoke increments the likes of a particular joke Item
func LikeJoke(c *gin.Context) {
  c.Header("Content-Type", "application/json")
  c.JSON(http.StatusOK, gin.H {
    "message":"LikeJoke handler not implemented yet",
  })
}
```

The content of the `main.go` file, should look like this:

```go
package main

import (
  "net/http"

  "github.com/gin-gonic/contrib/static"
  "github.com/gin-gonic/gin"
)

func main() {
  // Set the router as the default one shipped with Gin
  router := gin.Default()

  // Serve frontend static files
  router.Use(static.Serve("/", static.LocalFile("./views", true)))

  // Setup route group for the API
  api := router.Group("/api")
  {
    api.GET("/", func(c *gin.Context) {
      c.JSON(http.StatusOK, gin.H {
        "message": "pong",
      })
    })
  }
  // Our API will consit of just two routes
  // /jokes - which will retrieve a list of jokes a user can see
  // /jokes/like/:jokeID - which will capture likes sent to a particular joke
  api.GET("/jokes", JokeHandler)
  api.POST("/jokes/like/:jokeID", LikeJoke)

  // Start and run the server
  router.Run(":3000")
}

// JokeHandler retrieves a list of available jokes
func JokeHandler(c *gin.Context) {
  c.Header("Content-Type", "application/json")
  c.JSON(http.StatusOK, gin.H {
    "message":"Jokes handler not implemented yet",
  })
}

// LikeJoke increments the likes of a particular joke Item
func LikeJoke(c *gin.Context) {
  c.Header("Content-Type", "application/json")
  c.JSON(http.StatusOK, gin.H {
    "message":"LikeJoke handler not implemented yet",
  })
}
```

Let’s run our app again `go run main.go`, and access our routes; `http://localhost:3000/api/jokes` will return a `200 OK` header response, with message `jokes handler not implemented yet`, and a POST request to `http://localhost:3000/api/jokes/like/1` returns a `200 OK` header, and message `Likejoke handler not implemented yet`.

### Jokes Data

Since we already have our routes definition set, which does only one thing, which is to return a json response, we’ll spice our codebase a bit by adding some more code to it.

```go
// ... leave the code above untouched...

// Let's create our Jokes struct. This will contain information about a Joke

// Joke contains information about a single Joke
type Joke struct {
  ID     int     `json:"id" binding:"required"`
  Likes  int     `json:"likes"`
  Joke   string  `json:"joke" binding:"required"`
}

// We'll create a list of jokes
var jokes = []Joke{
  Joke{1, 0, "Did you hear about the restaurant on the moon? Great food, no atmosphere."},
  Joke{2, 0, "What do you call a fake noodle? An Impasta."},
  Joke{3, 0, "How many apples grow on a tree? All of them."},
  Joke{4, 0, "Want to hear a joke about paper? Nevermind it's tearable."},
  Joke{5, 0, "I just watched a program about beavers. It was the best dam program I've ever seen."},
  Joke{6, 0, "Why did the coffee file a police report? It got mugged."},
  Joke{7, 0, "How does a penguin build it's house? Igloos it together."},
}

func main() {
  // ... leave this block untouched...
}

// JokeHandler retrieves a list of available jokes
func JokeHandler(c *gin.Context) {
  c.Header("Content-Type", "application/json")
  c.JSON(http.StatusOK, jokes)
}

// LikeJoke increments the likes of a particular joke Item
func LikeJoke(c *gin.Context) {
  // confirm Joke ID sent is valid
  // remember to import the `strconv` package
  if jokeid, err := strconv.Atoi(c.Param("jokeID")); err == nil {
    // find joke, and increment likes
    for i := 0; i < len(jokes); i++ {
      if jokes[i].ID == jokeid {
        jokes[i].Likes += 1
      }
    }

    // return a pointer to the updated jokes list
    c.JSON(http.StatusOK, &jokes)
  } else {
    // Joke ID is invalid
    c.AbortWithStatus(http.StatusNotFound)
  }
}

// NB: Replace the JokeHandler and LikeJoke functions in the previous version to the ones above
```

With our code looking good, lets go ahead and test our API. We can test with `cURL`or `postman` , and sending a `GET` request to `http://localhost:3000/jokes` to get the full list of jokes, and a `POST` request to `http://localhost:3000/jokes/like/{jokeid}` to increment the likes of a joke.

```shell
$ curl http://localhost:3000/api/jokes

$ curl -X POST http://localhost:3000/api/jokes/like/4
```

### Building the UI (React)

We have our API in place, so let’s build a frontend to present the data from our API. For this, we’ll be using React. We won’t go too deep into React as it will be out of scope for this tutorial. If you need to learn more about React, checkout the official [tutorial](https://facebook.github.io/react/docs/tutorial.html). You can implement the UI with any frontend framework you’re comfortable with.

### Setup

We’ll edit the `index.html` file to add external libraries needed to run React, we’ll then need to create an`app.jsx` file in the `views/js` directory, which will contain our React code.

Our `index.html` file should look like this:

```html
<!DOCTYPE html>
<html>

<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" />
  <title>Jokeish App</title>
  <script src="http://code.jquery.com/jquery-2.1.4.min.js"></script>
  <script src="https://cdn.auth0.com/js/auth0/9.0/auth0.min.js"></script>
  <script type="application/javascript" src="https://unpkg.com/react@16.0.0/umd/react.production.min.js"></script>
  <script type="application/javascript" src="https://unpkg.com/react-dom@16.0.0/umd/react-dom.production.min.js"></script>
  <script type="application/javascript" src="https://unpkg.com/babel-standalone@6.26.0/babel.js"></script>
  <script type="text/babel" src="js/app.jsx"></script>
  <link href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.6/css/bootstrap.min.css" rel="stylesheet">
</head>

<body>
  <div id="app"></div>
</body>

</html>
```

### Building our Components

In React views are broken down into components. We’ll need to build some components. An `App` component as the main entry, that launches the application, a `Home` component which will face non logged-in users, a `LoggedIn` component with content only visible by authenticated users, and a `Joke` component to display a list of jokes. We’ll write all these components in the `app.jsx` file.

### The App component

This component bootstraps our entire React app. It decides on which component to show when a user is authenticated or not. We’ll start off with just its base, and later update it with more functionality.

```javascript
class App extends React.Component {
  render() {
    if (this.loggedIn) {
      return (<LoggedIn />);
    } else {
      return (<Home />);
    }
  }
}
```

### The Home component

This component is shown to non logged-in users. And a button which opens a Hosted lock screen (we’ll add this functionality later), where they can signup, or login.

```jsx
class Home extends React.Component {
  render() {
    return (
      <div className="container">
        <div className="col-xs-8 col-xs-offset-2 jumbotron text-center">
          <h1>Jokeish</h1>
          <p>A load of Dad jokes XD</p>
          <p>Sign in to get access </p>
          <a onClick={this.authenticate} className="btn btn-primary btn-lg btn-login btn-block">Sign In</a>
        </div>
      </div>
    )
  }
}
```

### LoggedIn component

This component is displayed when a user is authenticated. It stores in its `state` an array of jokes which is populated when the component mounts.

```jsx
class LoggedIn extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      jokes: []
    }
  }

  render() {
    return (
      <div className="container">
        <div className="col-lg-12">
          <br />
          <span className="pull-right"><a onClick={this.logout}>Log out</a></span>
          <h2>Jokeish</h2>
          <p>Let's feed you with some funny Jokes!!!</p>
          <div className="row">
            {this.state.jokes.map(function(joke, i){
              return (<Joke key={i} joke={joke} />);
            })}
          </div>
        </div>
      </div>
    )
  }
}
```

### The Joke component

The `Joke` component will contain information about each item from the jokes response to be displayed.

```jsx
class Joke extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      liked: ""
    }
    this.like = this.like.bind(this);
  }

  like() {
    // ... we'll add this block later
  }

  render() {
    return (
      <div className="col-xs-4">
        <div className="panel panel-default">
          <div className="panel-heading">#{this.props.joke.id} <span className="pull-right">{this.state.liked}</span></div>
          <div className="panel-body">
            {this.props.joke.joke}
          </div>
          <div className="panel-footer">
            {this.props.joke.likes} Likes &nbsp;
            <a onClick={this.like} className="btn btn-default">
              <span className="glyphicon glyphicon-thumbs-up"></span>
            </a>
          </div>
        </div>
      </div>
    )
  }
}
```

We’ve written our components, now let’s tell React where to render the app. We’ll add the block of code below to the bottom of our `app.jsx` file.

```javascript
ReactDOM.render(<App />, document.getElementById('app'));
```

Let’s restart our Go server `go run main.go`, and head over to our app’s URL `http://localhost:3000/`. You’d see the `Home` component is being rendered.

![golang-gin-home-component](img/38368612-a452879a-38dd-11e8-8a63-5cc7530fc9bd.png)

## Securing our Jokes App With Auth0

> Disclaimer: This isn’t a sponsored content.

**Auth0** issues [JSON Web Tokens](https://jwt.io/) on every login for your users. This means that you can have a solid [identity infrastructure](https://auth0.com/docs/identityproviders), including [single sign-on](https://auth0.com/docs/sso/single-sign-on), user management, support for social identity providers (Facebook, Github, Twitter, etc.), enterprise identity providers (Active Directory, LDAP, SAML, etc.) and your own database of users with just a few lines of code.

We can easily set up authentication in our GIN app by using Auth0. You’ll need an ccount to follow along with this part. If you don’t already have an Auth0 account, [sign up](https://auth0.com/signup) for one now.

### Creating the API client

Our tokens will be generated with Auth0, so we need to create an API and a Client from our Auth0 dashboard. If you haven’t already, [sign up](https://auth0.com/signup) for an Auth0 account.

To create a new API, navigate to the [APIs section](https://manage.auth0.com/#/apis) in your dashboard, and click the **Create API** button.

![golang-gin-apis-section](img/38367092-a7bbfffa-38d9-11e8-949e-04542283606c.png)

Choose an API **name**, and an **identifier**. The identifier will be the **audience** for the middleware. The **Signing Algorithm** should be **RS256**.

To create a new Client, navigate to the [clients section]() in your dashboard, and click the **Create Client** button, and select type `Regular Web Applications`.

![golang-gin-create-client](img/38453889-025202bc-3a55-11e8-9ff6-1d5265ebecd9.png)

Once the client is created, take note of the `client_id` and `client_secret`, as we’ll need it later.

![golang-gin-client-page](img/38453892-09268fea-3a55-11e8-9c3d-68de4b26295d.png)

We need to add the credentials needed for our API to an environment vairable. In the root directory, create a new file `.env` and add the following to it, with the details from the Auth0 dashboard:

```
export AUTH0_API_CLIENT_SECRET=""
export AUTH0_CLIENT_ID=""
export AUTH0_DOMAIN="yourdomain.auth0.com"
export AUTH0_API_AUDIENCE=""
```

### Securing our API Endpoints

Currently, our API is open to the world, so we need to secure them, so only authorized users can access them.

We are going to make use of a **JWT Middleware** to check for a valid *JSON Web Token*from each requests hitting our Endpoints.

Let’s create our middleware:

```go
// ...

var jwtMiddleWare *jwtmiddleware.JWTMiddleware

func main() {
  jwtMiddleware := jwtmiddleware.New(jwtmiddleware.Options{
    ValidationKeyGetter: func(token *jwt.Token) (interface{}, error) {
      aud := os.Getenv("AUTH0_API_AUDIENCE")
      checkAudience := token.Claims.(jwt.MapClaims).VerifyAudience(aud, false)
      if !checkAudience {
        return token, errors.New("Invalid audience.")
      }
      // verify iss claim
      iss := os.Getenv("AUTH0_DOMAIN")
      checkIss := token.Claims.(jwt.MapClaims).VerifyIssuer(iss, false)
      if !checkIss {
        return token, errors.New("Invalid issuer.")
      }

      cert, err := getPemCert(token)
      if err != nil {
        log.Fatalf("could not get cert: %+v", err)
      }

      result, _ := jwt.ParseRSAPublicKeyFromPEM([]byte(cert))
      return result, nil
    },
    SigningMethod: jwt.SigningMethodRS256,
  })

  // register our actual jwtMiddleware
  jwtMiddleWare = jwtMiddleware

  // ... the rest of the code below this function doesn't change yet
}

// authMiddleware intercepts the requests, and check for a valid jwt token
func authMiddleware() gin.HandlerFunc {
  return func(c *gin.Context) {
    // Get the client secret key
    err := jwtMiddleWare.CheckJWT(c.Writer, c.Request)
    if err != nil {
      // Token not found
      fmt.Println(err)
      c.Abort()
      c.Writer.WriteHeader(http.StatusUnauthorized)
      c.Writer.Write([]byte("Unauthorized"))
      return
    }
  }
}
```

The above code, we have a new `jwtMiddleWare` variable which is initialized in the `main` function, and is used in the `authMiddleware` middle function. If you notice, we are pulling our server-side credentials from an environment variable (one of the tenets of a **12-factor app**). Our middleware checks and receives a token from a request, it calls the `jwtMiddleWare.CheckJWT` method to validate the token sent.

Let’s also write the function to return the JSON Web Keys:

```go
// ... the code above is untouched...

// Jwks stores a slice of JSON Web Keys
type Jwks struct {
  Keys []JSONWebKeys `json:"keys"`
}

type JSONWebKeys struct {
  Kty string   `json:"kty"`
  Kid string   `json:"kid"`
  Use string   `json:"use"`
  N   string   `json:"n"`
  E   string   `json:"e"`
  X5c []string `json:"x5c"`
}

func main() {
  // ... the code in this method is untouched...
}

func getPemCert(token *jwt.Token) (string, error) {
  cert := ""
  resp, err := http.Get(os.Getenv("AUTH0_DOMAIN") + ".well-known/jwks.json")
  if err != nil {
    return cert, err
  }
  defer resp.Body.Close()

  var jwks = Jwks{}
  err = json.NewDecoder(resp.Body).Decode(&jwks)

  if err != nil {
    return cert, err
  }

  x5c := jwks.Keys[0].X5c
  for k, v := range x5c {
    if token.Header["kid"] == jwks.Keys[k].Kid {
      cert = "-----BEGIN CERTIFICATE-----\n" + v + "\n-----END CERTIFICATE-----"
    }
  }

  if cert == "" {
    return cert, errors.New("unable to find appropriate key.")
  }

  return cert, nil
}
```

### Using the JWT Middleware

Using the middleware is very straight forward. We just pass it as a parameter to our routes definition.

```go
...

api.GET("/jokes", authMiddleware(), JokeHandler)
api.POST("/jokes/like/:jokeID", authMiddleware(), LikeJoke)

...
```

Our `main.go` file should look like this:

```go
package main

import (
  "encoding/json"
  "errors"
  "fmt"
  "log"
  "net/http"
  "os"
  "strconv"

  jwtmiddleware "github.com/auth0/go-jwt-middleware"
  jwt "github.com/dgrijalva/jwt-go"
  "github.com/gin-gonic/contrib/static"
  "github.com/gin-gonic/gin"
)

type Response struct {
  Message string `json:"message"`
}

type Jwks struct {
  Keys []JSONWebKeys `json:"keys"`
}

type JSONWebKeys struct {
  Kty string   `json:"kty"`
  Kid string   `json:"kid"`
  Use string   `json:"use"`
  N   string   `json:"n"`
  E   string   `json:"e"`
  X5c []string `json:"x5c"`
}

type Joke struct {
  ID    int    `json:"id" binding:"required"`
  Likes int    `json:"likes"`
  Joke  string `json:"joke" binding:"required"`
}

/** we'll create a list of jokes */
var jokes = []Joke{
  Joke{1, 0, "Did you hear about the restaurant on the moon? Great food, no atmosphere."},
  Joke{2, 0, "What do you call a fake noodle? An Impasta."},
  Joke{3, 0, "How many apples grow on a tree? All of them."},
  Joke{4, 0, "Want to hear a joke about paper? Nevermind it's tearable."},
  Joke{5, 0, "I just watched a program about beavers. It was the best dam program I've ever seen."},
  Joke{6, 0, "Why did the coffee file a police report? It got mugged."},
  Joke{7, 0, "How does a penguin build it's house? Igloos it together."},
}

var jwtMiddleWare *jwtmiddleware.JWTMiddleware

func main() {
  jwtMiddleware := jwtmiddleware.New(jwtmiddleware.Options{
    ValidationKeyGetter: func(token *jwt.Token) (interface{}, error) {
      aud := os.Getenv("AUTH0_API_AUDIENCE")
      checkAudience := token.Claims.(jwt.MapClaims).VerifyAudience(aud, false)
      if !checkAudience {
        return token, errors.New("Invalid audience.")
      }
      // verify iss claim
      iss := os.Getenv("AUTH0_DOMAIN")
      checkIss := token.Claims.(jwt.MapClaims).VerifyIssuer(iss, false)
      if !checkIss {
        return token, errors.New("Invalid issuer.")
      }

      cert, err := getPemCert(token)
      if err != nil {
        log.Fatalf("could not get cert: %+v", err)
      }

      result, _ := jwt.ParseRSAPublicKeyFromPEM([]byte(cert))
      return result, nil
    },
    SigningMethod: jwt.SigningMethodRS256,
  })

  jwtMiddleWare = jwtMiddleware
  // Set the router as the default one shipped with Gin
  router := gin.Default()

  // Serve the frontend
  router.Use(static.Serve("/", static.LocalFile("./views", true)))

  api := router.Group("/api")
  {
    api.GET("/", func(c *gin.Context) {
      c.JSON(http.StatusOK, gin.H{
        "message": "pong",
      })
    })
    api.GET("/jokes", authMiddleware(), JokeHandler)
    api.POST("/jokes/like/:jokeID", authMiddleware(), LikeJoke)
  }
  // Start the app
  router.Run(":3000")
}

func getPemCert(token *jwt.Token) (string, error) {
  cert := ""
  resp, err := http.Get(os.Getenv("AUTH0_DOMAIN") + ".well-known/jwks.json")
  if err != nil {
    return cert, err
  }
  defer resp.Body.Close()

  var jwks = Jwks{}
  err = json.NewDecoder(resp.Body).Decode(&jwks)

  if err != nil {
    return cert, err
  }

  x5c := jwks.Keys[0].X5c
  for k, v := range x5c {
    if token.Header["kid"] == jwks.Keys[k].Kid {
      cert = "-----BEGIN CERTIFICATE-----\n" + v + "\n-----END CERTIFICATE-----"
    }
  }

  if cert == "" {
    return cert, errors.New("unable to find appropriate key")
  }

  return cert, nil
}

// authMiddleware intercepts the requests, and check for a valid jwt token
func authMiddleware() gin.HandlerFunc {
  return func(c *gin.Context) {
    // Get the client secret key
    err := jwtMiddleWare.CheckJWT(c.Writer, c.Request)
    if err != nil {
      // Token not found
      fmt.Println(err)
      c.Abort()
      c.Writer.WriteHeader(http.StatusUnauthorized)
      c.Writer.Write([]byte("Unauthorized"))
      return
    }
  }
}

// JokeHandler returns a list of jokes available (in memory)
func JokeHandler(c *gin.Context) {
  c.Header("Content-Type", "application/json")

  c.JSON(http.StatusOK, jokes)
}

func LikeJoke(c *gin.Context) {
  // Check joke ID is valid
  if jokeid, err := strconv.Atoi(c.Param("jokeID")); err == nil {
    // find joke and increment likes
    for i := 0; i < len(jokes); i++ {
      if jokes[i].ID == jokeid {
        jokes[i].Likes = jokes[i].Likes + 1
      }
    }
    c.JSON(http.StatusOK, &jokes)
  } else {
    // the jokes ID is invalid
    c.AbortWithStatus(http.StatusNotFound)
  }
}
```

Let’s install the `jwtmiddleware` libraries:

```shell
$ go get -u github.com/auth0/go-jwt-middleware
$ go get -u github.com/dgrijalva/jwt-go
```

Let’s source our environment file, and restart our app server:

```go
$ source .env
$ go run main.go
```

Now if we try accessing any of the endpoints, you’d be faced with a `401 Unauthorized` error. That’s because we need to send along a token with the request.

### Login with Auth0 and React

Let’s implement a login system, so users can login or create accounts, so they have access to our jokes. We’ll add to our `app.jsx` file, the following Auth0 credentials:

- `AUTH0_CLIENT_ID`
- `AUTH0_DOMAIN`
- `AUTH0_CALLBACK_URL` - The URL of your app
- `AUTH0_API_AUDIENCE`

> You can find the `AUTH0_CLIENT_ID`, `AUTH0_DOMAIN`, and `AUTH0_API_AUDIENCE` data from your Auth0 [management dashboard](https://manage.auth0.com/).

We need to set a `callback` which Auth0 redirects to. Navigate to the Clients section in your dashboard, and in the settings, let’s set the callback to `http://localhost:3000`:

![auth0-golang-gin-allowed-callbacks](img/38453893-0cf84032-3a55-11e8-9b03-d26ba1d3e56a.png)

With the credentials in place, lets update our React components.

#### APP component

```jsx
const AUTH0_CLIENT_ID = "aIAOt9fkMZKrNsSsFqbKj5KTI0ObTDPP";
const AUTH0_DOMAIN = "hakaselabs.auth0.com";
const AUTH0_CALLBACK_URL = location.href;
const AUTH0_API_AUDIENCE = "golang-gin";

class App extends React.Component {
  parseHash() {
    this.auth0 = new auth0.WebAuth({
      domain: AUTH0_DOMAIN,
      clientID: AUTH0_CLIENT_ID
    });
    this.auth0.parseHash(window.location.hash, (err, authResult) => {
      if (err) {
        return console.log(err);
      }
      if (
        authResult !== null &&
        authResult.accessToken !== null &&
        authResult.idToken !== null
      ) {
        localStorage.setItem("access_token", authResult.accessToken);
        localStorage.setItem("id_token", authResult.idToken);
        localStorage.setItem(
          "profile",
          JSON.stringify(authResult.idTokenPayload)
        );
        window.location = window.location.href.substr(
          0,
          window.location.href.indexOf("#")
        );
      }
    });
  }

  setup() {
    $.ajaxSetup({
      beforeSend: (r) => {
        if (localStorage.getItem("access_token")) {
          r.setRequestHeader(
            "Authorization",
            "Bearer " + localStorage.getItem("access_token")
          );
        }
      }
    });
  }

  setState() {
    let idToken = localStorage.getItem("id_token");
    if (idToken) {
      this.loggedIn = true;
    } else {
      this.loggedIn = false;
    }
  }

  componentWillMount() {
    this.setup();
    this.parseHash();
    this.setState();
  }

  render() {
    if (this.loggedIn) {
      return <LoggedIn />;
    }
    return <Home />;
  }
}
```

We updated the App component with three component methods (`setup`, `parseHash` and `setState`), and a lifecycle method `componentWillMount`. The `parseHash` method, initializes the `auth0` `webAuth` client, and parses the hash to a more readable format, saving them to localSt. to show the lock screen, capture and store the user token and add the correct authorization header to any requests to our API

#### Home component

Our Home component will be updated, we’ll add the functionality for the `authenticate` method, which will trigger the hosted lock screen to display, and allow our users login or signup.

```jsx
class Home extends React.Component {
  constructor(props) {
    super(props);
    this.authenticate = this.authenticate.bind(this);
  }
  authenticate() {
    this.WebAuth = new auth0.WebAuth({
      domain: AUTH0_DOMAIN,
      clientID: AUTH0_CLIENT_ID,
      scope: "openid profile",
      audience: AUTH0_API_AUDIENCE,
      responseType: "token id_token",
      redirectUri: AUTH0_CALLBACK_URL
    });
    this.WebAuth.authorize();
  }

  render() {
    return (
      <div className="container">
        <div className="row">
          <div className="col-xs-8 col-xs-offset-2 jumbotron text-center">
            <h1>Jokeish</h1>
            <p>A load of Dad jokes XD</p>
            <p>Sign in to get access </p>
            <a
              onClick={this.authenticate}
              className="btn btn-primary btn-lg btn-login btn-block"
            >
              Sign In
            </a>
          </div>
        </div>
      </div>
    );
  }
}
```

#### LoggedIn Component

We will update the `LoggedIn` component to communicate with our API, and pull all jokes, pass each joke as a `prop` to the `Joke` component, which renders a bootstrap pannel. Let’s write those:

```jsx
class LoggedIn extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      jokes: []
    };

    this.serverRequest = this.serverRequest.bind(this);
    this.logout = this.logout.bind(this);
  }

  logout() {
    localStorage.removeItem("id_token");
    localStorage.removeItem("access_token");
    localStorage.removeItem("profile");
    location.reload();
  }

  serverRequest() {
    $.get("http://localhost:3000/api/jokes", res => {
      this.setState({
        jokes: res
      });
    });
  }

  componentDidMount() {
    this.serverRequest();
  }

  render() {
    return (
      <div className="container">
        <br />
        <span className="pull-right">
          <a onClick={this.logout}>Log out</a>
        </span>
        <h2>Jokeish</h2>
        <p>Let's feed you with some funny Jokes!!!</p>
        <div className="row">
          <div className="container">
            {this.state.jokes.map(function(joke, i) {
              return <Joke key={i} joke={joke} />;
            })}
          </div>
        </div>
      </div>
    );
  }
}
```

#### Joke Component

We’ll also update the `Joke` component to format each Joke item passed to it from the Parent compoent (`LoggedIn`), and add a `like` method, which will increment the likes of a Joke.

```jsx
class Joke extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      liked: "",
      jokes: []
    };
    this.like = this.like.bind(this);
    this.serverRequest = this.serverRequest.bind(this);
  }

  like() {
    let joke = this.props.joke;
    this.serverRequest(joke);
  }
  serverRequest(joke) {
    $.post(
      "http://localhost:3000/api/jokes/like/" + joke.id,
      { like: 1 },
      res => {
        console.log("res... ", res);
        this.setState({ liked: "Liked!", jokes: res });
        this.props.jokes = res;
      }
    );
  }

  render() {
    return (
      <div className="col-xs-4">
        <div className="panel panel-default">
          <div className="panel-heading">
            #{this.props.joke.id}{" "}
            <span className="pull-right">{this.state.liked}</span>
          </div>
          <div className="panel-body">{this.props.joke.joke}</div>
          <div className="panel-footer">
            {this.props.joke.likes} Likes &nbsp;
            <a onClick={this.like} className="btn btn-default">
              <span className="glyphicon glyphicon-thumbs-up" />
            </a>
          </div>
        </div>
      </div>
    )
  }
}
```

### Putting it all together

With the UI and API complete, we can test our app. We’ll start of by booting our server `source .env && go run main.go`, and the navigate to `http://localhost:3000` from any browser, you should see the `Home` component with a signin button. Clicking on the signin button will redirect to a hosted Lock page, create an account or login, to continue using the application.

![golang-gin-home-component](img/38368612-a452879a-38dd-11e8-8a63-5cc7530fc9bd.png)*Home*

![golang-gin-login-screen](img/38369278-4685f35c-38df-11e8-97ac-98b2b9d07bab.png)*Auth0 Hosted Lock Screen*

![golang-gin-demo-shot](img/38371873-6ccabb50-38e5-11e8-9b67-b97cc2ce98c6.png)*LoggedIn App view*

### Conclusion

Congrats! You have learned how to build an application and an API with Go and the GIN framework.

This tutorial is designed to help you get started on building and adding authentication to a Golang app with the GIN framework.

Did I miss something important? Let me know of it in the comments.