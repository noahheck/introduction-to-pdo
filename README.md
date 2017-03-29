# Introduction to PDO

> The PHP Data Objects (PDO) extension defines a lightweight, consistent interface for accessing databases in PHP. Each database driver that implements the PDO interface can expose database-specific features as regular extension functions. Note that you cannot perform any database functions using the PDO extension by itself; you must use a database-specific PDO driver to access a database server.
>
> PDO provides a data-access abstraction layer, which means that, regardless of which database you're using, you use the same functions to issue queries and fetch data. PDO does not provide a database abstraction; it doesn't rewrite SQL or emulate missing features. You should use a full-blown abstraction layer if you need that facility.
>
> PDO ships with PHP 5.1, and is available as a PECL extension for PHP 5.0; PDO requires the new OO features in the core of PHP 5, and so will not run with earlier versions of PHP.
>
> -- http://php.net/manual/en/intro.pdo.php



### Creating PDO Objects

The constructor for the `PDO` class has the following arguments:

- `$dsn` - Data Source Name - database connection information (will be specific to the database driver)
- `$username` - database user
- `$password` - database user password
- `$options` - array of driver-specific connection options

*PHP does allow specifying default values for these parameters in `php.ini`. For more information on the `PDO` class, see http://php.net/pdo*

The following demonstrates how to create a PDO connection to the pdotest schema on a MySQL server on the localhost:

```php
<?php
$dsn      = "mysql:dbname=pdotest;host=localhost";
$username = "dbuser";
$password = "dbpass";

$pdo = new PDO($dsn, $username, $password);
```


### Database Queries
Database interactions using PDO are provided through the use of PDOStatement objects. A PDOStatement object represents both a prepared database statement and, after the statement is executed, the result set of that statement.
Generally, you will gain access to a PDOStatement object when you use the PDO object to prepare a SQL statement for execution:

```php
<?php
$query = “SELECT  name, email
            FROM  users
            WHERE active = 1”;
$stmt = $pdo->prepare($query);
```

In the example, `$stmt` is a `PDOStatement` object that, when executed, will query information from the misc_vars table. To execute the statement, use the execute method of the `PDOStatement` object:

```php
<?php
$stmt->execute();
```

The result set of the query statement is stored within the `PDOStatement` object. This data can be accessed using the `PDOStatement` `fetch*` family of methods. There are many ways to access the result set:

##### List-ing values

Values can be `list`-ed into scope level variables using the PHP `list` construct:

```php
<?php
while (list($name, $email) = $stmt->fetch()) {
    // Application logic here – code will execute once for each resultant row
}
```

If the result set will only have one record, the while construct is not necessary:

```php
list ($name, $email) = $stmt->fetch();
```

##### Row-level assignment

An entire result row can be assigned to a variable with the values of the result set assigned as values of that variable. The variable type that attains the values of the result can be modified. By default, fetch returns an array indexed by both column name and 0-indexed column number as returned in your result set:

```php
<?php
while ($row = $stmt->fetch()) {
    // Application logic here – code will execute once for each resultant row
    /*
        $row = array(
            'name'  => 'John Doe',
            '0'     => 'John Doe',
            'email' => 'john.doe@example.com',
            '1'     => 'john.doe@example.com',
        );
    */
}
```

For performance reasons, it may be a better idea to have the result set returned in only one of the available formats. You specify which style you would like to retrieve the result set in as the first argument to the fetch method (the default fetch style [above] is `PDO::FETCH_BOTH`). Common fetch styles are described here.

##### PDO::FETCH_NUM

Fetches the result as a 0-indexed array of columns

```php
<?php
while ($row = $stmt->fetch(PDO::FETCH_NUM)) {
    /*
        $row = array(
            0 => 'John Doe',
            1 => 'john.doe@example.com',
        );
    */
}
```

##### PDO::FETCH_ASSOC

Fetches the result as an associative array with column names as array keys

```php
<?php
while ($row = $stmt->fetch(PDO::FETCH_ASSOC)) {
    /*
        $row = array(
            'name'  => 'John Doe',
            'email' => 'john.doe@example.com',
        );
    */
}
```

##### PDO::FETCH_OBJ

Fetches the result as an anonymous object with column names as properties

```php
<?php
while ($row = $stmt->fetch(PDO::FETCH_OBJ)) {
    /*
        $row->name  = 'John Doe';
        $row->email = 'john.doe@example.com';
    */
}
```

##### Returning all result rows

The `PDOStatement` object also supports a `fetchAll` method which will return an array containing all of the result set rows. The same fetch style options can be provided the the `fetchAll` method (the same `PDO::FETCH_BOTH` default applies as well):

```php
<?php
$rows = $stmt->fetchAll(PDO::FETCH_ASSOC);
/*
    $rows = array(
        0 => array(
            'name'  => 'John Doe',
            'email' => 'john.doe@example.com',
        ),
        1 => array(
            'name'  => 'Jane Doe',
            'email' => 'jane.doe@example.com',
        )
    );
*/
```

### Prepared Statements and Parameter Binding

The PDO extension supports some advanced database concepts, including the use of prepared statements and parameter binding. This functionality helps ensure more consistent application performance and helps protect against SQL injection attacks.

Consider the following:

```php
<?php
$fname = $_POST['fname'];
$cid   = $_POST['cid'];
$query = “UPDATE contacts SET FNAME = '$fname' WHERE CID = $cid”;
mysql_query($query);
```

This potentially problematic query can be exploited by a user intentionally (or otherwise) submitting an inappropriate character within either the posted fname or cid value:

```php
<?php
$fname = $_POST['fname']; // '; DROP TABLE contacts;
$cid   = $_POST['cid']; // 12; DROP TABLE users;
```

The query that is executed could then drop a database table, or, even worse, grant access to the user to any and all information within the database. PDO uses parameterized queries to close this potential security hole.

In order to use parameterized queries, rather than placing the data to be placed into the database into the query, PDO allows placeholders to be placed into the query that represent where a value should be. A variable containing that value is then bound to that placeholder. When the query is executed against the database, the database (rather than the query) is responsible for placing the value in the correct location. The database recognizes each piece of data distinctly, so the possibility of the situation presented above is negated.

#### Binding Parameters

There are 2 different types of placeholders available for parameter binding: named placeholders and question mark (?) placeholders. Both provide the same security benefits, but their interactions with the PDOStatement are distinct.

**You cannot combine the two different types of placeholders in the same query.**

##### Question Mark Placeholders

Question mark placeholders are placed into the query at the point where a value should take it's place. After the query is prepared, the values that represent the placeholder information are provided as an input argument (array) to the execute method. The values are then sent to the database server to be interpreted there:

```php
<?php
$fname  = $_POST['fname'];
$cid    = $_POST['cid'];
$values = array($fname, $cid);

$query  = “UPDATE contacts SET FNAME = ? WHERE CID = ?”;
$stmt   = $pdo->prepare($query);

$stmt->execute($values);
```

Note that the location of the value in the input array needs to correspond to the location in the query of it's associated placeholder, e.g. the first element in the array will correspond to the first placeholder, etc.

##### Named Placeholders

Named placeholders are identified by their leading colon (:), and are referenced by their name when the binding occurs. The binding is done using the `bind*` methods of the `PDOStatement` object:

```php
<?php
$fname  = $_POST['fname'];
$cid    = $_POST['cid'];
$query  = “UPDATE contacts SET FNAME = :fname WHERE CID = :cid”;
$stmt   = $pdo->prepare($query);

$stmt->bindValue(“:fname”, $fname, PDO::PARAM_STR);
$stmt->bindValue(“:cid”, $cid, PDO::PARAM_INT);

$stmt->execute();
```

When the statement is executed, the query and the bound values are sent to the database server to be executed there.

The first argument to the bind* methods is the named placeholder to which you are binding. The second argument is the value you want to bind to that placeholder. The optional third argument is an integer value of either `PDO::PARAM_STR` (default) or `PDO::PARAM_INT`. This represents whether the value should be recognized by the database server as a string or an integer.

If the third argument is omitted, the default value for this is `PDO::PARAM_STR`.

*If binding to non-integer numeric columns (e.g. `DECIMAL`), use `PDO::PARAM_STR` as the datatype.*

The functionality of `bindValue` is to accept the provided value as it exists when the placeholder is bound. This means, for example, that if the value of the variable changes before the statement is executed, the bound value will not change with the variable:

```php
<?php
$dbValues = array(
    “fname”   => “John”,
    “cid”     => 726,
);
$query = “UPDATE contacts SET FNAME = :fname WHERE CID = :cid”;
$stmt  = $pdo->prepare($query);

foreach ($dbValues as $key => $value) {
    $placeholder = “:” . $key;
    $stmt->bindValue($placeholder, $value, PDO::PARAM_STR);
}
$stmt->execute();// UPDATE contacts SET FNAME = 'John' WHERE CID = '726'
```

By contrast, if a value is bound to the statement using the `bindParam` function, the data that will be used in place of the bound placeholder will be the current value of that variable when the statement is executed. This allows the same prepared statement to be executed multiple times using different data points.

```php
<?php
$query  = “UPDATE contacts SET FNAME = :fname WHERE CID = :cid”;
$stmt   = $pdo->prepare($query);

$stmt->bindParam(“:fname”, $fname, PDO::PARAM_STR);
$stmt->bindParam(“:cid”, $cid, PDO::PARAM_INT);

$fname  = “John”;
$cid        = 726;
$stmt->execute(); // UPDATE contacts SET FNAME = 'John' WHERE CID = 726

$fname  = “Jack”;
$cid        = 727;
$stmt->execute(); // UPDATE contacts SET FNAME = 'Jack' WHERE CID = 727
```

If a process has the potential for exploiting the functionality provided by `bindParam`, that process should be taken. When the query statement is prepared by the database, the database will analyze the query, compile it into components that it can act against, then optimize the execution of that statement within the schema. When that only has to be completed one time but the query can be executed with different values multiple times, fewer resources are consumed (meaning faster execution).

Read more about prepared statements at: http://php.net/manual/en/pdo.prepared-statements.php

### E_PDOStatement

When an issue is encountered with a database query, most developers are used to echo-ing the compiled query string and examining the result to aid in determining where the issue has occurred. Because PDO doesn't simply sanitize the parameters and inject them into the query (instead an emulation of that process is completed by the database), developers aren't able to view an interpretation of the query.

[noahheck/E_PDOStatement](https://github.com/noahheck/E_PDOStatement) is an open source project that aims to provide this functionality:

```php
<?php
$cid    = 726;
$query  = “SELECT * FROM contacts WHERE CID = :cid”;
$stmt   = $pdo->prepare($query);
$stmt->bindParam(“:cid”, $cid, PDO::PARAM_INT);
$stmt->execute();
$queryString = $stmt->fullQuery;
echo $queryString; // SELECT * FROM contacts WHERE CID = 726
```

The `E_PDOStatement` library provides two means for viewing it's approximation of the query being executed: either as the `fullQuery` property of the `E_PDOStatement` object after the query has been executed (useful, for example, for log files), and as the return value of the `interpolateQuery` method, which can be invoked before or after the statement is executed.

Read more about noahheck/E_PDOStatement at: https://github.com/noahheck/E_PDOStatement

---

This work is proudly licensed under the terms of the [Creative Commons Zero v1.0 Universal](https://choosealicense.com/licenses/cc0-1.0/#) license. All contributions to this project are understood to be made in accordance with the terms therein.
