+++
title = "Convert JSON to POPO"
date = "2020-05-09T13:37:00+04:00"
images = []
tags = ["PHP"]
keywords = []
description = ""
showFullContent = false
readingTime = true
hideComments = false
+++

Consider that you have `example.json` with the following content:

```json
{
  "firstName": "Temuri",
  "lastName": "Takalandze",
  "active": true,
  "position": {
    "title": "Developer",
    "department": {
      "title": "IT"
    }
  }
}
```

<!--more-->

And several POPO classes to represent this JSON data:

**Department.php**
```php

<?php

class Department
{
    /**
     * @var string
     */
    private $title;

    // Getters and Setters here...
}
```

**Position.php**
```php
<?php

class Position
{
    /**
     * @var string
     */
    private $title;

    /**
     * @var \ABGEO\POPO\Example\Department
     */
    private $department;

    // Getters and Setters here...
}
```

**Person.php**
```php
<?php

class Person
{
    /**
     * @var string
     */
    private $firstName;

    /**
     * @var string
     */
    private $lastName;

    /**
     * @var bool
     */
    private $active;

    /**
     * @var \ABGEO\POPO\Example\Position
     */
    private $position;

    // Getters and Setters here...
}
```

Now you want to convert this JSON to POPO with relations. My [ABGEO/json-to-popo](https://github.com/ABGEO/json-to-popo) package gives you this ability. Install it using [Composer](https://getcomposer.org/):

```shell
composer require abgeo/json-to-popo
```

Now let's create new `ABGEO\POPO\Composer` object and read `example.json` content:

```php
$composer = new Composer();
$jsonContent = file_get_contents(__DIR__ . '/example.json');
```

Time for magic! Call `composeObject()` with the contents of JSON and the main class, and this will give you POPO:

```php
$resultObject = $composer->composeObject($jsonContent, Person::class);
```

Print `$resultObject`:

```php
var_dump($resultObject);

//class ABGEO\POPO\Example\Person#2 (4) {
//  private $firstName =>
//  string(6) "Temuri"
//  private $lastName =>
//  string(10) "Takalandze"
//  private $active =>
//  bool(true)
//  private $position =>
//  class ABGEO\POPO\Example\Position#4 (2) {
//    private $title =>
//    string(9) "Developer"
//    private $department =>
//    class ABGEO\POPO\Example\Department#7 (1) {
//      private $title =>
//      string(2) "IT"
//    }
//  }
//}
```

- [json-to-popo on GitHub](https://github.com/ABGEO/json-to-popo)
- [json-to-popo on Packagist](https://packagist.org/packages/abgeo/json-to-popo)
