# **JWT Installation**

Link: [jwt-auth][jwtauth-link]

[jwtauth-link]: https://jwt-auth.readthedocs.io/en/develop/lumen-installation/ "jwt-auth"

* Installation
  - [Install via composer](#install-via-composer)
  - [Copy the config](#copy-the-config)
  - [Bootstrap file changes](#bootstrap-file-changes)
  - [Generate secret key](#generate-secret-key)
  - [Update your User model](#update-your-user-model)
  - [Configure Auth guard](#configure-auth-guard)
  - [Add some basic authentication routes](#add-some-basic-authentication-routes)
  - [Create the AuthController](#create-the-authcontroller)
  - [Authenticated requests](#authenticated-requests)

### __Installation__

#### __Install via composer__
```sh
composer require tymon/jwt-auth
```

#### __Copy the config__
* ```vendor/tymon/jwt-auth/config/config.phpconfigjwt.php``` 파일을 복사한 후 config 폴더에 ```jwt.php``` 로 저장합니다.
* ```bootstrap/app.php``` 파일에서 미들웨어 선언 이전에 다음을 추가합니다.
```php
$app->configure('jwt');
```


#### __Bootstrap file changes__
* ```bootstrap/app.php``` 파일에 다음을 추가합니다.
```php
// Register Middleware
// 주석 해제
$app->routeMiddleware([
  'auth' => App\Http\Middleware\Authenticate::class,
]);

// Register Service Providers
// 주석 해제
$app->register(App\Providers\AuthServiceProvider::class);
// 추가
$app->register(Tymon\JWTAuth\Providers\LumenServiceProvider::class);
```

#### __Generate secret key__
* 키를 생성하는 명령어
```sh
php artisan jwt:secret
```
* 생성된 key는 ```.env``` 파일에 다음과 같이 업데이트 됩니다. ```JWT_SECRET={key}```
* Token 서명에 사용되는 키 입니다.


#### __Update your User model__
* 우선 User modal에 ```Tymon\JWTAuth\Contracts\JWTSubject``` contract을 구현해야 합니다. 이를 위해 다음 두 메소드를 구현해야 합니다.
* ```getJWTIdentifier()```, ```getJWTCustomClaims()```
```php
<?php

namespace App;

use Tymon\JWTAuth\Contracts\JWTSubject;
use Illuminate\Notifications\Notifiable;
use Illuminate\Foundation\Auth\User as Authenticatable;

class User extends Authenticatable implements JWTSubject
{
  use Notifiable;

  // Rest omitted for brevity

  /**
   * Get the identifier that will be stored in the subject claim of the JWT.
   *
   * @return mixed
   */
  public function getJWTIdentifier()
  {
    return $this->getKey();
  }

  /**
   * Return a key value array, containing any custom claims to be added to the JWT.
   *
   * @return array
   */
  public function getJWTCustomClaims()
  {
    return [];
  }
}
```

#### __Configure Auth guard__
* Laravel 5.2 이상일 때만 작동합니다.
* ```config/auth.php``` 파일 내에서 인증을 강화하기 위해 ```jwt``` guard를 사용하도록 변경사항  적용
```php
<?php

return [
  'defaults' => [
    'guard' => 'api',
    'passwords' => 'users',
  ],
  'guards' => [
    'api' => [
      'driver' => 'jwt',
      'provider' => 'users',
    ],
  ],
  'providers' => [
    'users' => [
      'driver' => 'eloquent',
      'model' => \App\User::class
    ]
  ]
];
```
* 여기서는 ```api``` 가드에게 ```jwt``` 드라이버를 사용하라고 지시하고 있으며 ```api``` 가드를 기본값으로 설정하고 있습니다.
* 이제 jwt-auth가 뒤에서 작업을 수행하는 Laravel의 내장 인증 시스템을 사용할 수 있습니다.


#### __Add some basic authentication routes__
* ```routes/web.php```에 경로 추가
```php
$router->group(['prefix' => 'api'], function () use ($router) {
  $router->post('login', 'AuthController@login');
  $router->post('logout', 'AuthController@logout');
  $router->post('refresh', 'AuthController@refresh');
  $router->post('me', 'AuthController@me');
});
```


#### __Create the AuthController__
*  ```app/http/Controllers/AuthController.php``` 파일 생성 후 아래 내용을 작성합니다.
```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;

class AuthController extends Controller
{
    public function __construct()
    {
        $this->middleware('auth:api', ['except' => ['login', 'refresh', 'logout']]);
    }

    /**
     * Get a JWT via given credentials.
     *
     * @param  Request  $request
     * @return Response
     */
    public function login(Request $request)
    {
        $this->validate($request, [
            'email' => 'required|string',
            'password' => 'required|string',
        ]);

        $credentials = $request->only(['email', 'password']);

        if (! $token = Auth::attempt($credentials)) {
            return response()->json(['message' => 'Invalid credentials'], 401);
        }

        return $this->jsonResponse($token);
    }

    /**
     * Get the authenticated User.
     *
     * @return \Illuminate\Http\JsonResponse
     */
    public function me()
    {
        return response()->json(auth()->user());
    }

    /**
     * Log the user out (Invalidate the token).
     *
     * @return \Illuminate\Http\JsonResponse
     */
    public function logout()
    {
        auth()->logout();

        return response()->json(['message' => 'Successfully logged out']);
    }

    /**
     * Refresh a token.
     *
     * @return \Illuminate\Http\JsonResponse
     */
    public function refresh()
    {
        return $this->jsonResponse(auth()->refresh());
    }

    /**
     * Get the token array structure.
     *
     * @param  string $token
     *
     * @return \Illuminate\Http\JsonResponse
     */
    protected function jsonResponse($token)
    {
        return response()->json([
            'access_token' => $token,
            'token_type' => 'bearer',
            'expires_in' => auth()->factory()->getTTL() * 60 * 24
        ]);
    }
}
```
* POST ```http://localhost:8000/api/login``` 수행 시 아래와 같은 결과를 볼 수 있습니다.
```sh
{
    "access_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJodHRwOlwvXC9sb2NhbGhvc3Q6ODAwMFwvYXBpXC9yZWZyZXNoIiwiaWF0IjoxNjk4MzAzMDQ1LCJleHAiOjE2OTgzMDY2ODMsIm5iZiI6MTY5ODMwMzA4MywianRpIjoiRXpUNXlIRDV2TjQwSGV6RCIsInN1YiI6MSwicHJ2IjoiODdlMGFmMWVmOWZkMTU4MTJmZGVjOTcxNTNhMTRlMGIwNDc1NDZhYSJ9.KJRNCJG1tA9WvLjJym8K62wQzjPljVFva1ksziEkl0I",
    "token_type": "bearer",
    "expires_in": 86400
}
```


#### __Authenticated requests__
* 발급된 Token을 보내는 방법

```sh
// Authorization header
Authorization: Bearer eyJ0eXAiOiJKV1Q...
```

```sh
// Query string parameter
http://localhost:8000?token=eyJ0eXAiOiJKV1Q...
```



[TOP▲](#jwt-installation)
