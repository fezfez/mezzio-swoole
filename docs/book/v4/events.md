# Swoole HTTP Server Events

The `Swoole\Http\Server` [emits a number of events](https://www.swoole.co.uk/docs/modules/swoole-http-server-doc#events) during its lifecycle.
Each is listened to by calling the server's `on()` method with the event name and a callback for handling the event.
In each case, Swoole only allows exactly one listener.
In some cases, Swoole will raise an exception when you attempt to register more listeners; in others, it will silently replace the listener.

To make the system more flexible, version 3 of this library now has the `Mezzio\Swoole\SwooleRequestHandlerRunner` service compose a [PSR-14 EventDispatcherInterface](https://www.php-fig.org/psr/psr-14/) instance that is then triggered for each Swoole HTTP Server event.
Additionally, the arguments passed to the Swoole listener are now aggregated into typed event classes, allowing access to the arguments by your PHP listener classes.

## Event classes

All event classes are in the `Mezzio\Swoole\Event` namespace.

### Start Event

Describes the "start" event emitted by the Swoole HTTP Server.

```php
namespace Mezzio\Swoole\Event;

use Swoole\Http\Server as SwooleHttpServer;

/**
 * Describes "start" event
 */
class ServerStartEvent
{
    public function getServer(): SwooleHttpServer;
}
```

### Manager Start Event

Describes the "managerstart" event emitted by the Swoole HTTP Server.

```php
namespace Mezzio\Swoole\Event;

use Swoole\Http\Server as SwooleHttpServer;

class ManagerStartEvent
{
    public function getServer(): SwooleHttpServer;
}
```

### Manager Stop Event

Describes the "managerstop" event emitted by the Swoole HTTP Server.

```php
namespace Mezzio\Swoole\Event;

use Swoole\Http\Server as SwooleHttpServer;

class ManagerStopEvent
{
    public function getServer(): SwooleHttpServer;
}
```

### Worker Start Event

Describes the "workerstart" event emitted by the Swoole HTTP Server.

```php
namespace Mezzio\Swoole\Event;

use Swoole\Http\Server as SwooleHttpServer;

class WorkerStartEvent
{
    public function getServer(): SwooleHttpServer;
    public function getWorkerId(): int;
}
```

### Worker Stop Event

Describes the "workerstop" event emitted by the Swoole HTTP Server.

```php
namespace Mezzio\Swoole\Event;

use Swoole\Http\Server as SwooleHttpServer;

class WorkerStopEvent
{
    public function getServer(): SwooleHttpServer;
    public function getWorkerId(): int;
}
```

### Worker Error Event

Describes the "workererror" event emitted by the Swoole HTTP Server.

```php
namespace Mezzio\Swoole\Event;

use Swoole\Http\Server as SwooleHttpServer;

class WorkerErrorEvent
{
    public function getServer(): SwooleHttpServer;
    public function getWorkerId(): int;
    public function getExitCode(): int;
    public function getSignal(): int;
}
```

### Request Event

Describes the "request" event emitted by the Swoole HTTP Server.

```php
namespace Mezzio\Swoole\Event;

use Psr\EventDispatcher\StoppableEventInterface;
use Swoole\Http\Request as SwooleHttpRequest;
use Swoole\Http\Response as SwooleHttpResponse;
use Swoole\Http\Server as SwooleHttpServer;

class RequestEvent implements StoppableEventInterface
{
    public function isPropagationStopped(): bool;
    public function getRequest(): SwooleHttpRequest;
    public function getResponse(): SwooleHttpResponse;
    public function responseSent(): void;
}
```

### Before Reload Event

Describes the "beforereload" event emitted by the Swoole HTTP Server.

```php
namespace Mezzio\Swoole\Event;

use Swoole\Http\Server as SwooleHttpServer;

class BeforeReloadEvent
{
    public function getServer(): SwooleHttpServer;
}
```

### After Reload Event

Describes the "afterreload" event emitted by the Swoole HTTP Server.

```php
namespace Mezzio\Swoole\Event;

use Swoole\Http\Server as SwooleHttpServer;

class AfterReloadEvent
{
    public function getServer(): SwooleHttpServer;
}
```

### Task Event

Describes the "task" event emitted by the Swoole HTTP Server.

```php
namespace Mezzio\Swoole\Event;

use Psr\EventDispatcher\StoppableEventInterface;
use Swoole\Http\Server as SwooleHttpServer;

class TaskEvent implements StoppableEventInterface
{
    public function isPropagationStopped(): bool;
    public function getServer(): SwooleHttpServer;
    public function getTaskId(): int;
    public function getWorkerId(): int;

    /**
     * Data passed to the task() method
     */
    public function getData(): mixed;

    /**
     * "Return value" of the task; what to pass to finish()
     *
     * @param mixed $returnValue
     */
    public function setReturnValue($returnValue): void;

    /** @return mixed */
    public function getReturnValue();

    /**
     * Call this to indicate processing is complete, and to stop processing
     * further listeners.
     */
    public function taskProcessingComplete(): void;
}
```

### Finish Event

Describes the "finish" event emitted by the Swoole HTTP Server, called when processing a task has been completed.

```php
namespace Mezzio\Swoole\Event;

use Swoole\Http\Server as SwooleHttpServer;

class TaskFinishEvent
{
    public function getServer(): SwooleHttpServer;
    public function getTaskId(): int;

    /**
     * Data passed to the finish() method
     */
    public function getData(): mixed;
}
```

### Shutdown Event

Describes the "shutdown" event emitted by the Swoole HTTP Server.

```php
namespace Mezzio\Swoole\Event;

use Swoole\Http\Server as SwooleHttpServer;

class ServerShutdownEvent
{
    public function getServer(): SwooleHttpServer;
}
```

## Listeners

We define the following listeners for the specified events.
All events and listeners are in the `Mezzio\Swoole\Event` namespace.

- `ServerStartEvent`:
  - `ServerStartListener`: initializes the master and manager PIDs, sets the working directory, sets the master process name, and logs that the server has started.
- `WorkerStartEvent`:
  - `WorkerStartListener`: sets the working directory, sets the worker process name, and logs the worker has started.
  - `HotCodeReloaderWorkerStartListener`: initiates a hot code reload within the worker.
- `RequestEvent`
  - `RequestHandlerRequestListener`: marshals a PSR-7 request from the Swoole HTTP request, passes it to the application request handler to generate a PSR-7 response, marshals and sends the Swoole HTTP Response from the PSR-7 response, and logs the request.
    If an error occurs marshaling the PSR-7 request, it will generate a PSR-7 error response, emit it, and log the request.
    In both cases, it marks the `responseSent()` in the event, stopping propagation.
  - `StaticResourceRequestListener`: attempts to process a static resource request; if it was able, it sends the Swoole HTTP Response, logs the access, and marks the `responseSent()` in the event, stopping propagation.
    Otherwise, it returns, allowing the next listener to handle the event.
- `ServerShutdownEvent`:
  - `ServerShutdownListener`: destroys the master and manager PIDs, all worker processes, and logs the server shutdown.

> ### Short-circuiting
>
> Only one event among those provided can be short-circuited: `Mezzio\Swoole\Event\RequestEvent`.
> All others will trigger every listener encountered.
>
> With regards to the `Mezzio\Swoole\Event\RequestEvent`, the `Mezzio\Swoole\Event\RequestHandlerRequestListener` **always** stops propagation.
> As such, if you want other listeners on that event, they **must** resolve before that listener.
> An example of such a listener is the `Mezzio\Swoole\Event\StaticResourceRequestListener`, which only stops propagation if it was able to resolve the request to a static resource.

## Providing a dispatcher and listeners

The shipped [ConfigProvider](https://docs.laminas.dev/laminas-config-aggregator/config-providers/) uses a factory for the `Mezzio\Swoole\SwooleRequestHandlerRunner` service that consumes the following services:

- `Swoole\Http\Server`, which represents the actual Swoole HTTP Server instance to run.
- `Mezzio\Swoole\Event\EventDispatcherInterface`

`SwooleRequestHandlerRunner` accepts any [PSR-14](https://www.php-fig.org/psr/psr-14/) instance.
The `Mezzio\Swoole\Event\EventDispatcherInterface` service referenced by its factory is a marker interface used only as a service name; this is done to allow you to use a different PSR-14 instance for the web server versus other services in your application.

By default, the `Mezzio\Swoole\Event\EventDispatcherInterface` service points to `Mezzio\Swoole\Event\EventDispatcher`, which is a simple PSR-14 dispatcher implementation.
That service in turn consumes `Mezzio\Swoole\Event\SwooleListenerProvider`.

The factory for `SwooleListenerProvider`, `Mezzio\Swoole\Event\SwooleListenerProviderFactory`, consumes the `config` service, looking for the following configuration:

```php
[
    'mezzio-swoole' => [
        'swoole-http-server' => [
            'listeners' => [
                'EventClassName' => [
                    // service names of listeners
                ],
            ],
        ],
    ],
]
```

Listeners for each event type are attached in the order they are discovered.

We recommend using the `SwooleListenerProvider` service to aggregate the various listeners for Swoole server events, even if you do not use the shipped `EventDispatcher` implementation.
Most PSR-14 libraries have functionality for aggregating multiple providers, and should allow you to compose the `SwooleListenerProvider` within them.

## FAQ

- [Removing the StaticResourceRequestListener](cookbook/static-resource-listener-removal.md)
