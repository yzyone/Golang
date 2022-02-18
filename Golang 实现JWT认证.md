# Golang 实现JWT认证 #

【导读】本文演示使用Golang实现基于JWT认证的示例应用。


认证是让应用知道给应用发送请求的人是他所说的那个人。JSON web token (JWT)是认证的一种方式，相比于基于Session认证，在系统中并不存储任何关于用户信息。

本文演示使用Golang实现基于JWT认证的示例应用。

## 1. JWT ##

**1.1. JWT格式**

假设用户user1尝试登录应用，成功后收到token信息如下：

	eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6InVzZXIxIiwiZXhwIjoxNTQ3OTc0MDgyfQ.2Ye5_w1z3zpD4dSGdRp3s98ZipCNQqmsHRB9vioOx54

这就是JWT，有三个部分组成，使用.分隔。

- 1). 第一部分是头部(eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9)。头部信息是生成签名的算法，这部分非常标准，对于任何使用相同算法的JWT都是一样的。
- 2). 第二部分是有效载荷(eyJ1c2VybmFtZSI6InVzZXIxIiwiZXhwIjoxNTQ3OTc0MDgyfQ)，它包含特定应用程序信息(在我们的示例中是用户名)，以及关于token有效期和有效性的信息。
- 3). 第三部分是签名(2Ye5_w1z3zpD4dSGdRp3s98ZipCNQqmsHRB9vioOx54)。它是通过将前两部分与一个密钥合并和散列而生成的。

有趣的是头部和有效载荷是不加密的。仅使用base64进行编码，意味着任何人都能解码看到内容。使用下面命令(linux)可以看到：

	echo eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9 | base64 -d

返回：

	{ "alg": "HS256", "typ": "JWT" }

类似方式可以看到第二部分内容：

	{ "username": "user1", "exp": 1547974082 }

**1.2. JWT签名**

既然JWT前两部分任何人都可以读写，JWT安全由什么来保证呢？答案是最后部分（签名）是如何生成的。

假设应用程序希望向成功登录的用户(例如user1)发出JWT。
头部和有效负载非常简单:报头或多或少是固定的，而有效负载JSON对象是通过设置用户ID和终止时间(以unix毫秒为单位)形成的。

发出token的应用程序也有一个密钥，只有应用程序本身知道。应用程序将报头和有效负载的base64表示形式与密钥组合，然后通过一个散列算法(在本例中是HS256，报头中指定的)进行传递。

![](./jwt/640.webp)

算法是如何实现的细节超出了本文的范围,但要注意的是,这是一个方式,这意味着我们不能反向算法和获得的签名并使用…所以我们的密钥需保护好。

**1.3. 验证JWT**

为了验证请求的JWT,再次使用请求的报头和有效载荷以及密钥生成签名。如果签名匹配那么JWT为有效。

假设一个试图发出假token黑客，他可以轻松地生成报头和有效负载，但是如果不知道密钥，就无法生成有效的签名。如果试图篡改有效JWT的现有有效负载，签名将不再匹配。
![](./jwt/641.webp)

通过这种方式JWT作为一种安全方式认证用户，在应用服务器上无需存储任何信息(除了密钥)。

## 2. Golang实现示例 ##

现在我们已经了解JWT的认证机制，下面在GoLang中实现。

2.1. 创建HTTP服务器
首先初始化HTTP服务及必要的路由：

```
package main

import (
 "log"
 "net/http"
)

func main() {
 // "Signin" and "Welcome" are the handlers that we will implement
 http.HandleFunc("/signin", Signin)
 http.HandleFunc("/welcome", Welcome)
 http.HandleFunc("/refresh", Refresh)

 // start the server on port 8000
 log.Fatal(http.ListenAndServe(":8000", nil))
}
```

下面实现Signin和Welcome两个路由。

**2.2. 处理用户登录**

/signin路由使用用户凭证登录，为了简化我们在内存中存储用户信息：

```
// Create the JWT key used to create the signature
var jwtKey = []byte("my_secret_key")

var users = map[string]string{
 "user1": "password1",
 "user2": "password2",
}

// Create a struct to read the username and password from the request body
type Credentials struct {
 Password string `json:"password"`
 Username string `json:"username"`
}

// Create a struct that will be encoded to a JWT.
// We add jwt.StandardClaims as an embedded type, to provide fields like expiry time
type Claims struct {
 Username string `json:"username"`
 jwt.StandardClaims
}

// Create the Signin handler
func Signin(w http.ResponseWriter, r *http.Request) {
 var creds Credentials
 // Get the JSON body and decode into credentials
 err := json.NewDecoder(r.Body).Decode(&creds)
 if err != nil {
  // If the structure of the body is wrong, return an HTTP error
  w.WriteHeader(http.StatusBadRequest)
  return
 }

 // Get the expected password from our in memory map
 expectedPassword, ok := users[creds.Username]

 // If a password exists for the given user
 // AND, if it is the same as the password we received, the we can move ahead
 // if NOT, then we return an "Unauthorized" status
 if !ok || expectedPassword != creds.Password {
  w.WriteHeader(http.StatusUnauthorized)
  return
 }

 // Declare the expiration time of the token
 // here, we have kept it as 5 minutes
 expirationTime := time.Now().Add(5 * time.Minute)
 // Create the JWT claims, which includes the username and expiry time
 claims := &Claims{
  Username: creds.Username,
  StandardClaims: jwt.StandardClaims{
   // In JWT, the expiry time is expressed as unix milliseconds
   ExpiresAt: expirationTime.Unix(),
  },
 }

 // Declare the token with the algorithm used for signing, and the claims
 token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
 // Create the JWT string
 tokenString, err := token.SignedString(jwtKey)
 if err != nil {
  // If there is an error in creating the JWT return an internal server error
  w.WriteHeader(http.StatusInternalServerError)
  return
 }

 // Finally, we set the client cookie for "token" as the JWT we just generated
 // we also set an expiry time which is the same as the token itself
 http.SetCookie(w, &http.Cookie{
  Name:    "token",
  Value:   tokenString,
  Expires: expirationTime,
 })
}
```

如果用户登录凭证正确，该处理器在客户通过cookie设置JWT。客户端有了cookie，后续每个请求将携带cookie。下面我们实现Welcome处理器。

**2.3. 处理后续认证**

登录成功用户在客户端存储了会话信息，可以使用会话信息进行：

- 认证后续用户请求
- 获取用户请求的信息
- 
Welcome处理器实现：

```
func Welcome(w http.ResponseWriter, r *http.Request) {
 // We can obtain the session token from the requests cookies, which come with every request
 c, err := r.Cookie("token")
 if err != nil {
  if err == http.ErrNoCookie {
   // If the cookie is not set, return an unauthorized status
   w.WriteHeader(http.StatusUnauthorized)
   return
  }
  // For any other type of error, return a bad request status
  w.WriteHeader(http.StatusBadRequest)
  return
 }

 // Get the JWT string from the cookie
 tknStr := c.Value

 // Initialize a new instance of `Claims`
 claims := &Claims{}

 // Parse the JWT string and store the result in `claims`.
 // Note that we are passing the key in this method as well. This method will return an error
 // if the token is invalid (if it has expired according to the expiry time we set on sign in),
 // or if the signature does not match
 tkn, err := jwt.ParseWithClaims(tknStr, claims, func(token *jwt.Token) (interface{}, error) {
  return jwtKey, nil
 })
 if err != nil {
  if err == jwt.ErrSignatureInvalid {
   w.WriteHeader(http.StatusUnauthorized)
   return
  }
  w.WriteHeader(http.StatusBadRequest)
  return
 }
 if !tkn.Valid {
  w.WriteHeader(http.StatusUnauthorized)
  return
 }

 // Finally, return the welcome message to the user, along with their
 // username given in the token
 w.Write([]byte(fmt.Sprintf("Welcome %s!", claims.Username)))
}
```

**2.4. 更新token**

该示例中，有效时间为5分钟。用户当然不希望每5分钟登录一次。因此我们实现另一个路由/refresh，使用之前的token(仍然有效)获取带有更新时间的token。

为了尽量少使用JWT，过期时间通常保持在几分钟左右。通常，客户机应用程序将在后台刷新令牌。

```
func Refresh(w http.ResponseWriter, r *http.Request) {
 // (BEGIN) The code uptil this point is the same as the first part of the `Welcome` route
 c, err := r.Cookie("token")
 if err != nil {
  if err == http.ErrNoCookie {
   w.WriteHeader(http.StatusUnauthorized)
   return
  }
  w.WriteHeader(http.StatusBadRequest)
  return
 }
 tknStr := c.Value
 claims := &Claims{}
 tkn, err := jwt.ParseWithClaims(tknStr, claims, func(token *jwt.Token) (interface{}, error) {
  return jwtKey, nil
 })
 if err != nil {
  if err == jwt.ErrSignatureInvalid {
   w.WriteHeader(http.StatusUnauthorized)
   return
  }
  w.WriteHeader(http.StatusBadRequest)
  return
 }
 if !tkn.Valid {
  w.WriteHeader(http.StatusUnauthorized)
  return
 }
 // (END) The code up-till this point is the same as the first part of the `Welcome` route

 // We ensure that a new token is not issued until enough time has elapsed
 // In this case, a new token will only be issued if the old token is within
 // 30 seconds of expiry. Otherwise, return a bad request status
 if time.Unix(claims.ExpiresAt, 0).Sub(time.Now()) > 30*time.Second {
  w.WriteHeader(http.StatusBadRequest)
  return
 }

 // Now, create a new token for the current use, with a renewed expiration time
 expirationTime := time.Now().Add(5 * time.Minute)
 claims.ExpiresAt = expirationTime.Unix()
 token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
 tokenString, err := token.SignedString(jwtKey)
 if err != nil {
  w.WriteHeader(http.StatusInternalServerError)
  return
 }

 // Set the new token as the users `token` cookie
 http.SetCookie(w, &http.Cookie{
  Name:    "token",
  Value:   tokenString,
  Expires: expirationTime,
 })
}
```

## 3. 运行应用 ##

编译并运行程序：

    go build
    ./jwtDemo

现在使用支持cookie的HTTP客户端工具进行测试，如POSTMAN,使用凭证进行登录：

	POST http://localhost:8000/signin

	{"username":"user1","password":"password1"}

下面可以使用同样的工具发送welcome请求获取返回信息：

	GET http://localhost:8000/welcome

发送更新token请求，然后检查客户端cookie及token新的值：

	POST http://localhost:8000/refresh

## 4. 总结 ##

本文介绍了JWT认证机制，通过示例使用Golang进行演示。



> 转自：
> 
> blog.csdn.net/neweastsun/article/details/105919915