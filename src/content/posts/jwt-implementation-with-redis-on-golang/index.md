---
title: JWT Implementation with Redis on Golang
published: 2024-03-14
description: "Best practice for implementing JWT with Redis on Golang."
tags: ["JWT", "Redis", "Security", "Golang"]
category: Notes
draft: false
---

For now, JWT (JSON Web Token) is a popular way to authenticate users in web applications. JWT is a token that is used to authenticate users and is passed in the HTTP header. JWT is a self-contained token that contains information about the user. JWT is signed with a secret key, so it is secure and cannot be tampered with. The body of the JWT separated by a dot (`.`) contains three parts: header, payload, and signature. The header contains information about the algorithm used to sign the token. The payload contains information about the user. The signature is used to verify the token.

As far as I know, many developers implement JWT in a unsafe way. They store the JWT in the client-side (e.g., local storage, session storage, or cookie). This is not safe because:
1. If the JWT is stolen, the attacker can impersonate the user.
2. If the JWT is stolen, the attacker can access until the JWT is expired.
3. JWT can used even if the user has logged out.


How to solve that problem? We have 2 options:
1. Create the token with short expiration time (e.g., 15 minutes).
2. Store the token to server-side (e.g., database or cache).

The first approach is good, but it has a problem. The user must re-login every 15 minutes. This is not user-friendly. The second approach is better. The token is stored in the server-side, so the user cannot access the token. The problem is server-side must do query to database check the token. This is not efficient. The best practice is to store the token in cache. In this post, I will show you how to implement JWT with Redis in Golang. I use this approach in my projects, and it works well.

| Token         | Expired  | Function                         |
|---------------|----------|----------------------------------|
| Access Token  | 15 menit | User Access                      |
| Refresh Token | 1 bulan  | Refresh Access Token             |

The token is separated into 2 type: access token and refresh token. The access token is used to authenticate the user. The access token has a short expiration time (e.g., 15 minutes). The refresh token is used to get a new access token. The refresh token has a long expiration time (e.g., 7 days). The refresh token is stored in the Redis cache. When the user logs out, the refresh token is deleted from the Redis cache.

Why do we need to separate the token into 2 types?
1. If the access token is expired, the user must re-login. This is not user-friendly.
2. If the access token is stolen, the attacker can impersonate the user. The access token has a short expiration time, so the attacker cannot impersonate the user for a long time.
3. Logout process is easy. Just delete the refresh token from the Redis cache.


## Backend

We need to define the JWT models. We have 3 models: `JwtTokenDetails`, `JwtAuthModel`, and `JwtRtModel`. The `JwtTokenDetails` model contains the access token, refresh token, access UUID, refresh UUID, access token expiration time, and refresh token expiration time. The `JwtAuthModel` model contains the user ID, email, first name, last name, role, and duration. The `JwtRtModel` model contains the user ID, email, role, and duration.
```go
// JWT Models
package models

import "github.com/google/uuid"

type JwtTokenDetails struct {
	AccessToken  string
	RefreshToken string
	AccessUuid   string
	RefreshUuid  string
	AtExpires    int64
	RtExpires    int64
}

type JwtAuthModel struct {
	AccessId  uuid.UUID
	RefreshId uuid.UUID
	UserId    uuid.UUID
	Email     string
	FirstName string
	LastName  string
	Role      int16
	Duration  int64
}

type JwtRtModel struct {
	AccessId  uuid.UUID
	RefreshId uuid.UUID
	UserId    uuid.UUID
	Email     string
	Role      int16
	Duration  int64
}
```

Then we create JWT utils for handling generation of JWT. We have 2 functions: `GenerateNewAuthToken` and `GenerateNewAccessToken`. The `GenerateNewAuthToken` function is used to generate a new access token and refresh token. The `GenerateNewAccessToken` function is used to generate a new access token. The access token is signed with the secret key, and the refresh token is signed with the refresh key.

```go
// JWT Utils
package utils

import (
	"github.com/icaksh/cripis/app/models"
	"os"
	"strconv"
	"time"

	"github.com/golang-jwt/jwt"
)

func GenerateNewAuthToken(par *models.JwtAuthModel) (*models.JwtTokenDetails, error) {
	td := &models.JwtTokenDetails{}
	var err error
	minutesCount, _ := strconv.Atoi(os.Getenv("JWT_ACCESS_TIME_KEY_EXPIRE_MINUTES_COUNT"))
	accessTime := time.Now().Add(time.Minute * time.Duration(minutesCount)).Unix()
	atm := &JWTAccess{
		AccessUuid: par.AccessId,
		User:       par.UserId,
		Role:       par.Role,
		StandardClaims: jwt.StandardClaims{
			ExpiresAt: accessTime,
		},
	}
	accessToken, err := GenerateNewAccessToken(atm)

	rtm := &JWTRefresh{
		RefreshUuid: par.RefreshId,
		User:        par.UserId,
		Email:       par.Email,
		FirstName:   par.FirstName,
		LastName:    par.LastName,
		Role:        par.Role,
		StandardClaims: jwt.StandardClaims{
			ExpiresAt: par.Duration,
		},
	}
	refreshToken, err := GenerateNewRefreshToken(rtm)

	if err != nil {
		return nil, err
	}

	td = &models.JwtTokenDetails{
		AccessUuid:   par.AccessId.String(),
		AccessToken:  accessToken,
		RefreshUuid:  par.RefreshId.String(),
		RefreshToken: refreshToken,
		AtExpires:    accessTime,
		RtExpires:    par.Duration,
	}
	return td, err
}

func GenerateNewAccessToken(atm *JWTAccess) (string, error) {
	secret := os.Getenv("JWT_SECRET_KEY")

	token := jwt.NewWithClaims(jwt.SigningMethodHS256, atm)

	accessToken, err := token.SignedString([]byte(secret))
	if err != nil {
		// Return error, it JWT token generation failed.
		return "", err
	}
	return accessToken, nil
}

func GenerateNewRefreshToken(rtm *JWTRefresh) (string, error) {
	secret := os.Getenv("JWT_REFRESH_KEY")

	token := jwt.NewWithClaims(jwt.SigningMethodHS256, rtm)

	refreshToken, err := token.SignedString([]byte(secret))
	if err != nil {
		return "nil", err
	}
	return refreshToken, nil
}
```

Now, we create utils for JWT validation and extraction. We have 4 functions: `ExtractRefreshToken`, `ExtractRefreshTokenMetadata`, `ExtractTokenMetadata`, and `TokenValid`. The `ExtractRefreshToken` function is used to extract the refresh token from the request body. The `ExtractRefreshTokenMetadata` function is used to extract the refresh token from the request header. The `ExtractTokenMetadata` function is used to extract the access token from the request header. The `TokenValid` function is used to check if the access token is valid.

```go
package utils

import (
	"os"
	"strings"

	"github.com/gofiber/fiber/v2"
	"github.com/golang-jwt/jwt"
	"github.com/google/uuid"
)

type JWTAccess struct {
	AccessUuid uuid.UUID `json:"accessUuid"`
	User       uuid.UUID `json:"userId"`
	Role       int16     `json:"role"`
	jwt.StandardClaims
}

type JWTRefresh struct {
	RefreshUuid uuid.UUID `json:"refreshUuid"`
	User        uuid.UUID `json:"userId"`
	FirstName   string    `json:"firstName"`
	LastName    string    `json:"lastName"`
	Email       string    `json:"email"`
	Role        int16     `json:"role"`
	jwt.StandardClaims
}

func ExtractRefreshToken(t string) (*JWTRefresh, error) {
	token, err := jwt.ParseWithClaims(t, &JWTRefresh{}, jwtKeyFunc)
	if err != nil {
		return nil, err
	}
	claims, ok := token.Claims.(*JWTRefresh)
	if ok && token.Valid {
		return &JWTRefresh{
			RefreshUuid: claims.RefreshUuid,
			User:        claims.User,
			Email:       claims.Email,
			FirstName:   claims.FirstName,
			LastName:    claims.LastName,
			Role:        claims.Role,
			StandardClaims: jwt.StandardClaims{
				ExpiresAt: claims.ExpiresAt,
			},
		}, nil
	}
	return nil, err
}

func ExtractRefreshTokenMetadata(c *fiber.Ctx) (*JWTRefresh, error) {
	token, err := verifyRefreshToken(c)
	if err != nil {
		return nil, err
	}

	// Setting and checking token and credentials.
	claims, ok := token.Claims.(*JWTRefresh)
	if ok && token.Valid {
		// Expires time.
		return &JWTRefresh{
			RefreshUuid: claims.RefreshUuid,
			User:        claims.User,
			Email:       claims.Email,
			FirstName:   claims.FirstName,
			LastName:    claims.LastName,
			Role:        claims.Role,
			StandardClaims: jwt.StandardClaims{
				ExpiresAt: claims.ExpiresAt,
			},
		}, nil
	}

	return nil, err
}

func ExtractTokenMetadata(c *fiber.Ctx) (*JWTAccess, error) {
	token, err := verifyToken(c)
	if err != nil {
		return nil, err
	}

	claims, ok := token.Claims.(*JWTAccess)
	if ok && token.Valid {
		return &JWTAccess{
			AccessUuid: claims.AccessUuid,
			User:       claims.User,
			Role:       claims.Role,
			StandardClaims: jwt.StandardClaims{
				ExpiresAt: claims.ExpiresAt,
			},
		}, nil
	}

	return nil, err
}

func TokenValid(c *fiber.Ctx) error {
	token, err := verifyToken(c)
	if err != nil {
		return err
	}
	if _, ok := token.Claims.(jwt.Claims); !ok && !token.Valid {
		return err
	}
	return nil
}

func extractToken(c *fiber.Ctx) string {
	bearToken := c.Get("Authorization")

	// Normally Authorization HTTP header.
	onlyToken := strings.Split(bearToken, " ")
	if len(onlyToken) == 2 {
		return onlyToken[1]
	}

	return ""
}

func verifyToken(c *fiber.Ctx) (*jwt.Token, error) {
	tokenString := extractToken(c)

	token, err := jwt.ParseWithClaims(tokenString, &JWTAccess{}, jwtKeyFunc)
	if err != nil {
		return nil, err
	}

	return token, nil
}

func verifyRefreshToken(c *fiber.Ctx) (*jwt.Token, error) {
	tokenString := extractToken(c)

	token, err := jwt.ParseWithClaims(tokenString, &JWTRefresh{}, jwtKeyFunc)
	if err != nil {
		return nil, err
	}

	return token, nil
}
func jwtKeyFunc(token *jwt.Token) (interface{}, error) {
	return []byte(os.Getenv("JWT_SECRET_KEY")), nil
}
func jwtRefreshKeyFunc(token *jwt.Token) (interface{}, error) {
	return []byte(os.Getenv("JWT_REFRESH_KEY")), nil
}
```

Next, we need to create the JWT queries for Redis. We use the `github.com/go-redis/redis` package to connect to the Redis database. We have 3 functions: `CreateAuth`, `FetchAuth` and `DeleteAuth`. The `CreateAuth` function is used to create the refresh token in the Redis cache. The `FetchAuth` function is used to get the refresh token from the Redis cache. The `DeleteAuth` function is used to delete the refresh token from the Redis cache. 

```go
package queries

import (
	"fmt"
	"github.com/go-redis/redis/v7"
	"github.com/google/uuid"
	"github.com/icaksh/cripis/app/models"
	"github.com/icaksh/cripis/app/utils"
	"strconv"
	"time"
)

type JWTQueries struct {
	*redis.Client
}

func (q *JWTQueries) CreateAuth(userId uuid.UUID, td *models.JwtTokenDetails) error {
	fmt.Println(td)
	at := time.Until(time.Unix(td.AtExpires, 0))
	rt := time.Until(time.Unix(td.RtExpires, 0))
	fmt.Println(rt)
	errAccess := q.Set(td.AccessUuid, userId, at).Err()
	if errAccess != nil {
		return errAccess
	}
	errRefresh := q.Set(td.RefreshUuid, userId, rt).Err()
	if errRefresh != nil {
		return errRefresh
	}
	return nil
}

func (q *JWTQueries) FetchAuth(authD *utils.JWTAccess) (uint64, error) {
	userid, err := q.Get(authD.AccessUuid.String()).Result()
	if err != nil {
		return 0, err
	}
	userID, _ := strconv.ParseUint(userid, 10, 64)
	return userID, nil
}

func (q *JWTQueries) DeleteAuth(uuid uuid.UUID) (int64, error) {
	deleted, err := q.Del(uuid.String()).Result()
	if err != nil {
		return 0, err
	}
	return deleted, nil
}
```

After we create the queries, we can make database connection. We have 2 functions: `RedisConnection` and `RedisConnect`. The `RedisConnection` function is used to connect to the Redis database. The `RedisConnect` function is used to connect to the Redis database and return the JWT queries.

```go
package database

import (
	"github.com/go-redis/redis/v7"
	"os"
)

type RedisQueries struct {
	*queries.JWTQueries
}

func RedisConnection() (*redis.Client, error) {
	client := redis.NewClient(&redis.Options{
		Username: os.Getenv("REDIS_SERVER_NAME"),
		Password: os.Getenv("REDIS_SERVER_PASS"),
		Addr:     os.Getenv("REDIS_SERVER_URL"),
		DB:       0,
	})
	_, err := client.Ping().Result()
	if err != nil {
		return nil, err
	}
	return client, nil
}

func RedisConnect() (*RedisQueries, error) {
	redis, err := RedisConnection()
	if err != nil {
		return nil, err
	}

	return &RedisQueries{
		JWTQueries: &queries.JWTQueries{Client: redis},
	}, nil

}
```

Now we need to create the auth controller. We have 2 functions: `Login` and `Logout`. The `Login` function is used to authenticate the user and generate the access token and refresh token. The `Logout` function is used to delete the refresh token from the Redis cache.

```go
// Auth Controller
package controllers

import (
	"fmt"
	"github.com/google/uuid"
	"github.com/icaksh/cripis/app/models"
	"github.com/icaksh/cripis/app/utils"
	"github.com/icaksh/cripis/platform/database"
	"github.com/thanhpk/randstr"
	"os"
	"strconv"
	"strings"
	"time"

	"github.com/gofiber/fiber/v2"
	"golang.org/x/crypto/bcrypt"
)

func Login(c *fiber.Ctx) error {
	// Login Logic

	hoursCount, _ := strconv.Atoi(os.Getenv("JWT_REFRESH_TIME_KEY_EXPIRE_HOURS_COUNT"))

	if !creds.Remember {
		hoursCount, _ = strconv.Atoi(os.Getenv("JWT_REFRESH_TIME_KEY_EXPIRE_HOURS_COUNT_REMEMBER"))
	}
	refreshTime := time.Now().Add(time.Hour * time.Duration(hoursCount)).Unix()
	au := &models.JwtAuthModel{
		AccessId:  uuid.New(),
		RefreshId: uuid.New(),
		UserId:    auth.ID,
		Email:     auth.Email,
		FirstName: auth.FirstName,
		LastName:  auth.LastName,
		Role:      auth.Roles,
		Duration:  refreshTime,
	}
	token, err := utils.GenerateNewAuthToken(au)
	if err != nil {
		return utils.InternalServerError(c, err)
	}

	redisDb, err := database.RedisConnect()
	if err != nil {
		return utils.InternalServerError(c, err)
	}
	refresh := redisDb.CreateAuth(auth.ID, token)
	if refresh != nil {
		return utils.InternalServerError(c, err)
	}
	tokens := map[string]string{
		"access_token":  token.AccessToken,
		"refresh_token": token.RefreshToken,
	}
	return c.Status(fiber.StatusCreated).JSON(tokens)
}

func Logout(c *fiber.Ctx) error {

	au, err := utils.ExtractTokenMetadata(c)
	fmt.Println(au.AccessUuid)
	if err != nil {
		return c.Status(fiber.StatusUnauthorized).JSON(fiber.Map{
			"error":   true,
		})
	}
	redisDb, err := database.RedisConnect()

	if err != nil {
		return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
			"error":   true,
		})
	}

	deleted, err := redisDb.DeleteAuth(au.AccessUuid)
	if err != nil || deleted == 0 {
		return c.Status(fiber.StatusUnauthorized).JSON(fiber.Map{
			"error":   true,
		})
	}

	return c.Status(fiber.StatusOK).JSON(fiber.Map{
		"message": "Berhasil keluar",
	})
}
```

We also need to create the refresh token controller. We have 1 function: `RefreshToken`. The `RefreshToken` function is used to refresh the access token. The refresh token is sent in the request body, extracted, and then the user ID is fetched from the Redis cache. After that, the access token is generated with the new expiration time.

```go
package controllers

import (
	"fmt"
	"github.com/google/uuid"
	"github.com/icaksh/cripis/app/models"
	"github.com/icaksh/cripis/app/utils"
	"github.com/icaksh/cripis/platform/database"
	"github.com/thanhpk/randstr"
	"os"
	"strconv"
	"strings"
	"time"

	"github.com/gofiber/fiber/v2"
	"golang.org/x/crypto/bcrypt"
)

func RefreshToken(c *fiber.Ctx) error {
	body := models.RefreshToken{}
	err := c.BodyParser(&body)

	if err != nil {
		return utils.BadRequest(c, err)
	}

	validate := utils.NewValidator()
	if err := validate.Struct(&body); err != nil {
		return utils.BadRequest(c, err)
	}

	redisDb, err := database.RedisConnect()

	if err != nil {
		return utils.InternalServerError(c, err)
	}

	claims, err := utils.ExtractRefreshToken(body.RefreshToken)
	if err != nil {
		return c.Status(fiber.StatusUnauthorized).JSON(fiber.Map{
			"error":   true,
			"message": "Invalid token",
		})
	}

	deleted, err := redisDb.DeleteAuth(claims.RefreshUuid)

	if err != nil || deleted == 0 {
		return c.Status(fiber.StatusNotFound).JSON(fiber.Map{
			"error":   true,
			"message": "Invalid token",
		})
	}

	au := &models.JwtAuthModel{
		AccessId:  uuid.New(),
		RefreshId: uuid.New(),
		UserId:    claims.User,
		Email:     claims.Email,
		FirstName: claims.FirstName,
		LastName:  claims.LastName,
		Role:      claims.Role,
		Duration:  claims.ExpiresAt,
	}

	token, err := utils.GenerateNewAuthToken(au)
	if err != nil {
		return utils.InternalServerError(c, err)
	}

	refresh := redisDb.CreateAuth(claims.User, token)
	if refresh != nil {
		return utils.InternalServerError(c, err)
	}
	tokens := map[string]string{
		"access_token":  token.AccessToken,
		"refresh_token": token.RefreshToken,
	}

	return c.Status(fiber.StatusCreated).JSON(tokens)
}
```

We have finished building the logic we need to generate, validate, and refresh the token. Now we need to create the middleware to protect the route. We have 1 function: `JWTProtected`. The `JWTProtected` function is used to check if the access token is valid. The access token is sent in the request header, extracted, and then the user ID is fetched from the Redis cache. If the access token is valid, the user is authenticated. If the access token is invalid, the user is not authenticated.

```go
package middleware

import (
	"github.com/gofiber/fiber/v2"
	"github.com/icaksh/cripis/app/utils"
)

func JWTProtected(c *fiber.Ctx) error {
	err := utils.TokenValid(c)
	if err != nil {
		return c.Status(fiber.StatusUnauthorized).JSON(fiber.Map{
			"error": true,
			"msg":   err.Error(),
		})
	}
	c.Next()
	return nil
}
```

Last, in routes, add routes to refresh token and implement middleware to protect the route.

```go
package routes

import (
	"github.com/icaksh/cripis/app/controllers"
	"github.com/icaksh/cripis/pkg/middleware"
	recaptcha "github.com/jansvabik/fiber-recaptcha"

	"github.com/gofiber/fiber/v2"
)

func AuthRoutes(private fiber.Router, public fiber.Router) {
	pathName := "/auth"
	public.Post(pathName+"/login", recaptcha.Middleware, controllers.Login)
	public.Post(pathName+"/refresh", controllers.RefreshToken)
	public.Post(pathName+"/register", recaptcha.Middleware, controllers.Register)
	public.Post(pathName+"/reset", recaptcha.Middleware, controllers.Reset)
	//
	private.Post(pathName+"/logout", middleware.JWTProtected, controllers.Logout)
}

func UserRoutes(private fiber.Router, public fiber.Router) {
	pathName := "/user"
	private.Get(pathName+"s", middleware.JWTProtected, controllers.GetUsers)
	private.Get(pathName+"/", middleware.JWTProtected, controllers.GetUser)
	private.Get(pathName+"/:id", middleware.JWTProtected, controllers.GetUserById)
	private.Put(pathName+"/", middleware.JWTProtected, controllers.EditUser)
	private.Put(pathName+"/password", middleware.JWTProtected, controllers.EditUserPassword)
	private.Put(pathName+"/status", middleware.JWTProtected, controllers.EditUserStatus)
	private.Delete(pathName+"/:id", middleware.JWTProtected, controllers.DeleteUser)
	//
	public.Get(pathName+"/roles", controllers.GetUserRoles)
}
```

## Frontend

Handling token in frontend is easy. Just store the token in the local storage and configure the Axios to send the token in the header using interceptors when `request`. Here is the example code:

```javascript
axios.interceptors.request.use(
    (config) => {
        const user = JSON.parse(fastLocalStorage.getItem('user'));
        if (user && user.access_token) {
            config.headers.Authorization = `Bearer ${user.access_token}`;
        }
        return config;
    },
    (error) => {
        return Promise.reject(error);
    }
);
```

Now, the token is sent in the header when the request is sent. If the token is expired, the server will return a 401 status code. We need to handle this error. If the server returns a 401 status code, we need to send a request to the server to get a new access token using the refresh token and send the original request back. Here is the example code:

```javascript
axios.interceptors.response.use(
    (response) => {
        return response;
    },
    async (error) => {
        const originalRequest = error.config;
        if (error.response.status === 401 && !originalRequest._retry) {
            originalRequest._retry = true;
            try {
                const user = JSON.parse(fastLocalStorage.getItem('user'));
                const uninterceptedAxiosInstance = axios.create();
                const response = await uninterceptedAxiosInstance.post(process.env.VUE_APP_BACKEND_URL + '/public/auth/refresh', {
                    refresh_token: user.refresh_token,
                });
                fastLocalStorage.removeItem('user')
                fastLocalStorage.setItem('user', JSON.stringify(response.data));
                originalRequest.headers.Authorization = `Bearer ${response.data.access_token}`;
                return axios(originalRequest);
            } catch (refreshError) {
                Vue.$store.dispatch('logout');
            }
        }
        return Promise.reject(error);
    }
);
```

That's it! Now you have implemented JWT with Redis in Golang. This approach is secure and efficient. The token is stored in the Redis cache, so the user cannot access the token. The refresh token is used to get a new access token. The refresh token has a long expiration time, so the user does not need to re-login frequently. The refresh token is deleted from the Redis cache when the user logs out.

> [!NOTE]
> What if the refresh token is stolen? That is a good question. But I don't know exactly the answer. That's a risk we have to take. If the refresh token is stolen, the attacker can get a new access token. But the access token has a short expiration time, so the attacker cannot impersonate the user for a long time. The refresh token is deleted from the Redis cache when the user logs out. So, the risk is minimized.

For full implementation, you can check this repository

::github{repo="icaksh/cripis"}
::github{repo="icaksh/cripis-fe"}



`// END`