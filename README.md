# WireApi-PoC
Prototype for a ProcessWire API module for routing and API endpoints

This is a proof-of-concept implementation for a somewhat unified API endpoint class in ProcessWire.

## Status

alpha - just a prototype to play around run ideas against!

Don't even think of using this in production! The WireApi name may become a reserved class in
PW core at any time, and it will likely be something completely different from this!

## Goals
- Provide a neat routing syntax as concise as popular node.js routers or e.g. Laravel Routing
- Stick with familiar PW syntax / idioms
- Make use of builtin $sanitizer to validate route URLs and part values
- Allow handling of routes through functions, methods and PHP files
- Ease creation of API endpoints


## A Bit of Prose
WireApi registers itself as $api API variable and thus is available as either $api,
wire('api') or $this->api inside Wire derived classes.

Routes all end in a custom page that calls $api->handleRequest();

You can theoretically have multiple API endpoints by creating multiple pages with
their own template.

A minimal template (e.g. /site/templates/api.php) would look like this:
```php
<?php namespace ProcessWire;

$api->handleRequest();
```

## Endpoints

You can have any number of API endpoints. Behind the scene, they are just pages that
call ```$api->handleRequest()```.

To make this even easier, this module ships with ProcessWireApi which adds itself
to the backend under Setup -> API.

To create an API endpoint, just enter the desired page name (always under /home),
and optionally the template name and/or template file name if they differ.

ProcessWireApi will create the template, page and, if the server has write access
to site/templates, also the template file. If the web server doesn#t have write
access, you can download the template from the API creator interface or copy the
api-template.php from the modules directory there and rename it.

![API endpoint creation](https://raw.githubusercontent.com/BitPoet/bitpoet.github.io/master/img/ProcessWireApi_1.png)


## Routes
Routes can be attached from almost anywhere, i.e. inside modules, site/ready.php
or the endpoint template file itself.

At a minimum, a route definition needs the HTTP verb(s) it handles (or 'ALL' as a
wildcard) and a handler, which can be a plain function, a class method or any valid
callable definition accepted by PHP, or the path to a PHP file, which lets you build
your own 'API template folders', e.g. /site/templates/apitemplates/.

### Placeholders

Like almost all routing frameworks, you can use placeholders in your route paths
where dynamic values can occur and assign names to those, e.g.
```/api/user/{userid}```

The names then can be referenced in your checks.

### Handler Output

You can just return output like in any template.

There is a shorthand to output JSON data though, namely

```php
$api->jsonResponse($json);
```

This sets the content type to application/json and outputs the data in JSON notation.
If $json is already a string, no JSON conversion will be done.

### HTTP Status Code

API endpoints don't throw a Wire404Exception, instead, they set the status code manually.

If no matching route is found, HTTP status 404 (Not Found) is returned.

If a route is found, but one of the checks failed, HTTP status 403 (Forbidden) is returned.


## Checks

You will want to check not only if a route's path applies to the current request,
but also make sure that the correctly formatted arguments are passed in the URL
and that the user has the necessary roles.

After creating a new route, you can add any number of checks which will be
processed in the order in which they were added.

### Sanitizer Checks

WireApi makes use of $sanitizer to avoid reinventing the wheel. So no matter if you
want to check if a URL argument is numeric, is uppercase or matches a regex, just to
name a few, if there is already a Sanitizer method for it in PW, you can use it.

All methods of $sanitizer are exposed as check_SANITIZERMETHOD, i.e.
```$sanitizer->unsignedInt``` becomes ```$route->check_unsignedInt```etc.

The first argument passed to check_SOMETHING must be the name of the URL argument.

```php
$api->route('/api/user/{userid}/', 'apitemplates/user')->check_unsignedInt('userid');
```

WireApi calls the $sanitizer method on the value in the captured field and compares
it with its original value. If both are identical, the check is okay.

### Role Checks

You can limit API access to certain user roles. Just call ```roles()``` on the route
and pass it the role name that is allowed. If you want to grant execution rights to
multiple roles, simply separate them with a pipe.

```php
$api->route('/api/secret/call/', 'apitemplates/secret')->roles('superuser|guest');
```

### Custom Functions / Methods

You may want to perform your very own checks to see if a route may be called.
In this case, you can assign a function or method that takes the following
arguments:

- $url The URL that was called
- $route The WireRoute object
- $check The associative array with the configuration data of the current check
- $values The associative array with all URL arguments

```
$route = $api->route(['POST'], '/api/v1/user/{userid}/', $this, 'setUserData');

// Set the checkUserData method in the current class as the check method:
$route->check($this, 'checkUserData');

// The next two are equivalent and assign a named function:
$route->check(null, 'checkUserData')
$route->check('checkUserData')

// Use an anomymous inline function:
$route->check(function($url, $route, $check, $values) { return $values['userid'] < 999999; });
```

## Verbose Examples
```PHP
// Simple GET route that populates the URL pattern paremeter 'number' and handles
// requests through an anonymous function.
$api->route(
  ['GET'],
  '/api/test/{number}/',
  function($url, $values) { echo json_encode(['number' => $value['number']]); }
 );

// More verbose POST route that populates $values['userid'], handles the request
// through the setUserData method of the current class and runs two checks to validate
// the path, the first through $sanitizer->unsignedInt, the second through a user defined
// function. It also verifies that the supplied Content-Type header is either text/json
// or application/json.
$api->route(['POST'], '/api/v1/user/{userid}/', $this, 'setUserData')
  ->check_unsignedInt('userid')
  ->check(function($url, $route, $check, $values) { return $values['userid'] < 999999; })
  ->ctype('text/json|application/json');

// Simple GET route using the shorthand method. The output is generated by rendering the
// apitemplates/userlist.php file.
$api->routeGET('/api/userlist/', 'apitemplates/userlist');
```

## License
Released under the terms of Mozilla Public License 2.0. See the LICENSE file in this repository.
