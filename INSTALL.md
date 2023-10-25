# **Installation**

* Installation
  - [Server Requirements](#server-requirements)
  - [Installing Lumen](#installing-lumen)
  - [Configuration](#configuration)

### __Installation__

#### _Server Requirements__
* PHP >= 7.2
* OpenSSL PHP Extension
* PDO PHP Extension
* Mbstring PHP Extension

#### __Installing Lumen__
```sh
composer create-project --prefer-dist laravel/lumen {project-name}
```

#### __Serving Your Application__
```sh
php -S localhost:8000 -t public
```

#### __Configuration__
* Lumen의 모든 구성 옵션은 ```.env``` 파일에 저장됩니다.

##### __Application Key__
* Lumen 설치 후 application key를 임의의 문자열로 설정 하여야 합니다.
* 문자열의 길이는 32자여야 하며, ```.env``` 파일에서 설정할 수 있습니다.
* **application key**가 설정되지 않으면 사용자가 암호화한 데이터가 안전하지 않습니다.
