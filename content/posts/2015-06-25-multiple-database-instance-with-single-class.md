---
title: Multiple database instances with single class
date: 2015-06-25
categories:
---

During the development of an API for a client i feel the need to have one class that can return me instance of different databases. I made a singleton pattern to achieve this. Below is the class that you can utilize if needed.

```php
<?php

class Connection {

    /**
     * [$_mongo_connection description]
     * @var null
     */
    private static $_mongo_connection = null;

    /**
     * [$_mysql_connection description]
     * @var null
     */
    private static $_mysql_connection = null;

    /**
     * A private construction to achieve singleton
     * @param string $type type of database
     */
    private function __construct($type)
    {
        switch ($type)
        {
            case 'Mongo':
                self::$_mongo_connection = new MongoClient('mongodb://localhost');
                break;

            case 'MySQL':
                $dsn = 'mysql:dbname=mydatabase;host=localhost';
                $user = 'root';
                $password = 'mypassword';
                self::$_mysql_connection  = new PDO($dsn, $user, $password);
                break;

            default:
                return;
                break;
        }
    }

    /**
     * Returns instance of MongoDB
     * @return  Resource
     */
    public static function getMongoInstance()
    {
        if(is_null(self::$_mongo_connection))
        {
            new Connection('Mongo');
        }
        return self::$_mongo_connection;
    }

    /**
     * Returns instance of MongoDB
     * @return  Resource
     */
    public static function getMySQLInstance()
    {
        if(is_null(self::$_mysql_connection))
        {
            new Connection('MySQL');
        }
        return self::$_mysql_connection;
    }

    /**
     * Set the instannces to null on object disposal
     */
    public function __destruct()
    {
        self::$_mongo_connection = null;
        self::$_mysql_connection = null;
    }
}
```
