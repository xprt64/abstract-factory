# Abstract factory + dependency injector + dependency injection container (in PHP)

## Problem
You must create a lot of objects that create a lot of other objects that have a lot of dependencies but you don't want to pass them all the way down. 

## Solution
Create an _abstract factory_ that use a _dependency injector_ to detect and inject object dependencies. 
Dependency injector uses a _dependency injection container_ to resolve dependencies. 
Declare dependencies using [`interfaces`](http://php.net/manual/ro/language.oop5.interfaces.php). 
Use [`traits`](http://php.net/manual/ro/language.oop5.traits.php) to implement those `interfaces`.

## Examples

In these examples I will copy/paste some of my work from _Gica_ framework

### The abstract factory
The [single responsability](https://en.wikipedia.org/wiki/Single_responsibility_principle) of this class is object creation (using the `new` operator).

The interface is this:
```php
namespace Gica\Interfaces\Dependency;

interface AbstractFactory
{
    public function createObject($objectClass, $constructorArguments = []);
}

```

And the production implementation is this:

```php
namespace Gica\Dependency;


class AbstractFactory implements \Gica\Interfaces\Dependency\AbstractFactory, \Gica\Interfaces\Dependency\WithDependencyInjector
{
    use WithDependencyInjector;

    /**
     * @param string $objectClass
     * @param array $constructorArguments
     * @return object instanceof $class
     */
    public function createObject($objectClass, $constructorArguments = [])
    {
        $instance = new $objectClass(...$constructorArguments);

        $this->getDependencyInjector()->resolveDependencies($instance);

        return $instance;
    }
}
```
In tests you would replace the implementation with a mocked abstract factory.

### The depedency injector
The [single responsability](https://en.wikipedia.org/wiki/Single_responsibility_principle) of this class is to detect and inject (but not resolve) dependecies. 
I know that the word _and_ would suggest multiple responsabilities but the two actions (_detect_ and _inject_) are very tied together so must stay together; they change together so SRP is not broken.

The interface is this:

```php
namespace Gica\Interfaces\Dependency;

interface DependencyInjector extends WithDependencyContainer
{
    public function resolveDependencies($instance);
}
```

And one implementation is this:

```php
namespace Gica\Dependency;

class DependencyInjector implements \Gica\Interfaces\Dependency\DependencyInjector
{
    use \Gica\Traits\WithDependencyContainer;

    public function resolveDependencies($instance)
    {
        $sm = $this->getDependencyInjectionContainer();

        if ($instance instanceof \Gica\Interfaces\WithAuthenticator) {
            $instance->setAuthenticator($sm->get(\Gica\Interfaces\Authentication\Authenticator::class));
        }
        if ($instance instanceof \Gica\Interfaces\WithPdo) {
            $instance->setPdo($sm->get(\Gica\SqlQuery\Connection::class));
        }
        if ($instance instanceof \Gica\Interfaces\WithRepository) {
            $instance->setRepository($sm->get(\Gica\Interfaces\Object\Repository::class));
        }
        if ($instance instanceof \Gica\Interfaces\Event\WithEventManager) {
            $instance->setEventManager($sm->get(\Gica\Interfaces\Event\EventManager::class));
        }
        if ($instance instanceof \Gica\Interfaces\Object\Descriptor\WithDescriptorsManagerInterface) {
            $instance->setDescriptorsManager($sm->get(\Gica\Interfaces\Object\Descriptor\DescriptorsManager::class));
        }
        if ($instance instanceof \Gica\Interfaces\Object\Descriptor\WithObjectTypeMapper) {
            $instance->setObjectTypeMapper($sm->get(\Gica\Interfaces\Object\Descriptor\ObjectTypeMapper::class));
        }
        if ($instance instanceof \Gica\Interfaces\Dependency\WithDependencyContainer) {
            $instance->setDependencyInjectionContainer($sm);
        }
        if ($instance instanceof \Gica\Interfaces\Dependency\WithDependencyInjector) {
            $instance->setDependencyInjector($sm->get(\Gica\Interfaces\Dependency\DependencyInjector::class));
        }
        if ($instance instanceof \Gica\Interfaces\Dependency\WithAbstractFactory) {
            $instance->setAbstractFactory($sm->get(\Gica\Interfaces\Dependency\AbstractFactory::class));
        }
        if ($instance instanceof \Gica\Interfaces\WithFileStorage) {
            $instance->setFileStorage($sm->get(\Gica\Interfaces\FileStorage::class));
        }
    }
}
```

Please notice that dependencies are detected by naming convention: all dependencies are declared using interfaces that start with the word __With__, like this one `\Gica\Interfaces\WithFileStorage`. 
Every time you create an interface that declare a dependency you must update only this file.

### The dependency injection container
Fortunatelly, there is an [standard interface](https://github.com/container-interop/container-interop/blob/master/src/Interop/Container/ContainerInterface.php)  from [PHP-FIG](http://www.php-fig.org/psr/) for this container: `\Interop\Container\ContainerInterface`

### Dependency declaring interface
Every interface that declares a dependency should be named based on convention. My convention is to name them starting with the word __With__. Others prefer to name them with __AwareInterface__. Any convention that reveals this intention should be fine. Below is an example of such interface:

```php
namespace Gica\Interfaces;

interface WithAuthenticator
{
    /**
     * @return \Gica\Interfaces\Authentication\Authenticator
     */
    public function getAuthenticator();

    /**
     * @param \Gica\Interfaces\Authentication\Authenticator
     * @return static
     */
    public function setAuthenticator(\Gica\Interfaces\Authentication\Authenticator $authenticator);
}
```
The _getter_ is used by the dependent class and the _setter_ is used by the _dependency injector_.

In order to avoid code duplication you should use a `trait` to implement this interface, like this one:
```php
namespace Gica\Traits;

trait WithAuthenticator
{
    /**
     *
     * @var \Gica\Interfaces\Authentication\Authenticator
     */
    protected $authenticator;

    /**
     * @return \Gica\Interfaces\Authentication\Authenticator
     */
    public function getAuthenticator()
    {
        return $this->authenticator;
    }

    /**
     * @param \Gica\Interfaces\Authentication\Authenticator $authenticator
     * @return static
     */
    public function setAuthenticator(\Gica\Interfaces\Authentication\Authenticator $authenticator)
    {
        $this->authenticator = $authenticator;

        return $this;
    }
}
```

### The dependent class
In the dependent class, you must implement all the dependency declaring interfaces and use `trait`s to support those interfaces. Below is an example:

```php
namespace Web\Helper;

class SomeHelper implements \Gica\Interfaces\WithAuthenticator, \Gica\Interfaces\Dependency\WithAbstractFactory
{
    use \Gica\Traits\WithAuthenticator, 
        \Gica\Dependency\WithAbstractFactory;

    public function helpAction()
    {
       $someObject = $this->getAbstractFactory()->createObject(\Gika\Interfaces\SomeObject::class);

       $result = $someObject->computeSomeValue($this->getAuthenticator()->getAuthenticatedIdentityId());
       
       return $result;
    }
}

```
Please notice that `SomeHelper` class declares that it depends on two other classes to do its job: `Authenticator` to get current identity ID and `AbstractFactory` to create some other object (that could have other dependencies as well). Using an abstract factory, `SomeHelper` doesn't care what are the dependencies of `$someObject`.

### Client code

In this example we assume an dependency injection container instance in `$container`.

Somewhere in the client code:

```php
$abstractFactory = $container->get(\Gica\Interfaces\Dependency\AbstractFactory::class);

$someHelper = $abstractFactory->createObject(\Web\Helper\SomeHelper::class);

echo $someHelper->helpAction();
```

Notice that dependencies are hidden, and we can focus on the main bussiness. My client code doesn't care or know that `$someHelper` need an `Authenticator` or that `helpAction` need an `SomeObject` to do its work;

In the background a lot of things happen, a lot of dependencies are detected, resolved and injected. 
Notice that I don't use the `new` operator to create `$someObject`. The responsability of actual creation of the object is passed to the `AbstractFactory`

# Notes
The `ObjectFactory` is NOT this [abstract factory](https://en.wikipedia.org/wiki/Abstract_factory_pattern).
