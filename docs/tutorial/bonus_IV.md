# Bonus IV - OOP Flavour

The previous bonus part introduced Event Machine Flavours, especially the **FunctionalFlavour**.
Biggest change was the replacement of generic Event Machine messages with dedicated message types.

## Original Object-Oriented Programming

Event Machine emphasizes the usage of a functional core. That's true for the **PrototypingFlavour** and unfolds completely
with the **FunctionalFlavour**. A functional core has huge advantages compared to its object-oriented counterpart.
At least compared to the way we tend to work with objects in our projects the last twenty years or so.

Dr. Alan Kay (who has coined the term) had quite a different idea of object-oriented programming back in 1967.

> I thought of objects being like biological cells and/or individual computers on a network, only able to communicate with messages

*[source](http://userpage.fu-berlin.de/~ram/pub/pub_jf47ht81Ht/doc_kay_oop_en)*

Let that sink in - *only able to communicate with messages*.

If you look at what we've built so far, you might recognize that we are very close to that statement.
Finder/Resolver, Event Listener, Process Manager and Projector all are invoked with messages. They don't interact with each other directly.
Event Machine takes over coordination. It's like the network Alan Kay is talking about. But what about aggregate functions?
The functions are stateless and don't have side effects. They are **pure**. Immutable data types and messages (commands or events) are passed to
them. With coordination performed by Event Machine pure functions work great.

## It Is Not Functional Programming

We are not used to work with pure functions in PHP.
It's not a functional programming language, right? Autoloading functions doesn't work so we are either forced to require all files manually
or use the workaround shown in the tutorial to turn pure functions into static methods of otherwise useless classes.

{.alert .alert-dark}
Personally, I don't have a big problem with the latter approach. I see those classes as the last part of the namespace or even similar to an ES6 module
(if you're familiar with JavaScript). The module (PHP class) can export functions (public static functions) and use internal functions (private static functions).
But I have to admit that it is a workaround.

## OopFlavour on top of FunctionalFlavour

What can we do if the workaround is not acceptable for a project or personal taste? **Exactly, we can pick another Flavour :D**

The **OopFlavour** in a nutshell:

{.alert .alert-info}
Aggregate functions (command handling and apply functions) are combined with state into one object. Each aggregate manages its own state internally.
Commands trigger state changes. A state change is first recorded as an event and then applied by the aggregate.

You know what this means, right?

> I thought of objects being like biological cells and/or individual computers on a network, only able to communicate with messages

As I said, we're very close to that statement. That's the reason why the **OopFlavour** uses the **FunctionalFlavour** internally. It works on top of it
only to combine aggregate functions and state. More on that in a minute. First we need a solid foundation for event sourced objects.

## OOP Port

Similar to the `Functional\Port` we need to implement an `Oop\Port` to use the **OopFlavour**. Let's start again by looking at the required methods.
Create a new class `EventSourcedAggregatePort` in `src/Infrastructure/Flavour`:

```php
<?php
declare(strict_types=1);

namespace App\Infrastructure\Flavour;

use Prooph\EventMachine\Runtime\Oop\Port;

final class EventSourcedAggregatePort implements Port
{
    /**
     * @param string $aggregateType
     * @param callable $aggregateFactory
     * @param $customCommand
     * @param null|mixed $context
     * @return mixed Created aggregate
     */
    public function callAggregateFactory(string $aggregateType, callable $aggregateFactory, $customCommand, $context = null)
    {
        // TODO: Implement callAggregateFactory() method.
    }

    /**
     * @param mixed $aggregate
     * @param mixed $customCommand
     * @param null|mixed $context
     */
    public function callAggregateWithCommand($aggregate, $customCommand, $context = null): void
    {
        // TODO: Implement callAggregateWithCommand() method.
    }

    /**
     * @param mixed $aggregate
     * @return array of custom events
     */
    public function popRecordedEvents($aggregate): array
    {
        // TODO: Implement popRecordedEvents() method.
    }

    /**
     * @param mixed $aggregate
     * @param mixed $customEvent
     */
    public function applyEvent($aggregate, $customEvent): void
    {
        // TODO: Implement applyEvent() method.
    }

    /**
     * @param mixed $aggregate
     * @return array
     */
    public function serializeAggregate($aggregate): array
    {
        // TODO: Implement serializeAggregate() method.
    }

    /**
     * @param string $aggregateType
     * @param iterable $events history
     * @return mixed Aggregate instance
     */
    public function reconstituteAggregate(string $aggregateType, iterable $events)
    {
        // TODO: Implement reconstituteAggregate() method.
    }
}

```

This time we don't work top to bottom but start in the middle. `popRecordedEvents` and `applyEvent` are the first targets.

{.alert .alert-info}
Same basic rules apply here as we discussed for the `Functional\Port`.
Event Machine does not require a specific strategy to work with event sourced aggregates. You can implement them in any way as
long as the `Oop\Port` is able to fulfill the contract. That said, the approach shown in the tutorial is just a suggestion.
We're going to use a simple and pragmatic implementation with publicly accessible methods that are actually internal methods.
You might want to hide them in your project using a decorator or PHP's Reflection API. Anyway, that would be overkill for the tutorial.

## Event Sourced Aggregate Root

{.alert .alert-light}
A state change is first recorded as an event and then applied by the aggregate.

Let's create an interface for the port to rely on:

`src/Model/Base/AggregateRoot.php`

```php
<?php
declare(strict_types=1);

namespace App\Model\Base;

interface AggregateRoot
{
    /**
     * @return DomainEvent[]
     */
    public function popRecordedEvents(): array;

    public function apply(DomainEvent $event): void;
}

```

We don't have a `DomainEvent` type yet. Add it next to the `AggregateRoot` interface in the same directory.

`src/Model/Base/DomainEvent.php`

```php
<?php
declare(strict_types=1);

namespace App\Model\Base;

interface DomainEvent
{
    //Marker interface
}

```

With those two interfaces we can implement the first methods of the `Oop\Port`:

`src/Infrastructure/Flavour/EventSourcedAggregatePort.php`

```php
<?php
declare(strict_types=1);

namespace App\Infrastructure\Flavour;

use App\Model\Base\AggregateRoot;
use Prooph\EventMachine\Runtime\Oop\Port;

final class EventSourcedAggregatePort implements Port
{
    /* ... */

    /**
     * @param mixed $aggregate
     * @return array of custom events
     */
    public function popRecordedEvents($aggregate): array
    {
        if(!$aggregate instanceof AggregateRoot) {
            throw new \RuntimeException(
                sprintf("Cannot pop recorded events. Given aggregate is not an instance of %s. Got %s",
                    AggregateRoot::class,
                    (is_object($aggregate)? get_class($aggregate) : gettype($aggregate))
                )
            );
        }

        return $aggregate->popRecordedEvents();
    }

    /**
     * @param mixed $aggregate
     * @param mixed $customEvent
     */
    public function applyEvent($aggregate, $customEvent): void
    {
        if(!$aggregate instanceof AggregateRoot) {
            throw new \RuntimeException(
                sprintf("Cannot apply event. Given aggregate is not an instance of %s. Got %s",
                    AggregateRoot::class,
                    (is_object($aggregate)? get_class($aggregate) : gettype($aggregate))
                )
            );
        }

        $aggregate->apply($customEvent);
    }

    /* ... */
}

```

## Aggregate Root Lifecycle

Next two methods we are looking at are `callAggregateFactory` and `reconstituteAggregate`. The former starts the lifecycle of a new aggregate and the latter brings it
back into shape by passing aggregate event history (all events previously recorded by the aggregate) to the method.

Traits are a great way to reuse code snippets without inheritance. It's like copy and pasting methods from a blueprint into a class. Let's define one for common event sourcing
logic that we can later use in aggregates.

`src/Model/Base/EventSourced.php`

```php
<?php
declare(strict_types=1);

namespace App\Model\Base;

trait EventSourced
{
    /**
     * @var DomainEvent[]
     */
    private $recordedEvents = [];

    /**
     * @param DomainEvent[] $domainEvents
     * @return EventSourced aggregate
     */
    public static function reconstituteFromHistory(DomainEvent ...$domainEvents): AggregateRoot
    {
        $self = new self();
        foreach ($domainEvents as $domainEvent) {
            $self->apply($domainEvent);
        }
        return $self;
    }

    private function __construct()
    {
        //Do not override this!!!!
        //Use named constructors aka public static factory methods to create aggregae instances!
    }

    private function recordThat(DomainEvent $event): void
    {
        $this->recordedEvents[] = $event;
    }

    /**
     * @return DomainEvent[]
     */
    public function popRecordedEvents(): array
    {
        $events = $this->recordedEvents;
        $this->recordedEvents = [];
        return $events;
    }

    public function apply(DomainEvent $event): void
    {
        $whenMethod = $this->deriveMethodNameFromEvent($event);

        if(!method_exists($this, $whenMethod)) {
            throw new \RuntimeException(\sprintf(
                "Unable to apply event %s. Missing method %s in class %s",
                \get_class($event),
                $whenMethod,
                \get_class($this)
            ));
        }

        $this->{$whenMethod}($event);
    }

    private function deriveMethodNameFromEvent(DomainEvent $event): string
    {
        $nameParts = \explode('\\', \get_class($event));
        return 'when' . \array_pop($nameParts);
    }
}

```

The trait provides implementations for `popRecordedEvents` and `apply` defined by `AggregateRoot`. But it contains some more stuff!

### Derive Method Name From Event

A convention is used that says: **An aggregate should have an apply method for each domain event following the naming pattern "when\<EventName\>",
whereby \<EventName\> is the class name of the event without namespace.**

### Record That

An aggregate should use `recordThat` to record new domain events. The trait takes care of storing recorded events internally until the `Oop\Port` calls `popRecordedEvents()`.

### Private Empty Constructor

While a trait cannot enforce a private empty `__construct` (it could be overridden by a class), it's still included in the trait as a reminder for future developers to not
use `__construct` in aggregate roots but rather use named constructors. This rule is important for `Oop\Port::callAggregateFactory()`. More on that in a minute.

### Reconstitute From History

`reconstituteFromHistory` should be called by the `Oop\Port`. But the port works against our `AggregateRoot` interface, so we should add such a method signature there, too.

`src/Model/Base/AggregateRoot.php`

```php
<?php
declare(strict_types=1);

namespace App\Model\Base;

interface AggregateRoot
{
    public static function reconstituteFromHistory(DomainEvent ...$domainEvents): self;

    /**
     * @return DomainEvent[]
     */
    public function popRecordedEvents(): array;

    public function apply(DomainEvent $event): void;
}

```

Cool, we can implement the next port method now!

`src/Infrastructure/Flavour/EventSourcedAggregatePort.php`

```php
<?php
declare(strict_types=1);

namespace App\Infrastructure\Flavour;

use App\Model\Base\AggregateRoot;
use Prooph\EventMachine\Runtime\Oop\Port;

final class EventSourcedAggregatePort implements Port
{
    /* ... */

    /**
     * @param string $aggregateType
     * @param iterable $events history
     * @return mixed Aggregate instance
     */
    public function reconstituteAggregate(string $aggregateType, iterable $events)
    {
        $arClass = $this->getAggregateClassOfType($aggregateType);

        /** @var AggregateRoot $arClass */
        return $arClass::reconstituteFromHistory(...$events);
    }

    private function getAggregateClassOfType(string $aggregateType): string
    {
        switch ($aggregateType) {
            case Aggregate::BUILDING:
                return Building::class;
            default:
                throw new \RuntimeException("Unknown aggregate type $aggregateType");
        }
    }
}

```

Obviously, this won't work. We did not touch `Building` yet. Let's do that next.

## Merge Functions And State

Our `Building` aggregate consists of a set of pure functions grouped in a class and immutable data types. Turning it into an event sourced
object is less work than you might expect:

`src/Model/Building.php`

```php
<?php
declare(strict_types=1);

namespace App\Model;

use App\Model\Base\AggregateRoot;
use App\Model\Base\EventSourced;
use App\Model\Building\Command\AddBuilding;
use App\Model\Building\Command\CheckInUser;
use App\Model\Building\Command\CheckOutUser;
use App\Model\Building\Event\BuildingAdded;
use App\Model\Building\Event\DoubleCheckInDetected;
use App\Model\Building\Event\DoubleCheckOutDetected;
use App\Model\Building\Event\UserCheckedIn;
use App\Model\Building\Event\UserCheckedOut;

final class Building implements AggregateRoot
{
    use EventSourced;

    /**
     * @var Building\State
     */
    private $state;

    public static function add(AddBuilding $addBuilding): AggregateRoot
    {
        $self = new self();
        $self->recordThat(BuildingAdded::fromArray($addBuilding->toArray()));
        return $self;
    }

    public function whenBuildingAdded(BuildingAdded $buildingAdded): void
    {
        $this->state = Building\State::fromArray($buildingAdded->toArray());
    }

    public function checkInUser(CheckInUser $checkInUser): void
    {
        if($this->state->isUserCheckedIn($checkInUser->name())) {
            $this->recordThat(DoubleCheckInDetected::fromArray($checkInUser->toArray()));
            return;
        }

        $this->recordThat(UserCheckedIn::fromArray($checkInUser->toArray()));
    }

    private function whenUserCheckedIn(UserCheckedIn $userCheckedIn): void
    {
        $this->state = $this->state->withCheckedInUser($userCheckedIn->name());
    }

    private function whenDoubleCheckInDetected(DoubleCheckInDetected $event): void
    {
        //No state change required
    }

    public function checkOutUser(CheckOutUser $checkOutUser): void
    {
        if(!$this->state->isUserCheckedIn($checkOutUser->name())) {
            $this->recordThat(DoubleCheckOutDetected::fromArray($checkOutUser->toArray()));
            return;
        }

        $this->recordThat(UserCheckedOut::fromArray($checkOutUser->toArray()));
    }

    private function whenUserCheckedOut(UserCheckedOut $userCheckedOut): void
    {
        $this->state = $this->state->withCheckedOutUser($userCheckedOut->name());
    }

    private function whenDoubleCheckOutDetected(DoubleCheckOutDetected $event): void
    {
        //No state change required
    }
}


```

Here are the refactoring steps:

- All events need to implement `App\Model\Base\DomainEvent`
- `Building` implements `AggregateRoot`
- `Building` uses `EventSourced`
- `Building` stores `Building\State` internally in a `state` property
- `Building::add()` creates an instance of itself and records `BuildingAdded` instead of yielding it
- All other command handling functions:
    - Remove `static`, they become instance methods
    - Change return type to `void`
    - `Building\State` is no longer an argument, but accessed internally
    - Domain events get recorded
- All apply/when functions
    - Remove `static` and make them `private`, they become internal methods
    - Change return type to `void`
    - `Building\State` is no longer an argument, but accessed internally

## Aggregate Factory

`Building::add()` is the aggregate factory for `Building`. The `Oop\Port` can simply call it.

`src/Infrastructure/Flavour/EventSourcedAggregatePort.php`

```php
<?php
declare(strict_types=1);

namespace App\Infrastructure\Flavour;

use App\Model\Base\AggregateRoot;
use Prooph\EventMachine\Runtime\Oop\Port;

final class EventSourcedAggregatePort implements Port
{
    /**
     * @param string $aggregateType
     * @param callable $aggregateFactory
     * @param $customCommand
     * @param null|mixed $context
     * @return mixed Created aggregate
     */
    public function callAggregateFactory(string $aggregateType, callable $aggregateFactory, $customCommand, $context = null)
    {
        return $aggregateFactory($customCommand, $context);
    }

    /* ... */
}

```

The `callable $aggregateFactory` passed to the port, is still the one we've defined in the Event Machine Description:

`src/Api/Aggregate.php`

```php
<?php
declare(strict_types=1);

namespace App\Api;

use App\Model\Building;
use Prooph\EventMachine\EventMachine;
use Prooph\EventMachine\EventMachineDescription;

class Aggregate implements EventMachineDescription
{
    const BUILDING = 'Building';

    /**
     * @param EventMachine $eventMachine
     */
    public static function describe(EventMachine $eventMachine): void
    {
        $eventMachine->process(Command::ADD_BUILDING)
            ->withNew(self::BUILDING)
            ->identifiedBy(Payload::BUILDING_ID)
            ->handle([Building::class, 'add']) //<-- Aggregate Factory
            ->recordThat(Event::BUILDING_ADDED)
            ->apply([Building::class, 'whenBuildingAdded']);

        /* ... */
    }
}

```

{.alert .alert-warning}
`$context` is not an argument of `Building::add()` but PHP does not care. We can use that to our advantage.
The port does not need to know if an aggregate factory or command handling function is interested in a context or not.
It just passes it always to the function. If context is `null` and the function doesn't care, everything is fine.

## Command Handling

`Oop\Port::callAggregateWithCommand()` is next on the list. Let's see ...

```php
/**
 * @param mixed $aggregate
 * @param mixed $customCommand
 * @param null|mixed $context
 */
public function callAggregateWithCommand($aggregate, $customCommand, $context = null): void
{
    // TODO: Implement callAggregateWithCommand() method.
}
```

We get the `$aggregate` instance, a `$customCommand` and optionally a `$context`. We could use a `switch (command) -> call $aggregate->method` approach,
but we are lazy. We don't want to touch the port each time we add a new command to the system. Conventions work great to get around the issue.

**An aggregate root should have a method named like the command, whereby command name is derived from its class name without namespace. The first letter of the name is lowercase.**

Looking at `Building` methods, it's exactly what we already have in place ;) We just need to implement the convention in the port.

`src/Infrastructure/Flavour/EventSourcedAggregatePort.php`

```php
<?php
declare(strict_types=1);

namespace App\Infrastructure\Flavour;

use App\Model\Base\AggregateRoot;
use Prooph\EventMachine\Runtime\Oop\Port;

final class EventSourcedAggregatePort implements Port
{
    /* ... */

    /**
     * @param mixed $aggregate
     * @param mixed $customCommand
     * @param null|mixed $context
     */
    public function callAggregateWithCommand($aggregate, $customCommand, $context = null): void
    {
        $commandNameParts = \explode('\\', \get_class($customCommand));
        $handlingMethod = \lcfirst(\array_pop($commandNameParts));
        $aggregate->{$handlingMethod}($customCommand, $context);
    }

    /* ... */
}

```

Low hanging fruits, right? But the Event Machine Aggregate Description is broken! Handle and apply functions are no longer callable (except aggregate factory),
because they are instance methods now. To get around the issue, we can replace the definition with a `FlavourHint`.

`src/Api/Aggregate.php`

```php
<?php
declare(strict_types=1);

namespace App\Api;

use App\Model\Building;
use Prooph\EventMachine\EventMachine;
use Prooph\EventMachine\EventMachineDescription;
use Prooph\EventMachine\Runtime\Oop\FlavourHint;

class Aggregate implements EventMachineDescription
{
    const BUILDING = 'Building';

    /**
     * @param EventMachine $eventMachine
     */
    public static function describe(EventMachine $eventMachine): void
    {
        $eventMachine->process(Command::ADD_BUILDING)
            ->withNew(self::BUILDING)
            ->identifiedBy(Payload::BUILDING_ID)
            ->handle([Building::class, 'add'])
            ->recordThat(Event::BUILDING_ADDED)
            ->apply([FlavourHint::class, 'useAggregate']);

        $eventMachine->process(Command::CHECK_IN_USER)
            ->withExisting(self::BUILDING)
            ->handle([FlavourHint::class, 'useAggregate'])
            ->recordThat(Event::USER_CHECKED_IN)
            ->apply([FlavourHint::class, 'useAggregate'])
            ->orRecordThat(Event::DOUBLE_CHECK_IN_DETECTED)
            ->apply([FlavourHint::class, 'useAggregate']);

        $eventMachine->process(Command::CHECK_OUT_USER)
            ->withExisting(self::BUILDING)
            ->handle([FlavourHint::class, 'useAggregate'])
            ->recordThat(Event::USER_CHECKED_OUT)
            ->apply([FlavourHint::class, 'useAggregate'])
            ->orRecordThat(Event::DOUBLE_CHECK_OUT_DETECTED)
            ->apply([FlavourHint::class, 'useAggregate']);
    }
}

```

{.alert .alert-warning}
That's a bit of a drawback of the **OopFlavour**. It relies less on Event Machine, but Event Machine still wants to make
sure that you don't forget to handle a command or apply an event (handle and apply definition is mandatory). With the
`FlavourHint` we basically tell Event Machine: "Don't worry, we know what we're doing!". It's a small extra step, but trust
me, it still saves you time. Forgetting to add a route for a message to some config or have a typo somewhere is one of the
most silly bugs that can cost you hours for nothing!

## Aggregate State

One method left in the port: `serializeAggregate()`. This method is only important when using aggregate projections.
Let's check the Projection Description:

`src/Api/Projecion.php`

```php
<?php

declare(strict_types=1);

namespace App\Api;

use App\Infrastructure\Projector\UserBuildingList;
use Prooph\EventMachine\EventMachine;
use Prooph\EventMachine\EventMachineDescription;
use Prooph\EventMachine\Persistence\Stream;

class Projection implements EventMachineDescription
{
    const USER_BUILDING_LIST = 'user_building_list';

    /**
     * @param EventMachine $eventMachine
     */
    public static function describe(EventMachine $eventMachine): void
    {
        $eventMachine->watch(Stream::ofWriteModel())
            ->withAggregateProjection(Aggregate::BUILDING);

        $eventMachine->watch(Stream::ofWriteModel())
            ->with(self::USER_BUILDING_LIST, UserBuildingList::class)
            ->filterEvents([
                Event::USER_CHECKED_IN,
                Event::USER_CHECKED_OUT,
            ]);
    }
}

```

Ok, we do use it, so we have to implement the port method, otherwise we could throw a `\BadMethodCallException` and are done.

{.alert .alert-danger}
A word about aggregate projection: It's a default projection provided by Event Machine for rapid application development.
With an aggregate projection you can get very close to RAD frameworks that work with relational databases and an ORM.
The main difference is, with Event Machine you use CQRS / ES from the beginning. However, using aggregate
state as a read model couples the aggregate with read concerns. It cannot simply change its internal state without
the risk of breaking a query. The good news is, that you can remove that coupling at any point in time by turning
an aggregae projection into a dedicated projection like the `USER_BUILDING_LIST` projection. You can start simple,
but make a cut whenever things start to look messy.

A simple `toArray()` on the aggregate is sufficient. We add it to the `AggregateRoot` interface to enforce its implementation.

`src/Model/Base/AggregateRoot.php`

```php
<?php
declare(strict_types=1);

namespace App\Model\Base;

interface AggregateRoot
{
    /**
     * @return DomainEvent[]
     */
    public function popRecordedEvents(): array;

    public function apply(DomainEvent $event): void;

    public function toArray(): array;
}

```

`Building` can call the `toArray` method of `Building\State` ...

`src/Model/Building.php`

```php
<?php
declare(strict_types=1);

namespace App\Model;

use App\Model\Base\AggregateRoot;
use App\Model\Base\EventSourced;
use App\Model\Building\Command\AddBuilding;
use App\Model\Building\Command\CheckInUser;
use App\Model\Building\Command\CheckOutUser;
use App\Model\Building\Event\BuildingAdded;
use App\Model\Building\Event\DoubleCheckInDetected;
use App\Model\Building\Event\DoubleCheckOutDetected;
use App\Model\Building\Event\UserCheckedIn;
use App\Model\Building\Event\UserCheckedOut;

final class Building implements AggregateRoot
{
    use EventSourced;

    /**
     * @var Building\State
     */
    private $state;

    /* ... */

    public function toArray(): array
    {
        return $this->state->toArray();
    }
}


```

... and the `Oop\Port` does the same:

`src/Infrastructure/Flavour/EventSourcedAggregatePort.php`

```php
<?php
declare(strict_types=1);

namespace App\Infrastructure\Flavour;

use App\Model\Base\AggregateRoot;
use Prooph\EventMachine\Runtime\Oop\Port;

final class EventSourcedAggregatePort implements Port
{
    /* ... */

    /**
     * @param mixed $aggregate
     * @return array
     */
    public function serializeAggregate($aggregate): array
    {
        if(!$aggregate instanceof AggregateRoot) {
            throw new \RuntimeException(
                sprintf("Cannot serialize aggregate. Given aggregate is not an instance of %s. Got %s",
                    AggregateRoot::class,
                    (is_object($aggregate)? get_class($aggregate) : gettype($aggregate))
                )
            );
        }

        return $aggregate->toArray();
    }

    /* ... */
}

```

{.alert .alert-info}
Of course, you can use a totally different serialization strategy. Organising aggregate state in a single immutable state object is also
only a suggestion. Do whatever you like or don't use aggregate projections at all and don't implement the functionality. It's your choice!

## Activate OopFlavour

As a last step (before looking at the tests 🙈) we should activate the **OopFlavour** in the `ServiceFactory`:

`src/Service/ServiceFactory.php`

```php
<?php
namespace App\Service;

use App\Infrastructure\Flavour\AppMessagePort;
use Prooph\EventMachine\Runtime\Flavour;
use Prooph\EventMachine\Runtime\FunctionalFlavour;
/* ... */

final class ServiceFactory
{
    use ServiceRegistry;

    /**
     * @var ArrayReader
     */
    private $config;

    /**
     * @var ContainerInterface
     */
    private $container;

    public function __construct(array $appConfig)
    {
        $this->config = new ArrayReader($appConfig);
    }

    public function setContainer(ContainerInterface $container): void
    {
        $this->container = $container;
    }

    //Flavour
    public function flavour(): Flavour
    {
        return $this->makeSingleton(Flavour::class, function () {
            return new OopFlavour(
                new EventSourcedAggregatePort(),
                new FunctionalFlavour(new AppMessagePort())
            );
        });
    }

    /* ... */
}

```

As stated at the beginning, the **OopFlavour** uses the **FunctionalFlavour** mainly to make use of custom message handling.

## Fixing tests

At least the `BuildingTest` should fail after latest changes. Let's see if we need to work some extra hours or can go out to have a beer with a friend:

```bash
docker-compose run php php vendor/bin/phpunit
```

As expected, `BuildingTest` is broken, but should be easy to fix.
First, we need to change the Flavour in the test suite as well.

`tests/BaseTestCase.php`

```php
<?php
declare(strict_types=1);

namespace AppTest;

use App\Api\Event;
use App\Infrastructure\Flavour\AppMessagePort;
use App\Infrastructure\Flavour\EventSourcedAggregatePort;
use App\Model\Base\AggregateRoot;
use App\Model\Base\DomainEvent;
use PHPUnit\Framework\TestCase;
use Prooph\EventMachine\Container\ContainerChain;
use Prooph\EventMachine\Container\EventMachineContainer;
use Prooph\EventMachine\EventMachine;
use Prooph\EventMachine\Messaging\Message;
use Prooph\EventMachine\Messaging\MessageBag;
use Prooph\EventMachine\Runtime\Flavour;
use Prooph\EventMachine\Runtime\FunctionalFlavour;
use Prooph\EventMachine\Runtime\Oop\FlavourHint;
use Prooph\EventMachine\Runtime\OopFlavour;

class BaseTestCase extends TestCase
{
    /**
     * @var EventMachine
     */
    protected $eventMachine;

    /**
     * @var Flavour
     */
    protected $flavour;

    protected function setUp()
    {
        $this->eventMachine = new EventMachine();
        $this->flavour = new OopFlavour(
            new EventSourcedAggregatePort(),
            new FunctionalFlavour(new AppMessagePort())
        );

        $config = include __DIR__ . '/../config/autoload/global.php';

        foreach ($config['event_machine']['descriptions'] as $description) {
            $this->eventMachine->load($description);
        }

        $this->eventMachine->initialize(
            new ContainerChain(
                new FlavourContainer($this->flavour),
                new EventMachineContainer($this->eventMachine)
            )
        );
    }

    /* ... */

    protected function applyEvents(AggregateRoot $aggregateRoot)
    {
        array_walk($aggregateRoot->popRecordedEvents(), function (DomainEvent $event) use ($aggregateRoot) {
            $this->flavour->callApplySubsequentEvent(
                [FlavourHint::class, 'useAggregate'],
                $aggregateRoot,
                new MessageBag(
                    Event::nameOf($event),
                    MessageBag::TYPE_EVENT,
                    $event
                )
            );
        });
    }
}

```

With a little test helper `applyEvents` we can use the Flavour to apply recorded events.

When testing event sourced objects we cannot simply prepare state and call a function.
We have to invoke all command handling functions needed to get the aggregate into desired state.
That's the change we have to make in `BuildingTest`:

```php
<?php
declare(strict_types=1);

namespace AppTest\Model;

use App\Api\Event;
use App\Api\Payload;
use AppTest\BaseTestCase;
use Ramsey\Uuid\Uuid;
use App\Model\Building;

class BuildingTest extends BaseTestCase
{
    private $buildingId;
    private $buildingName;
    private $username;

    protected function setUp()
    {
        $this->buildingId = Uuid::uuid4()->toString();
        $this->buildingName = 'Acme Headquarters';
        $this->username = 'John';

        parent::setUp();
    }

    /**
     * @test
     */
    public function it_checks_in_a_user()
    {
        //Prepare expected aggregate state
        $addBuilding = Building\Command\AddBuilding::fromArray([
            Payload::BUILDING_ID => $this->buildingId,
            Payload::NAME => $this->buildingName
        ]);
        /** @var Building $building */
        $building = Building::add($addBuilding);

        //New test helper to apply recorded events
        $this->applyEvents($building);

        $command = Building\Command\CheckInUser::fromArray([
            Payload::BUILDING_ID => $this->buildingId,
            Payload::NAME => $this->username,
        ]);

        $building->checkInUser($command);

        $events = $building->popRecordedEvents();

        $this->assertRecordedEvent(Event::USER_CHECKED_IN, [
            Payload::BUILDING_ID => $this->buildingId,
            Payload::NAME => $this->username
        ], $events);
    }

    /**
     * @test
     */
    public function it_detects_double_check_in()
    {
        //Prepare expected aggregate state
        $addBuilding = Building\Command\AddBuilding::fromArray([
            Payload::BUILDING_ID => $this->buildingId,
            Payload::NAME => $this->buildingName
        ]);
        /** @var Building $building */
        $building = Building::add($addBuilding);

        $this->applyEvents($building);

        $checkInUser = Building\Command\CheckInUser::fromArray([
            Payload::BUILDING_ID => $this->buildingId,
            Payload::NAME => $this->username,
        ]);

        $building->checkInUser($checkInUser);

        $this->applyEvents($building);

        //Test double check in

        $command = Building\Command\CheckInUser::fromArray([
            Payload::BUILDING_ID => $this->buildingId,
            Payload::NAME => $this->username,
        ]);

        $building->checkInUser($command);

        $events = $building->popRecordedEvents();

        //Another test helper to assert that list of recorded events contains given event
        $this->assertRecordedEvent(Event::DOUBLE_CHECK_IN_DETECTED, [
            Payload::BUILDING_ID => $this->buildingId,
            Payload::NAME => $this->username
        ], $events);

        //And the other way round, list should not contain event with given name
        $this->assertNotRecordedEvent(Event::USER_CHECKED_IN, $events);
    }
}

```

## Wrap Up

{.alert .alert-success}
At this point the tutorial ends. Thank you for taking the tour through the world of CQRS and Event Sourcing with Event Machine.
We started our tour with a rapid development approach. Event Machine really shines here. The skeleton application is preconfigured including
some best practices like splitting Event Machine Descriptions by functionality. We learned how to react on domain events and how
to project them into a read model, that we can access using queries and finders. All that with a minimum of boilerplate. Finally, Event Machine
Flavours gave us a way to write more explicit code and harden the domain model. Every team can find its own style by mixing Flavours, conventions
and serialization techniques.

**What's next?**

You can start to work on your own project. Event Machine docs cover advanced topics and a lot more details, but get some practice first and revisit them every now and then.









