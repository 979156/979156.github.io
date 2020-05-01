---
title: "JWT Auth"
date: 2020-05-01T15:28:46+09:00
---
## JWT Auth
Example to run atreugo server with basic JWT authentication. [![Link](https://img.icons8.com/officexs/16/000000/link.png)](https://github.com/atreugo/examples/tree/master/jwt_auth)

*main.go*
```go
package main

import (
	"fmt"

	"github.com/savsgio/atreugo/v11"
	"github.com/savsgio/go-logger/v2"
	"github.com/valyala/fasthttp"
)

func init() { //nolint:gochecknoinits
	logger.SetLevel(logger.DEBUG)
}

func main() {
	config := atreugo.Config{
		Addr: "0.0.0.0:8000",
	}
	server := atreugo.New(config)

	// Register authentication middleware at first of all
	server.UseBefore(authMiddleware)

	// Register index route
	server.GET("/", func(ctx *atreugo.RequestCtx) error {
		return ctx.HTTPResponse(fmt.Sprintf(`<h1>You are login with JWT</h1>
				JWT cookie value: %s`, ctx.Request.Header.Cookie("atreugo_jwt")))
	})

	// Register login route
	server.GET("/login", func(ctx *atreugo.RequestCtx) error {
		qUser := []byte("savsgio")
		qPasswd := []byte("mypasswd")

		jwtCookie := ctx.Request.Header.Cookie("atreugo_jwt")

		if len(jwtCookie) == 0 {
			tokenString, expireAt := generateToken(qUser, qPasswd)

			// Set cookie for domain
			cookie := fasthttp.AcquireCookie()
			defer fasthttp.ReleaseCookie(cookie)

			cookie.SetKey("atreugo_jwt")
			cookie.SetValue(tokenString)
			cookie.SetExpire(expireAt)
			ctx.Response.Header.SetCookie(cookie)
		}

		return ctx.RedirectResponse("/", ctx.Response.StatusCode())
	})

	// Run
	if err := server.ListenAndServe(); err != nil {
		panic(err)
	}
}
```

*auth_utils.go*
```go
package main

import (
	"time"

	"github.com/dgrijalva/jwt-go"
	"github.com/savsgio/go-logger/v2"
)

var jwtSignKey = []byte("TestForFasthttpWithJWT")

type userCredential struct {
	Username []byte `json:"username"`
	Password []byte `json:"password"`
	jwt.StandardClaims
}

func generateToken(username []byte, password []byte) (string, time.Time) {
	logger.Debugf("Create new token for user %s", username)

	expireAt := time.Now().Add(1 * time.Minute)

	// Embed User information to `token`
	newToken := jwt.NewWithClaims(jwt.SigningMethodHS512, &userCredential{
		Username: username,
		Password: password,
		StandardClaims: jwt.StandardClaims{
			ExpiresAt: expireAt.Unix(),
		},
	})

	// token -> string. Only server knows the secret.
	tokenString, err := newToken.SignedString(jwtSignKey)
	if err != nil {
		logger.Error(err)
	}

	return tokenString, expireAt
}

func validateToken(requestToken string) (*jwt.Token, *userCredential, error) {
	logger.Debug("Validating token...")

	user := &userCredential{}
	token, err := jwt.ParseWithClaims(requestToken, user, func(token *jwt.Token) (interface{}, error) {
		return jwtSignKey, nil
	})

	return token, user, err
}
```

*middlewares.go*
```go
package main

import (
	"errors"

	"github.com/savsgio/atreugo/v11"
	"github.com/valyala/fasthttp"
)

// checkTokenMiddleware middleware to check jwt token authorization
func authMiddleware(ctx *atreugo.RequestCtx) error {
	// Avoid middleware when you are going to login view
	if string(ctx.Path()) == "/login" {
		return ctx.Next()
	}

	jwtCookie := ctx.Request.Header.Cookie("atreugo_jwt")

	if len(jwtCookie) == 0 {
		return ctx.ErrorResponse(errors.New("login required"), fasthttp.StatusForbidden)
	}

	token, _, err := validateToken(string(jwtCookie))
	if err != nil {
		return err
	}

	if !token.Valid {
		return ctx.ErrorResponse(errors.New("your session is expired, login again please"), fasthttp.StatusForbidden)
	}

	return ctx.Next()
}
```