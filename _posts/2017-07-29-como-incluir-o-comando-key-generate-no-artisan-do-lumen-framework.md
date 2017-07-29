---
layout: post
title: "Como incluir o comando key:generate no artisan do Lumen Framework"
date: 2017-07-29 13:08:00
description: 'Esta semana resolvi brincar um pouco com o Lumen para entender na prática suas diferenças em relação ao Laravel.  
Logo de inicio seguindo a documentação de como instalar ele ao final dizia para configurar a key da aplicação no arquivo .env, no Laravel temos o comando artisan key:generate, mas no Lumen não, eis que fui atrás desta informação para saber como iria gerar essa key.'
tags:
- php
- lumen
categories:
- PHP
image: '/assets/img/'
---
Esta semana resolvi brincar um pouco com o Lumen para entender na prática suas diferenças em relação ao Laravel.  
Logo de inicio seguindo a documentação de como instalar ele ao final dizia para configurar a key da aplicação no arquivo .env, no Laravel temos o comando artisan key:generate, mas no Lumen não, eis que fui atrás desta informação para saber como iria gerar essa key.  
Uma forma simples que encontrei foi através do php-cli rodar um str_random(32), copiar o resultado e colocar lá no .env, porém encontrei também uma forma de colocar como comando no artisan, deixando assim parecido com o Laravel, e vou explicar como fazer isso neste post!  

Primeiro, você precisa registrar seu comando de key generator, coloque [este Lumen Key Generator Command](https://gist.github.com/krisanalfa/0407dd822f2888226f45) em app/Console/Commands/KeyGenerateCommand.php.  
Para deixar este comando disponivel no artisan, altere o arquivo app\Console\Kernel.php como abaixo:
```php
<?php
/**
 * The Artisan commands provided by your application.
 *
 * @var array
 */
protected $commands = [
    'App\Console\Commands\KeyGenerateCommand',
];
...
```



Depois disso, configure sua aplicação para que a instância Illuminate\Config\Repository tenha o valor de app.key. Para fazer isso altere o arquivo bootstrap/app.php como abaixo:
```php
<?php

require_once __DIR__.'/../vendor/autoload.php';

Dotenv::load(__DIR__.'/../');

/*
|--------------------------------------------------------------------------
| Create The Application
|--------------------------------------------------------------------------
|
| Here we will load the environment and create the application instance
| that serves as the central piece of this framework. We'll use this
| application as an "IoC" container and router for this framework.
|
*/

$app = new Laravel\Lumen\Application(
    realpath(__DIR__.'/../')
);

$app->configure('app');
...
```



Depois disso, copie seu arquivo .env.example para .env
```
cp .env.example .env
```
**Ignore este passo se você já usa um arquivo .env.**



Agora aproveite seu comando key:generate através de:
```
php artisan key:generate
```

Até mais!