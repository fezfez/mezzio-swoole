# Static Resources

One feature of a web server is the ability to serve static files from your
filesystem. mezzio-swoole provides that capability as well.

To enable this, the package provides an alternate
[`RequestHandlerRunner`](https://docs.laminas.dev/laminas-httphandlerrunner/runner/)
implementation via the class `Mezzio\Swoole\SwooleRequestHandlerRunner`
that performs two duties:

- If a static resource is matched, it serves that.
- Otherwise, it passes off handling to the composed application pipeline.

Internally, the `SwooleRequestHandlerRunner` composes another class, a
`Mezzio\Swoole\StaticResourceHandlerInterface` instance. This instance
is passed the Swoole request and response, and returns a value indicating
whether or not it was able to identify and serve a matching static resource.

Our default implementation, `Mezzio\Swoole\StaticResourceHandler`,
provides an approach that checks an incoming request path against a list of
known extensions, and a configured document root. If the extension matches, it
then checks to see if the file exists in the document root. If it does, it will
serve it.

<!-- markdownlint-disable-next-line header-increment-->
> ### Disabling static resources
>
> By default, we ship with static resource handling enabled.
> This is done by having the `Mezzio\Swoole\Event\StaticResourceRequestListener` in the list of listeners provided for the `Mezzio\Swoole\Event\RequestEvent`.
>
> To disable that listener, you will need to **replace** the set of listeners for that event, to include only the `Mezzio\Swoole\Event\RequestHandlerRequestListener`.
> You can do that in your application configuration as follows:
>
> ```php
> // in config/autoload/dependencies.global.php:
>
> use Laminas\Stdlib\ArrayUtils\MergeReplaceKey;
> use Mezzio\Swoole\Event;
>
> return [
>     // ...
>     'mezzio-swoole' => [
>         // ...
>         'swoole-http-server' => [
>             // ...
>             'listeners' => [
>                 Event\RequestEvent::class => new MergeReplaceKey([
>                     Event\RequestHandlerRequestListener::class,
>                 ]),
>             ],
>         ],
>     ],
>     // ...
> ];
> ```

## Middleware

The `StaticResourceHandler` implementation performs its work by composing a
queue of middleware to execute when attempting to serve a matched file. Using
this approach, we are able to provide a configurable set of capabilities for
serving static resources. What we currently provide is as follows:

- `CacheControlMiddleware` will set a `Cache-Control` header based on
  configuration you provide it. Configuration uses a combination of regular
  expressions to match against the path, with the `Cache-Control` directive to
  use when the match occurs.

- `ClearStatCacheMiddleware` will, if configured to do so, call
  `clearstatcache()` either on every request, or at specific intervals. This is
  useful if you anticipate filesystem changes in your document root.

- `ContentTypeFilterMiddleware` checks the incoming filename against a map of
  known extensions and their associated Content-Type values. If it cannot
  match the file, it returns a value indicating no match was found so that the
  application can continue processing the request. Otherwise, it provides the
  Content-Type for the associated response. This middleware is generally best
  used as the outermost layer, to ensure no other middleware executes in the
  case that the file cannot be matched.

- `ETagMiddleware` will set an `ETag` header using either a strong or weak
  algorithm, and only on files matching given regular expressions. If the `ETag`
  header value matches either an `If-Match` or `If-None-Match` request header,
  it will provide a response status of `304` and disable sending content.

- `GzipMiddleware` detects the `Accept-Encoding` request header and, if present,
  and the compression level provided to the instance allows, it will compress
  the returned response content using either gzip or deflate compression as
  requested.

- `HeadMiddleware` will force an empty response. (The status and headers may be
  set by other middleware.)

- `LastModifiedMiddleware` will set a `Last-Modified` header using the
  `filemtime()` value of the requested resource. If the header value is later
  than an `If-Modified-Since`  request header, it will provide a response status
  of `304` and disable sending content.

- `MethodNotAllowedMiddleware` will set the response status to `405`, and set an
  `Allow` header indicating the allowed methods when an unsupported request
  method is provided.

- `OptionsMiddleware` will force an empty response with an `Allow` header set
  to the allowed methods. (Other headers may also be present!)

By default, these are registered in the following order, contingent on
configuration being provided:

- `ContentTypeFilterMiddleware`
- `MethodNotAllowedMiddleware`
- `OptionsMiddleware`
- `HeadMiddleware`
- `GzipMiddleware`
- `ClearStatCacheMiddleware`
- `CacheControlMiddleware`
- `LastModifiedMiddleware`
- `ETagMiddleware`

This approach ensures that the most expensive operations are never called unless
other conditions are met (e.g., if the HTTP request method is not allowed,
there's no need to calculate the `Last-Modified` or `ETag` headers); it also
ensures that all possible headers are provided whenever possible (e.g., a `HEAD`
request should also expose `Cache-Control`, `Last-Modified`, and `ETag`
headers).

> ### Providing your own middleware
>
> If you want to disable middleware, or to provide an alternate list of middleware
> (including your own!), you will need to provide an alternate
> `StaticResourceHandler` factory. In most cases, you can extend
> `StaticResourceHandlerFactory` and override the `configureMiddleware(array
> $config) : array` method to do so. Be sure to remember to add a `dependencies`
> setting mapping the `StaticResourceHandlerInterface` service to your new factory
> when done!

## Configuration

We provide a factory for the `StaticResourceHandler` that uses a
configuration-driven approach in order to:

- Set the document root.
- Set the map of allowed extensions to content-types.
- Configure and provide middleware.

The following demonstrates all currently available configuration options:

```php
// config/autoload/swoole.local.php

return [
    'mezzio-swoole' => [
        'swoole-http-server' => [
            'static-files' => [
                // Since 2.1.0: Set to false to disable any serving of static
                // files; all other configuration will then be ignored.
                'enable' => true,

                // Document root; defaults to "getcwd() . '/public'"
                'document-root' => '/path/to/static/files/to/serve',

                // Extension => content-type map.
                // Keys are the extensions to map (minus any leading `.`),
                // values are the MIME type to use when serving them.
                // A default list exists if none is provided.
                'type-map' => [],

                // How often a worker should clear the filesystem stat cache.
                // If not provided, it will never clear it. The value should be
                // an integer indicating the number of seconds between clear
                // operations. 0 or negative values will clear on every request.
                'clearstatcache-interval' => 3600,

                // Which ETag algorithm to use.
                // Must be one of "weak" or "strong"; the default, when none is
                // provided, is "weak".
                'etag-type' => 'weak|strong',

                // gzip options
                'gzip' => [
                    // Compression level to use.
                    // Should be an integer between 1 and 9; values less than 1
                    // disable compression.
                    'level' => 4,
                ],

                // Rules governing which server-side caching headers are emitted.
                // Each key must be a valid regular expression, and should match
                // typically only file extensions, but potentially full paths.
                // When a static resource matches, all associated rules will apply.
                'directives' => [
                    'regex' => [
                        'cache-control' => [
                            // one or more valid Cache-Control directives:
                            // - must-revalidate
                            // - no-cache
                            // - no-store
                            // - no-transform
                            // - public
                            // - private
                            // - max-age=\d+
                        ],
                        'last-modified' => bool, // Emit a Last-Modified header?
                        'etag' => bool, // Emit an ETag header?
                    ],
                ],
            ],
        ],
    ],
];
```

> ### Security warning
>
> Never add `php` as an allowed static file extension, as doing so could expose
> the source code of your PHP application!

> ### Document root
>
> If no `document_root` configuration is present, the default is to use
> `getcwd() . '/public'`. If either the configured or default document root
> does not exist, we raise an exception.

> ### Default extension/content-types
>
> By default, we serve files with extensions in the whitelist defined in the
> constant `Mezzio\Swoole\StaticResourceHandler\ContentTypeFilterMiddleware::DEFAULT_STATIC_EXTS`,
> which is derived from a [list of common web MIME types maintained by Mozilla](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/MIME_types/Complete_list_of_MIME_types).

### Configuration Example

The example which follows provides the following options:

- Sets the document root to `/var/www/htdocs`.
- Adds a custom extension / content-type map.
- Provides a clearstatcache interval of 2 hours.
- Selects the "strong" ETag algorithm.
- Indicates a gzip compression level of 3.
- Sets Cache-Control, Last-Modified, and ETag directives for JS, CSS, and image files.
- Sets Cache-Control directives for plain text files.

```php
// config/autoload/swoole.local.php

return [
    'mezzio-swoole' => [
        'swoole-http-server' => [
            'static-files' => [
                'enable'        => true,
                'document-root' => '/var/www/htdocs',

                'type-map' => [
                    'css'   => 'text/css',
                    'gif'   => 'image/gif',
                    'ico'   => 'image/x-icon',
                    'jpg'   => 'image/jpg',
                    'jpeg'  => 'image/jpg',
                    'js'    => 'application/javascript',
                    'png'   => 'image/png',
                    'svg'   => 'image/svg+xml',
                    'txt'   => 'text/plain',
                ],

                'clearstatcache-interval' => 7200,

                'etag-type' => 'strong',

                'gzip' => [
                    'level' => 3,
                ],

                'directives' => [
                    '/\.(css|gif|ico|jpg|jpeg|png|svg|js)$/' => [
                        'cache-control' => [
                            'public',
                            'no-transform',
                        ],
                        'last-modified' => true,
                        'etag' => true,
                    ],
                    '/\.txt$/' => [
                        'cache-control' => [
                            'public',
                            'no-cache',
                        ],
                    ],
                ],
            ],
        ],
    ],
];
```

## Writing Middleware

Static resource middleware must implement
`Mezzio\Swoole\StaticResourceHandler\MiddlewareInterface`, which
defines the following:

```php
namespace Mezzio\Swoole\StaticResourceHandler;

use Swoole\Http\Request;

interface MiddlewareInterface
{
    /**
     * @param string $filename The discovered filename being returned.
     * @param callable $next has the signature:
     *     function (Request $request, string $filename) : StaticResourceResponse
     */
    public function __invoke(
        Request $request,
        string $filename,
        callable $next
    ) : StaticResourceResponse;
}
```

The `$next` argument has the following signature:

```php
namespace Mezzio\Swoole\StaticResourceHandler;

use Swoole\Http\Request;

public function __invoke(
    Request $request,
    string $filename
) : StaticResourceResponse;
```

Typically, middleware will look something like this:

```php
$response = $next($request, $filename);

// if some request condition does not match:
// return $response;

// Otherwise, manipulate the returned $response instance and then return it.
```

Middleware either produces or manipulates a
`Mezzio\Swoole\StaticResourceHandler\StaticResourceResponse` instance.
That class looks like the following:

```php
class StaticResourceResponse
{
    /**
     * @param callable $responseContentCallback Callback to use when emitting
     *     the response body content via Swoole. Must have the signature:
     *     function (SwooleHttpResponse $response, string $filename) : void
     */
    public function __construct(
        int $status = 200,
        array $headers = [],
        bool $sendContent = true,
        callable $responseContentCallback = null
    );

    public function addHeader(string $name, string $value) : void;

    public function disableContent() : void;

    /**
     * Call this method to indicate that the request cannot be served as a
     * static resource. The request runner will then proceed to execute
     * the associated application in order to generate the response.
     */
    public function markAsFailure() : void;

    /**
     * @param callable $responseContentCallback Callback to use when emitting
     *     the response body content via Swoole. Must have the signature:
     *     function (SwooleHttpResponse $response, string $filename) : void
     */
    public function setResponseContentCallback(callable $callback) : void;

    /**
     * Use this within a response content callback to set the associated
     * Content-Length of the generated response. Loggers can then query
     * for this information in order to provide that information in the logs.
     */
    public function setContentLength(int $length) : void;

    public function setStatus(int $status) : void;
}
```

Most middleware will conditionally set the status, one or more headers, and
potentially disable returning the response body (via `disableContent()`).
Middleware that restricts access or filters out specific files will also use
`markAsFailure()`.

> ### Providing an alternative mechanism for sending response content
>
> In some cases, you may want to alter how the `Swoole\Http\Response` receives the
> body content. By default, we use `Swoole\Http\Response::sendfile()`. However,
> this may not work well when performing tasks such as compression, appending a
> watermark, etc. As an example, the `GzipMiddleware` adds a compression filter to
> a filehandle representing the file to send, and then calls
> `Swoole\Http\Response::write()` in a loop until all content is sent.
>
> To perform work like this, you can call the
> `StaticResourceResponse::setResponseContentCallback()` method as detailed in the
> section above within your middleware.

## Alternative static resource handlers

As noted at the beginning of this chapter, the `SwooleRequestHandlerRunner`
composes a `StaticResourceHandlerInterface` instance in order to determine if a
resource was matched by the request, and then to serve it.

If you want to provide an alternative mechanism for doing so (e.g., to serve
files out of a caching server), you will need to implement
`Mezzio\Swoole\StaticResourceHandlerInterface`:

```php
declare(strict_types=1);

namespace Mezzio\Swoole;

use Swoole\Http\Request as SwooleHttpRequest;
use Swoole\Http\Response as SwooleHttpResponse;

interface StaticResourceHandlerInterface
{
    /**
     * Attempt to process a static resource based on the current request.
     *
     * If the resource cannot be processed, the method should return null.
     * Otherwise, it should return the StaticResourceResponse that was used
     * to send the Swoole response instance. The runner can then query this
     * for content length and status.
     */
    public function processStaticResource(
        SwooleHttpRequest $request,
        SwooleHttpResponse $response
    ) : ?StaticResourceHandler\StaticResourceResponse;
}
```

Once implemented, map the service
`Mezzio\Swoole\StaticResourceHandlerInterface` to a factory that
returns your custom implementation within your `dependencies` configuration.

## Example alternate static resource handler: StaticMappedResourceHandler

- Since 2.7.0

The default static resource handler, `Mezzio\Swoole\StaticResourceHandler`, requires all files to be in the specified document root directory (by default, "public") when instantiating the handler.
If you are using modules generating templates with associated file assets (JavaScript, CSS, etc.), those files must be copied to the "public" directory if you wish to allow access to them.
This can be done via scripting, but is one more step to consider when testing or deploying a site.
Ideally, a module should be able to contain both its template and any dependencies that template relies upon.

For example, assume you have a module, AwesomeModule, with a handler called "HomeHandler", which renders the 'home' template.
You designate the prefix, `/awesome-home` for rendering the assets.
The structure of your module files looks like this:

```text
AwesomeModule
├── src
|   ├── Handler
|   |   ├── HomeHandler.php
|   |   ├── HomeHandlerFactory.php
|   ├── ConfigProvider.php
├── templates
│   ├── home
|   |   ├── home.html
|   |   ├── style.css
│   ├── layouts
```

In your `home.html` template, you can refer to the `style.css` file, using `/awesome-home` as follows:

```html
<link href="/awesome-home/style.css" rel="stylesheet" type="text/css">
```

Currently, however, this will cause errors, as the stylesheet will not be available under the `public/` tree of the application.
You would need to copy or symlink the files to the appropriate location to make that work.

The `StaticMappedResourceHandler` solves this problem.

### Using StaticMappedResourceHandler

To use `Mezzio\Swoole\StaticMappedResourceHandler` from an application or module:

1. Define what your URI prefix will be (e.g., `/awesome-home`).
2. Update references to linkable resources in your templates to use the desired prefix (e.g., `<script src='/awesome-home/style.css'></script>`).
3. In your application/module configuration (or `ConfigProvider`), add the relationship between your prefix (`awesome-home`) and any directories containing the assets.
4. In the application's configuration, set the alias of `Mezzio\Swoole\StaticResourceHandlerInterface` to use `Mezzio\Swoole\StaticMappedResourceHandler`.

For step #3, in your module's ConfigProvider, you can add a configuration setting as follows:

```php
    public function __invoke() : array
    {
        return [
            'config' => [
                'mezzio-swoole' => [
                    'swoole-http-server' => [
                        'static-files' => [
                            'mapped-document-roots' => [
                                'awesome-home' => __DIR__ . '/../../templates/home'
                            ]
                        ]
                    ]
                ]
            ]
        ];
    }
```

Note that prefixes are always the first part after the host, and specifying the initial slash is optional (i.e. `awsesome-home` and `/awesome-home` both work and represent the same thing).

In step #4, in your application's configuration (`autoload/dependencies.global.php` is a good place), override the default implementation of `StaticResourceHandlerInterface`:

```php
return [
    'dependencies' => [
        'aliases' => [
            Mezzio\Swoole\StaticResourceHandlerInterface::class => Mezzio\Swoole\StaticMappedResourceHandler::class
            // Fully\Qualified\ClassOrInterfaceName::class => Fully\Qualified\ClassName::class,
        ],
        // etc.
```

For step #3, an alternative to storing a configuration is dynamically associating `/awesome-home` to a directory in code (probably within a factory).
This approach could be useful if the directory of the assets isn't know until runtime.

```php
use Psr\Container\ContainerInterface;
use Mezzio\Template\TemplateRendererInterface;
use Mezzio\Swoole\StaticResourceHandler\FileLocationRepositoryInterface;

class AwesomeHomeHandlerFactory
{
    public function __invoke(ContainerInterface $container) : DocumentationViewHandler
    {
        // Establish location for the home template assets
        $repo = $container->get(FileLocationRepositoryInterface::class);
        $repo->addMappedDocumentRoot(
            'awesome-home',
            realpath(__DIR__ . '/../../templates/home')
        );

        return new AwesomeHomeHandler(
            $container->get(TemplateRendererInterface::class)
        );
    }
}
```

When the template renders, the client will request `/awesome-home/style.css`, which the `StaticMappedResourceHandler` will now retrieve from the `templates/home/` folder of the module.

`Mezzio\Swoole\StaticMappedResourceHandler` uses the `Mezzio\Swoole\StaticResourceHandler\FileLocationRepository` (which implements `Mezzio\Swoole\StaticResourceHandler\FileLocationRepositoryInterface`) to maintain an association of URI prefixes with file directories.
If you require using a file location that requires authentication, decompression, etc. you can override the default functionality by creating your own implementation of `Mezzio\Swoole\StaticResourceHandler\FileLocationRepositoryInterface`.
