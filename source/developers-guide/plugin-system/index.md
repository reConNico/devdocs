---
layout: default
title: The 5.2 Plugin System
github_link: developers-guide/plugin-system/index.md
indexed: true
shopware_version: 5.2
---

<div class="alert alert-warning">
This document is work in progress and not finished yet.
Please feel free to open a pull request on github to extend parts of this document.
</div>

<div class="toc-list"></div>

## Parallel mode
The new plugin system runs fully in parallel to the "legacy" plugin system.

## Directory Structure

The 5.2 Plugins are located in the `custom/plugins/` directory. There is no seperation in `frontend`, `core` or `backend` like in the "legacy" plugin system.

## Plugin Name

The plugin name should always be prefixed with your developer prefix so it's unique in the Shopware universe.
To submit plugins to the [shopware store](http://store.shopware.com/) you have to obtain your developer prefix in the [Shopware Account](https://account.shopware.com).

In the following examples the developer prefix "Swag" will be used (short for shopware AG).

## Minimal Plugin Example

The most minimal Plugin is just a directory and one bootstrap file.
The directory must be named after the plugin name. The bootstrap file is called `SwagSloganOfTheDay.php`:

### Directory structure

```
SwagSloganOfTheDay
└──SwagSloganOfTheDay.php
```

### Plugin Bootstrap file

The Bootstrap `SwagSloganOfTheDay.php` must be namespaced with your plugin name and extend `\Shopware\Components\Plugin`:

```
<?php
namespace SwagSloganOfTheDay;

class SwagSloganOfTheDay extends \Shopware\Components\Plugin
{
}
```


### Install and activate

Now the plugin can be installed using the Shopware [CLI Commands](/developers-guide/shopware-5-cli-commands/) or the Plugin Manager in the backend.

```
$ php ./bin/console sw:plugin:refresh
Successfully refreshed
```

```
$ php ./bin/console sw:plugin:install --activate SwagSloganOfTheDay
Plugin SwagSloganOfTheDay has been installed successfully.
Plugin SwagSloganOfTheDay has been activated successfully.
```

At this point the plugin has no functionality at all.


## Pluginbootstrap as Event Subscriber

The Pluginbootstrap implements `\Enlight\Event\SubscriberInterface` so it can be used as a [Event Subscriber](/developers-guide/event-guide/#subscribers).

```
<?php
namespace SwagSloganOfTheDay;

class SwagSloganOfTheDay extends \Shopware\Components\Plugin
{
    public static function getSubscribedEvents()
    {
        return [
            'Enlight_Controller_Front_RouteStartup' => 'onRouteStartup'
        ];
    }

    public function onRouteStartup(\Enlight_Controller_EventArgs $args)
    {
        die("A rolling stone gathers no moss");
    }
}
```

### Access to the DI-Container

Inside the plugin bootstrap the DI-Container can be accessed with the `$this->container` property:

```php
    public function onRouteStartup(\Enlight_Controller_EventArgs $args)
    {
        $conn = $this->container->get('dbal_connection');
        $conn->.... // do some query
    }
```

## Autoloading

The plugin namespace is registered as a [PSR-4](http://www.php-fig.org/psr/psr-4/) Autoloading prefix.
For example the class `\SwagSloganOfTheDay\Log\Writer` will be loaded from file `SwagSloganOfTheDay/Log/Writer.php`.

## Plugin Install / Update

During plugin installation / deinstallation / update / activate / deactivate a method on the plugin bootstrap is called that can optionally be overwritten.

```
<?php
namespace SwagSloganOfTheDay;

use Shopware\Components\Plugin\Context\ActivateContext;
use Shopware\Components\Plugin\Context\DeactivateContext;
use Shopware\Components\Plugin\Context\InstallContext;
use Shopware\Components\Plugin\Context\UpdateContext;
use Shopware\Components\Plugin\Context\UninstallContext;

class SwagSloganOfTheDay extends \Shopware\Components\Plugin
{
    public function install(InstallContext $context)
    {
    }

    public function update(UpdateContext $context)
    {
    }

    public function activate(ActivateContext $context)
    {
    }

    public function deactivate(DeactivateContext $context)
    {
    }

    public function uninstall(UninstallContext $context)
    {
    }
}
```


## Container Configuration

The [Symfony DependencyInjection Component](http://symfony.com/doc/current/components/dependency_injection/introduction.html)

The container configuration is the main extension point for shopware plugins.
In this configuration new services can be defined, core services can be decorated or replaced or functionality can be added to the system.

```
SwagSloganOfTheDay
├── Resources
│   └── services.xml
├──SloganPrinter.php
└──SwagSloganOfTheDay.php
```

```
<?php
namespace SwagSloganOfTheDay;

class SwagSloganOfTheDay extends \Shopware\Components\Plugin
{
    public static function getSubscribedEvents()
    {
        return [
            'Enlight_Controller_Front_RouteStartup' => 'onRouteStartup'
        ];
    }

    public function onRouteStartup(\Enlight_Controller_EventArgs $args)
    {
        $sloganPrinter = $this->container->get('swag_slogan_of_the_day.slogan_printer');
        $sloganPrinter->print();
    }
}
```

`SwagSloganOfTheDay/Resources/services.xml`

```
<?xml version="1.0" ?>

<container xmlns="http://symfony.com/schema/dic/services"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://symfony.com/schema/dic/services http://symfony.com/schema/dic/services/services-1.0.xsd">

    <services>
        <service id="swag_slogan_of_the_day.slogan_printer" class="SwagSloganOfTheDay\SloganPrinter">
            <argument type="service" id="dbal_connection" />
        </service>
    </services>
</container>
```


### Extended Container Configuration

By overwriting the `build()`-method the `ContainerBuilder` can extended:

```
<?php
namespace SwagSloganOfTheDay;

use Symfony\Component\DependencyInjection\ContainerBuilder;

class SwagSloganOfTheDay extends \Shopware\Components\Plugin
{
    public function build(ContainerBuilder $container)
    {
        $container->setParameter('swag_slogan_of_the_day.plugin_dir', $this->getPath());
        $container->addCompilerPass(new SloganCompilerPass());

        parent::build($container);
    }
}
```

### Event subscriber
The new plugin system has the ability to add event subscriber by adding subscribers in the `services.xml`.

```
SwagSloganOfTheDay
├── Resources
│   └── services.xml
├──SloganPrinter.php
├──RouteSubscriber.php
└──SwagSloganOfTheDay.php
```

The `onRouteStartup` subscriber above now will be encapsulated in a subscriber class. 

`SwagSloganOfTheDay/RouteSubscriber.php`

```
<?php
namespace SwagSloganOfTheDay\Subscriber;

use Enlight\Event\SubscriberInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;

class RouteSubscriber implements SubscriberInterface
{
    private $sloganPrinter;

    public static function getSubscribedEvents()
    {
        return [
            'Enlight_Controller_Front_RouteStartup' => 'onRouteStartup'
        ];
    }
    
    public function __construct(SloganPrinter $sloganPrinter)
    {
        $this->sloganPrinter = $sloganPrinter;
    }

    public function onRouteStartup(\Enlight_Controller_EventArgs $args)
    {
        $this->sloganPrinter->print();
    }
}
```

After adding the `RouteSubscriber.php`, the subscriber can be added to the `services.xml` as a tagged service ([Symfony - Working with Tagged Services](http://symfony.com/doc/current/components/dependency_injection/tags.html)).
This allows shopware to load all event subscriber automatically so you don't need to register the subscriber manually.

`SwagSloganOfTheDay/Resources/services.xml`
```
<?xml version="1.0" ?>

<container xmlns="http://symfony.com/schema/dic/services"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://symfony.com/schema/dic/services http://symfony.com/schema/dic/services/services-1.0.xsd">

    <services>
        <service id="swag_slogan_of_the_day.subscriber.route" class="SwagSloganOfTheDay\Subscriber\Route">
            <argument type="service" id="swag_slogan_of_the_day.slogan_printer" />
            <tag name="shopware.event_subscriber" />
        </service>

        <service id="swag_slogan_of_the_day.slogan_printer" class="SwagSloganOfTheDay\SloganPrinter">
            <argument type="service" id="dbal_connection" />
        </service>
    </services>
</container>
```

## Register plugin controller with template
```php
<?php
namespace SwagControllerExample;

use Shopware\Components\Plugin;

class SwagControllerExample extends Plugin
{
    /**
     * @inheritdoc
     */
    public static function getSubscribedEvents()
    {
        return [
            'Enlight_Controller_Dispatcher_ControllerPath_Frontend_MyController' => 'registerController',
        ];
    }

    public function registerController(\Enlight_Event_EventArgs $args)
    {
        $this->container->get('Template')->addTemplateDir(
            $this->getPath() . '/Resources/views/'
        );

        return $this->getPath() . "/Controllers/Frontend/MyController.php";
    }
}
```

## Add console commands

There are two ways to add Shopware [CLI Commands](/developers-guide/shopware-5-cli-commands/).

### Implement registerCommands

You can implement the method `registerCommands()` and add commands to the Console application:

```
<?php
namespace SwagCommandExample;

use Shopware\Components\Plugin;
use Shopware\Components\Console\Application;
use SwagCommandExample\Commands\FirstCommand;
use SwagCommandExample\Commands\SecondCommand;

class SwagCommandExample extends Plugin
{
    public function registerCommands(Application $application)
    {
        $application->add(new FirstCommand());
        $application->add(new SecondCommand());
    }
}
```

### Commands as Services

As of Shopware 5.2.2 you can also register commands as a service and tag it with `console.command`:

```xml
<!-- Resources/services.xml -->
<?xml version="1.0" encoding="UTF-8" ?>
<container xmlns="http://symfony.com/schema/dic/services"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://symfony.com/schema/dic/services http://symfony.com/schema/dic/services/services-1.0.xsd">

    <services>
        <service
            id="swag_command_example.commands.first_command"
            class="SwagCommandExample\Commands\FirstCommand">
            <tag name="console.command"/>
        </service>
    </services>
</container>
```

You can read more in the Symfony Documentation: [How to Define Commands as Services](https://symfony.com/doc/2.8/cookbook/console/commands_as_services.html).

## Plugin Resources

Plugin meta data and configurations will be configured by using xml files which will be placed like in the example below.
IDE`s like PhpStorm support auto completion by default for these files if schema file location is valid.

```
SwagSloganOfTheDay
├──plugin.xml
├── Resources
│   ├── config.xml
│   └── menu.xml
└──SwagSloganOfTheDay.php
```

You can find the schema of the xml files in `engine/Shopware/Components/Plugin/schema`.
 - **config.xml:** Defines the plugin configuration form which you can access by the `Basic Settings` or in the detail window of a plugin.
 - **menu.xml:** Defines new menu items in the backend menu structure of Shopware.
 - **plugin.xml:** Defines the meta data of your plugin, i.e. label, version, compatibility or the changelog. 
 
<div class="alert alert-warning">
At the moment it is necessary that the order of the xml elements is equal to the schema file, otherwise you will receive an exception. <br/>
You can use the CLI to install the plugin with extended error messages: <code>php ./bin/console sw:plugin:install SwagSloganOfTheDay -v</code>
</div>

### Plugin Metadata 
 
```
<!-- plugin.xml -->
<?xml version="1.0" encoding="utf-8"?>
<plugin xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="https://raw.githubusercontent.com/shopware/shopware/5.2/engine/Shopware/Components/Plugin/schema/plugin.xsd">
    <label lang="de">Slogan des Tages</label>
    <label lang="en">Slogan of the day</label>

    <version>1.0.0</version>
    <link>http://example.org</link>
    <author>shopware AG</author>
    <compatibility minVersion="5.2.0" />

    <changelog version="1.0.0">
        <changes lang="de">Veröffentlichung</changes>
        <changes lang="en">Release</changes>
    </changelog>
</plugin>
```

### Plugin Configuration / Forms


Backend plugin configuration can be extended by `config.xml` file. This replaces the usage of `$this->Form()` on old `Shopware_Components_Plugin_Bootstrap`


```
<!-- Resources/config.xml -->
<?xml version="1.0" encoding="utf-8"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="https://raw.githubusercontent.com/shopware/shopware/5.2/engine/Shopware/Components/Plugin/schema/config.xsd">
    <elements>
        <element required="true" type="text">
            <name>slogan</name>
            <label lang="de">Dein Slogan</label>
            <label lang="en">Your slogan</label>
            <value>XML is fun!</value>
        </element>
    </elements>
</config>
```

Configuration is accessible by following code snippet:

```
Shopware()->Config()->getByNamespace('SwagSloganOfTheDay', 'slogan'),
```

### Backend Menu Items

```xml
<!-- Resources/menu.xml -->
<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="https://raw.githubusercontent.com/shopware/shopware/5.2/engine/Shopware/Components/Plugin/schema/menu.xsd">
	<entries>
		<entry>
			<name>SloganOfTheDay</name>
			<label lang="en">Slogan of the Days</label>
			<label lang="de">Spruch des Tages</label>
			<controller>SwagSloganOfTheDay</controller>
			<action>index</action>
			<class>sprite-metronome</class>
			<parent identifiedBy="controller">Marketing</parent>
		</entry>
	</entries>
</menu>
```

For available parent controllers take a look into table s_core_menu (column controller). For example you can use one of the following:
- Article
- Content
- Customer
- ConfigurationMenu
- Marketing

Menuitem won't be displayed if controller and action are missing.

To know which class for which icon take a look at the <a href="/designers-guide/backend-icons/">Backen icon set overview</a>.

## Access to other plugins

Other plugins can be accessed via the `getPlugins()` method of the kernel.

```php
$swagExample = Shopware()->Container()->get('kernel')->getPlugins()['SwagExample'];
$path = $swagExample->getPath();
```



## Update from legacy plugin system

Shopware recognizes whether the plugin is based on the legacy or 5.2 plugin system and moves it to the correct directory. Shopware does not support moving of extracted plugins based on the 5.2 plugin system, if they are placed in the legacy directory structure.
Further the zip archive structure changed. 

**Legacy zip structure:**
```
SwagSloganOfTheDay.zip
└──Frontend
   └──SwagSloganOfTheDay
      ├──Bootstrap.php
      └──...
```

**New 5.2 zip structure:**
```
SwagSloganOfTheDay.zip
└──SwagSloganOfTheDay
   ├──SwagSloganOfTheDay.php
   └──...
```

## Example Plugins

- <a href="https://github.com/shyim/shopware-profiler">github.com/shyim/shopware-profiler</a>
- <a href="https://github.com/bcremer/SwagModelTest">github.com/bcremer/SwagModelTest</a>
