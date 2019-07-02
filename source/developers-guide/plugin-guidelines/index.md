---
layout: default
title: Plugin guidelines
github_link: developers-guide/plugin-guidelines/index.md
indexed: true
menu_title: Plugin guidelines
menu_order: 50
group: Developer Guides
subgroup: Developing plugins
---

<div class="toc-list"></div>

# Plugin guidelines

There are some issues we come across frequently, when reviewing plugins for the
community store. This document is intended to help plugin developers implement
plugins according to
[our quality guidelines](https://docs.shopware.com/de/plugin-standard-community-store).
We hope the examples provided here will be helpful, please feel free to submit
corrections and suggestions to our
[devdocs repository](https://github.com/shopware/devdocs/)
on Github.

## Internationalisation

### Snippets

If your plugin contains user-facing text content, we suggest that you should make
use of the snippet system provided by Shopware. When using the snippet system,
the text content can be easily distributed, using `*.ini` files. Also, every
text contained in a snippet is editable by shop administrators and translatable
as well. You can use this article in
[our documentation](https://developers.shopware.com/designers-guide/snippets/)
for guidance on how and where to use snippets.

### Plugin metadata

For a plugin to be ready for the community store, the text content of the
`plugin.xml` and similar files needs to be translated into the languages
provided by the plugin. This is an example of a correctly translated element in
a `config.xml`-file:

```xml
<?xml version="1.0" encoding="utf-8"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="../../../../engine/Shopware/Components/Plugin/schema/config.xsd">

    <elements>
        <element type="boolean">
            <name>examplesetting</name>
            <label>This is an example</label>
            <label lang="de">Das hier ist ein Beispiel</label>
            <value>1</value>
        </element>
    </elements>

</config>
```

Note that there is a label for each language provided by the plugin, in this
case english and german.

## Logging 101

When you need to inform the user or administrators about a noteworthy event
regarding your plugin (like an error, or if you fall back to a default setting,
...) please use the plugin-logger provided by Shopware. This class is present in
the Symfony DIC with the ID `pluginlogger`. Depending on the class you're
writing, there are several methods to access the `pluginlogger`.

Most of the time, you should inject the pluginlogger into the classes that need
it:

`SwagExample/Resources/services.xml`:
```xml
<?xml version="1.0" ?>

<container xmlns="http://symfony.com/schema/dic/services"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://symfony.com/schema/dic/services http://symfony.com/schema/dic/services/services-1.0.xsd">

    <services>
        <service id="swag_example.some_subscriber" class="SwagExample\Subscriber\SomeSubscriber">
            <argument id="pluginlogger" />
        </service>
    </services>
</container>
```

`SwagExample/Subscriber/SomeSubscriber.php`:
```php
<?php

namespace SwagExample\Subscriber;

use Shopware\Components\Logger;

class SomeSubscriber
{
    /**
     * @var Logger $logger
     */
    private $logger;

    /**
     * SomeSubscriber constructor.
     */
    public function __construct(Logger $logger)
    {
        $this->logger = $logger;
    }

    /**
     * This method writes a message to the plugin error log.
     */
    public function someMethod(): void
    {
        $this->logger->addError('Insert helpful error message here');
    }
}
```

In case you're writing a small plugin which only consists of a plugin base
class, you may also get the logger directly from the DIC:

```php
$logger = $this->container->get('pluginlogger');
```

### Plugin-specific logger

If your plugins minimum Shopware version is greater than or equal to v5.6.0, you
can and should use the provided plugin-specific logger. This is a service in the
DIC as well. Its ID is a combination of the plugin's service prefix (by default
this is the plugin's name in `snake_case`) and `.logger`. So for a plugin called
`SwagExample`, the service-ID of the plugin-specific logger would be
`swag_example.logger`. You can read more about the plugin-specific logger in the
corresponding
[upgrade document](https://github.com/shopware/shopware/blob/5.6/UPGRADE-5.6.md#plugin-specific-logger).

## Plugin installation routine

When implementing a plugin's `install`-method, many developers choose to add a
call to clear caches. We advise not to do this, and instead clear the necessary
caches when the `activate`-method is called. Also please consider which caches
actually need to be cleared for your plugin to work, since regenerating caches
may cause a high load on the server.

## Plugin uninstallation routine

When implementing a plugin's `uninstall`-method, please be careful and don't
delete any plugin data when the plugin is reinstalled. You can find out if the
user wants to keep the plugins data by examining the `$context`:

```php
/**
 * {@inheritdoc}
 */
public function uninstall(UninstallContext $context): void
{
    $this->secureUninstall();

    if (!$context->keepUserData()) {
        $this->removeAllTables();
        $this->someOtherDestructiveMethod();
    }
}
```

When the plugin is reinstalled, the `keepUserData` method returns `true` as
well, so the uninstall method is safe when implemented in the way shown above.

## Adding and Removing attributes

For guidance on how to add and remove attributes, please read our
[documentation](https://developers.shopware.com/developers-guide/attribute-system/#delete-an-existing-attribute).

## Clearing the cache when configuration changes

Sometimes when a value is updated via a plugin's configuration form, the cache
needs to be cleared. The following is a simple example subscriber, which takes
care of this:

```php
<?php

namespace SwagExample\Subscriber;

use Enlight\Event\SubscriberInterface;
use Shopware\Components\CacheManager;
use Shopware_Controllers_Backend_Config;

class SomeSubscriber implements SubscriberInterface
{
    /**
     * @var string
     */
    private $pluginName;

    /**
     * @var CacheManager
     */
    private $cacheManager;

    /**
     * SomeSubscriber constructor.
     */
    public function __construct(string $pluginName, CacheManager $cacheManager)
    {
        $this->pluginName = $pluginName;
        $this->cacheManager = $cacheManager;
    }

    public static function getSubscribedEvents(): array
    {
        return [
            'Enlight_Controller_Action_PostDispatchSecure_Backend_Config' => 'onPostDispatchConfig'
        ];
    }

    public function onPostDispatchConfig(\Enlight_Event_EventArgs $args): void
    {
        /** @var Shopware_Controllers_Backend_Config $subject */
        $subject = $args->get('subject');
        $request = $subject->Request();

        // If this is a POST-Request, and affects our plugin, we may clear the config cache
        if($request->isPost() && $request->getParam('name') === $this->pluginName) {
            $this->cacheManager->clearByTag(CacheManager::CACHE_TAG_CONFIG);
        }
    }
}
```

## Adding E-Mail-Templates

Plugins have the ability to create additional E-Mail-Templates using the
Mail-model. The handling of templates is different in this case, since the
template contents are saved in the model and written to the database accordingly.

```php
<?php

namespace SwagExample;

use Shopware\Components\Model\ModelManager;
use Shopware\Components\Plugin;
use Shopware\Components\Plugin\Context\InstallContext;
use Shopware\Components\Plugin\Context\UninstallContext;
use Shopware\Models\Mail\Mail;

class SwagExample extends Plugin
{
    public const MAIL_TEMPLATE_NAME = 'MyTestMail';

    public function install(InstallContext $context): void
    {
        $this->installMailTemplate();
    }

    public function uninstall(UninstallContext $context): void
    {
        $this->uninstallMailTemplate();
    }

    /**
     * installMailTemplate takes care of creating the new E-Mail-Template
     */
    private function installMailTemplate(): void
    {
        $entityManager = $this->container->get('models');
        $mail = new Mail();

        // After creating an empty instance, some technical info is set
        $mail->setName(self::MAIL_TEMPLATE_NAME);
        $mail->setMailtype(Mail::MAILTYPE_USER);

        // Now the templates basic information can be set
        $mail->setSubject($this->getSubject());
        $mail->setContent($this->getContent());
        $mail->setContentHtml($this->getContentHtml());

        /**
         * Finally the new template can be persisted.
         *
         * transactional is a helper method which wraps the given function
         * in a transaction and executes a rollback if something goes wrong.
         * Any exception that occurs will be thrown again and, since we're in
         * the install method, shown in the backend as a growl message.
         */
        $entityManager->transactional(static function ($em) use ($mail) {
            /** @var ModelManager $em */
            $em->persist($mail);
        });
    }

    /**
     * uninstallMailTemplate takes care of removing the plugin's E-Mail-Template
     */
    private function uninstallMailTemplate(): void
    {
        $entityManager = $this->container->get('models');
        $repo = $entityManager->getRepository(Mail::class);

        // Find the mail-type we created
        $mail = $repo->findOneBy(['name' => self::MAIL_TEMPLATE_NAME]);

        $entityManager->transactional(static function ($em) use ($mail) {
            /** @var ModelManager $em */
            $em->remove($mail);
        });
    }

    private function getSubject(): string
    {
        return 'Default Subject';
    }

    private function getContent(): string
    {
        /**
         * Notice the string:{...} in the include's file-attribute.
         * This causes the referenced config value to be loaded into
         * a string and passed on as the template's content. This works
         * because the file-attribute can accept any template resource
         * which includes paths to files and several other types as well.
         * For more information about template resources, have a look here:
         * https://www.smarty.net/docs/en/resources.string.tpl
         */
        return <<<'EOD'
{include file="string:{config name=emailheaderplain}"}

{* Content *}

{include file="string:{config name=emailfooterplain}"}
EOD;
    }

    private function getContentHtml(): string
    {
        return <<<'EOD'
{include file="string:{config name=emailheaderhtml}"}

{* Content *}

{include file="string:{config name=emailfooterhtml}"}
EOD;
    }
}
```

## Adding URLs to the sitemap.xml

Since Shopware v5.5, new URLs may be added to the `sitemap.xml` by registering a
service with a specific tag. The general principle is described in our
[Upgrade Guide](https://developers.shopware.com/developers-guide/shopware-5-upgrade-guide-for-developers/#sitemap).

The registration of such a service in a plugins `services.xml` could look like
this:

```xml
<service id="swag_example.url_provider" class="SwagExample\Components\UrlProvider\MyUrlProvider">
    <tag name="sitemap_url_provider" />
</service>
```

## Validating user input

Inadequate or even missing validation of user input is a security risk. If your
plugin doesn't validate user input, it is not eligible for the community store.
Since input validation is not a Shopware-specific issue and implementing it can
be dependent on the business case, we can only provide broad orientation here.
If you'd like to read more about input validation and how to implement it, have
a look at the
[cheatsheets released by the OWASP](https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Input_Validation_Cheat_Sheet.md).

