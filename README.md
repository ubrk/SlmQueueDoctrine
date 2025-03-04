SlmQueueDoctrine
================

Important notice
----------

**We decided to move onto Symfony Messenger and we are therefore not maintaining this repository anymore. Feel free to fork it and
make it your own.**

Created by Stefan Kleff

Requirements
------------
* [SlmQueue](https://github.com/ubrk/SlmQueue)
* [Doctrine 2 ORM Module](https://github.com/doctrine/DoctrineORMModule) or [roave/psr-container-doctrine](https://github.com/roave/psr-container-doctrine)

Note: it's necessary require the doctrine package in composer.json file.

Installation
------------

To install this fork, adds the new repositories to your `composer.json` file

```json
  "repositories": [
    {
      "type": "vcs",
      "url": "https://github.com/ubrk/SlmQueue"
    },
    {
      "type": "vcs",
      "url": "https://github.com/ubrk/SlmQueueDoctrine"
    }
  ]
```

Then, run `composer require slm/queue-doctrine`.

If you have the [laminas/laminas-component-installer](https://github.com/laminas/laminas-component-installer) package installed, it will ask you to enable the module (and `SlmQueue`), both in Laminas and Mezzio. Otherwise, add the module to the list:
* in Laminas MVC, enable the module by adding `SlmQueueDoctrine` in your application.config.php file.
* in Mezzio, enable the module by adding `SlmQueueDoctrine\ConfigProvider::class,` in your config.php file.

Note: Don't forget install [SlmQueue](https://github.com/ubrk/SlmQueue) in you config file, which is required.

Documentation
-------------

Before reading SlmQueueDoctrine documentation, please read [SlmQueue documentation](https://github.com/ubrk/SlmQueue/blob/master/docs/1.Introduction.md).

### Configuring the connection

You need to register a doctrine connection which SlmQueueDoctrine will use to access the database into the service manager. Here are some [examples](https://github.com/Roave/psr-container-doctrine/tree/master/example).

Connection parameters can be defined in the application configuration:

```php
<?php
return [
    'doctrine' => [
        'connection' => [
            // default connection name
            'orm_default' => [
                'driverClass' => 'Doctrine\DBAL\Driver\PDOMySql\Driver',
                'params' => [
                    'host'     => 'localhost',
                    'port'     => '3306',
                    'user'     => 'username',
                    'password' => 'password',
                    'dbname'   => 'database',
                ]
            ]
        ]
    ],
];
```

### Creating the table from SQL file

You must create the required table that will contain the queue's you may use the schema located in 'data/queue_default.sql'. If you change the table name look at [Configuring queues](./#configuring-queues)

```
>mysql database < data/queue_default.sql
```
### Creating the table from Doctrine Entity
There is an alternative way to create 'queue_default' table in your database by copying Doctrine Entity 'date/DefaultQueue.php' to your entity folder ('Application\Entity' in our example) and executing Doctrine's 'orm:schema-tool:update' command which should create the table for you. Notice that DefaultQueue entity is only used for table creation and is not used by this module internally.


### Adding queues

```php
return [
  'slm_queue' => [
    'queue_manager' => [
      'factories' => [
        'foo' => 'SlmQueueDoctrine\Factory\DoctrineQueueFactory'
      ]
    ]
  ]
];
```
### Adding jobs

```php
return [
  'slm_queue' => [
    'job_manager' => [
      'factories' => [
        'My\Job' => 'My\JobFactory'
      ]
    ]
  ]
];

``` 
### Configuring queues

The following options can be set per queue ;
	
- connection (defaults to 'doctrine.connection.orm_default') : Name of the registered doctrine connection service
- table_name (defaults to 'queue_default') : Table name which should be used to store jobs
- deleted_lifetime (defaults to 0) : How long to keep deleted (successful) jobs (in minutes)
- buried_lifetime (defaults to 0) : How long to keep buried (failed) jobs (in minutes)


```php
return [
  'slm_queue' => [
    'queues' => [
      'foo' => [
        // ...
      ]
    ]
  ]
];
 ```
 
Provided Worker Strategies
--------------------------

In addition to the provided strategies by [SlmQueue](https://github.com/JouwWeb/SlmQueue/blob/master/docs/6.Events.md) SlmQueueDoctrine comes with these strategies;

#### ClearObjectManagerStrategy

This strategy will clear the ObjectManager before execution of individual jobs. The job must implement the [DoctrineModule\Persistence\ObjectManagerAwareInterface](https://github.com/doctrine/DoctrineModule/blob/master/src/DoctrineModule/Persistence/ObjectManagerAwareInterface.php) or [SlmQueueDoctrine\Persistence\ObjectManagerAwareInterface](https://github.com/JouwWeb/SlmQueueDoctrine/blob/master/src/Persistence/ObjectManagerAwareInterface.php).

listens to:

- `process.job` event at priority 1000

options:

- none

This strategy is enabled by default.

#### IdleNapStrategy

When no jobs are available in the queue this strategy will make the worker wait for a specific amount time before quering the database again.

listens to:

- `process.idle` event at priority 1

options:

- `nap_duration` defaults to 1 (second)

This strategy is enabled by default.

### Operations on queues

#### push

Valid options are:

* scheduled: the time when the job will be scheduled to run next
	* numeric string or integer - interpreted as a timestamp
	* string parserable by the DateTime object
	* DateTime instance
* delay: the delay before a job become available to be popped (defaults to 0 - no delay -)
	* numeric string or integer - interpreted as seconds
	* string parserable (ISO 8601 duration) by DateTimeInterval::__construct
	* string parserable (relative parts) by DateTimeInterval::createFromDateString
	* DateTimeInterval instance
* priority: the lower the priority is, the sooner the job get popped from the queue (default to 1024)

Examples:
```php
// scheduled for execution asap
$queue->push($job);

// will get executed before jobs that have higher priority
$queue->push($job, [
    'priority' => 200,
]);

// scheduled for execution 2015-01-01 00:00:00 (system timezone applies)
$queue->push($job, [
    'scheduled' => 1420070400,
]);

// scheduled for execution 2015-01-01 00:00:00 (system timezone applies)
$queue->push($job, [
    'scheduled' => '2015-01-01 00:00:00'
]);

// scheduled for execution at 2015-01-01 01:00:00
$queue->push($job, [
    'scheduled' => '2015-01-01 00:00:00',
    'delay' => 3600
]);

// scheduled for execution at now + 300 seconds
$queue->push($job, [
    'delay' => 'PT300S'
]);

// scheduled for execution at now + 2 weeks (1209600 seconds)
$queue->push($job, [
    'delay' => '2 weeks'
]);

// scheduled for execution at now + 300 seconds
$queue->push($job, [
    'delay' => new DateInterval("PT300S"))
]);
```


### Worker actions

Interact with workers from the command line from within the public folder of your Laminas Framework 2 application

#### Starting a worker
Start a worker that will keep monitoring a specific queue for jobs scheduled to be processed. This worker will continue until it has reached certain criteria (exceeds a memory limit or has processed a specified number of jobs).

`vendor/bin/laminas slm-queue:start <queueName>`

A worker will exit when you press cntr-C *after* it has finished the current job it is working on. (PHP doesn't support signal handling on Windows)

#### Recovering jobs

To recover jobs which are in the 'running' state for prolonged period of time (specified in minutes) use the following command.

`vendor/bin/laminas slm-queue:doctrine:recover <queueName> [--executionTime=]`

*Note : Workers that are processing a job that is being recovered are NOT stopped.*

