---
title: "Passing Parameters in Golang HTTP Context"
date: 2021-03-01T20:44:13+08:00
draft: false
categories:
- Web
- Golang
tags:
- Web
- Golang
- Gin-Gonic
- HTTP
---
When developing HTTP APIs, we may have to process the same request-specific data throughout middlewares. Since it's a quite common pattern, I decide to figure it out and share how I solve it.

<!--more-->
## Example
Here we are using [Gin](https://github.com/gin-gonic/gin), a web framework written in golang featuring performance and good productivity.

Suppose we have an api endpoint `/api/hello` handled by `HelloHandleFunc`. For each incoming request, we want to authenticate its JWT (in `Authentication` header) and pass the decoded user ID to `HelloHandleFunc`. [This website](https://jwt.io) clearly illustrates what JWT is.

The main idea is to make use of `context.Context` in `http.Request`. The context of each request controls the entire request life cycle. Thus, we can set request-specific data (key-value pairs) in the context using `context.WithValue()` and retrives it using `Context.Value()`.

The following code shows the implementation of `AddUserID()` middleware, which summarizes the above procedure:
```go
func extractToken(r *http.Request) string {
	bearToken := r.Header.Get(conf.JWTAuthHeader)
	strArr := strings.Split(bearToken, " ")
	if len(strArr) == 2 {
		return strArr[1]
	}
	return ""
}
func AddUserID() gin.HandlerFunc {
	return func(c *gin.Context) {
        accessToken := extractToken(c.Request)
        if accessToken == "" {
		    c.AbortWithStatus(http.StatusUnauthorized)
			return
		}
        userID := DecodeJWT(accessToken)
		c.Request = c.Request.WithContext(context.WithValue(c.Request.Context(), "user_id", userID))
		c.Next()
	}
}
```
Retrieve `user_id` from the context in `HelloHandleFunc`:
```go
func HelloHandleFunc(c *gin.Context) {
	customerID, ok := c.Request.Context().Value("user_id").(string)
	if !ok {
		c.Abort()
		return
	}
	c.String(http.StatusOK, fmt.Printf("hi, %s!", customerID))
	return
}
```
Register the route with `AddUserID()` middleware enabled:
```go
router := gin.New()
group := router.Group("/api/hello")
group.Use(AddUserID())
group.GET("/hello", HelloHandleFunc)
```
Start the server:
```go
router.Run(":8080")
```