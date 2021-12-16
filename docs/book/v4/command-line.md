# Command Line Tooling

This package provides integration with [laminas-cli](https://docs.laminas.dev/laminas-cli/), and provides the following commands:

- `mezzio:swoole:start` to start the server
- `mezzio:swoole:stop` to stop the server (when run in daemonized mode)
- `mezzio:swoole:reload` to reload the server (when run in daemonized mode)
- `mezzio:swoole:status` to determine the server status (running or not running)

You may obtain help for each command using the `help` meta-command:

```bash
$ ./vendor/bin/laminas help mezzio:swoole:start
```

The `mezzio:swoole:stop`, `mezzio:swoole:status`, and `mezzio:swoole:reload` commands are sufficiently generic to work regardless of runtime or application, as they work directly with the Swoole process manager.
The `mezzio:swoole:start` command, however, may need customizations if you have customized your application bootstrap.

## The mezzio:swoole:start command

The `mezzio:swoole:start` command will start the web server using the following steps:

- It pulls the `Swoole\Http\Server` service from the application dependency injection container, and calls `set()` on it with options denoting the number of workers to run (provided via the `--num-workers` or `-w` option), the number of task workers to spawn (provided via the `--num-task-workers` or `-t` option), and whether or not to daemonize the server (provided via the `--daemonize` or `-d` option).

- It pulls the `Mezzio\Application` and `Mezzio\MiddlewareFactory` services from the container.

- It loads the `config/pipeline.php` and `config/routes.php` files, invoking their return values with the application, middleware factory, and dependency injection container instances.

- It calls the `run()` method of the application instance.

These are roughly the steps taken within the application bootstrap (`public/index.php`) of the Mezzio skeleton application.

### Writing a custom mezzio:swoole:start command

If your application needs alternate bootstrapping (e.g., if you have modified the `public/index.php`, or if you are using this package with a different middleware runtime), we recommend writing a custom `mezzio:swoole:start` command.

As an example, let's say you have altered your application such that you're defining your routes in multiple files, and instead of:

```php
(require 'config/routes.php')($app, $factory, $container);
```

you instead have something like:

```php
$handle = opendir('config/routes/');
while (false !== ($entry = readdir($handle))) {
    if (false === strrpos($entry, '.php')) {
        continue;
    }
    (require $entry)($app, $factory, $container);
}
```

You could write a command such as the following:

```php
// In src/App/Command/StartCommand.php:

namespace App\Command;

use Psr\Container\ContainerInterface;
use Swoole\Http\Server as SwooleHttpServer;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;
use Mezzio\Application;
use Mezzio\MiddlewareFactory;
use Mezzio\Swoole\Command\StartCommand as BaseStartCommand;
use Mezzio\Swoole\PidManager;

class StartCommand extends BaseStartCommand
{
    protected function execute(InputInterface $input, OutputInterface $output) : int
    {
        // This functionality is identical to the base start command, and should
        // be copy and pasted to your implementation:
        $this->pidManager = $this->container->get(PidManager::class);
        if ($this->isRunning()) {
            $output->writeln('<error>Server is already running!</error>');
            return 1;
        }

        $server = $this->container->get(SwooleHttpServer::class);
        $server->set([
            'daemonize' => $input->getOption('daemonize'),
            'worker_num' => $input->getOption('num-workers') ?? self::DEFAULT_NUM_WORKERS,
        ]);

        /** @var \Mezzio\Application $app */
        $app = $this->container->get(Application::class);

        /** @var \Mezzio\MiddlewareFactory $factory */
        $factory = $this->container->get(MiddlewareFactory::class);

        // Execute programmatic/declarative middleware pipeline and routing
        // configuration statements
        (require 'config/pipeline.php')($app, $factory, $this->container);

        //
        // This is the new code from above:
        //
        $handle = opendir(getcwd() . '/config/routes/');
        while (false !== ($entry = readdir($handle))) {
            if (false === strrpos($entry, '.php')) {
                continue;
            }
            (require $entry)($app, $factory, $container);
        }

        // And now we return to the original code:

        // Run the application
        $app->run();

        return 0;
    }
}
```

You will also need to write a factory for the class:

```php
// In src/App/Command/StartCommandFactory.php:

namespace App\Command;

use Psr\Container\ContainerInterface;

class StartCommandFactory
{
    public function __invoke(ContainerInterface $container) : StartCommand
    {
        return new StartCommand($container);
    }
}
```

If this is all you're changing, you can map this new command to the existing `Mezzio\Swoole\Command\StartCommand` service within your configuration:

```php
// in config/autoload/dependencies.global.php:

use App\Command\StartCommandFactory;
use Mezzio\Swoole\Command\StartCommand;

return [
    'dependencies' => [
        'factories' => [
            StartCommand::class => StartCommandFactory::class,
        ],
    ],
];
```

Since the `laminas` binary uses your application configuration and container, this will substitute your command for the shipped command!
