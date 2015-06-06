---
layout: post
title:  "Speeding Up Your ZF2 Application"
date:   2015-05-11 12:00:00
categories: zf2
---



After about a year developing a Zend Framework 2 application, we decided it was time to do some optimizations. Page load times were up to several seconds on our bigger pages, and none of our pages were loading in under 2 seconds. We took a few days to profile our application and scour the various ZF2 articles out there to see what could be done to reduce the load times. We found some pretty obvious causes as well as a few inconspicuous ones. Here's a brief list of our findings, along with some steps on how to improve your ZF2 applications:
<h4><strong>Standard vs Classmap Autoloading</strong></h4>
In order to fully understand how your application is behaving, you need to use a profiler. Without profiling your code, there is no way of knowing your bottlenecks. And without knowing your bottlenecks, you're going to spend a lot of time optimizing code you don't need to.

With that in mind, our first discovery: Autoloading is slow.

After analyzing execution statistics (with Xdebug and PhpStorm), we found that the <code>Composer\Autoload\ClassLoader</code> class was responsible for 4 of the top 10 slowest functions (ordered by <em>own time</em>, which refers to the amount of time spent in that function alone). This was obviously having a huge impact on our load times. Here's how we sped up autoloading:

The following is a common block of code you'd find in your Module.php files:

{% highlight ruby %}

public function getAutoloaderConfig()
{
    return array(
        'Zend\Loader\StandardAutoloader' => array(
            'namespaces' => array(
                __NAMESPACE__ => __DIR__ . '/src/' . __NAMESPACE__
            )
        )
    );
}
{% endhighlight %}

Here, we're specifying that all of our classes are located in <code>./module/my_module/src/my_module</code>, and we let the <code>StandardAutoloader</code> generate the links between the namespace and pathname for each of our classes. This is done at runtime, and it shouldn't be! This is where <code>ClassMapAutoLoader</code> comes in handy. Here's how to use it:

First, we generate our classmaps using a script located in <code>./vendor/bin/classmap_generator.php</code>. The following generates a classmap in our module folder:

{% highlight ruby %}

cd ./module/my_module_name
../../vendor/bin/classmap_generator.php
{% endhighlight %}

We should now have a file named <code>autoload_classmap.php</code> in the root of our module. Next, we replace our <code>getAutoloaderConfig()</code> code with the following:

{% highlight ruby %}

public function getAutoloaderConfig()
{
    return array(
        'Zend\Loader\ClassMapAutoloader' => array(
            __DIR__ . '/autoload_classmap.php',
        ),
        'Zend\Loader\StandardAutoloader' => array(
            'namespaces' => array(
                __NAMESPACE__ => __DIR__ . '/src/' . __NAMESPACE__
            )
        )
    );
}
{% endhighlight %}

Now, all the autoloader needs to do is read the <code>autoload_classmap.php</code> file to retrieve the relationship between namespace and pathname. That's it! Looking back at our execution statistics, we found that only one of <code>Composer\Autoload\ClassLoader</code>'s four function calls was left in the list of top 10 slowest functions. Nice!
<h4><strong>Event Listeners</strong></h4>
The top 3 most <em>time</em> consuming calls in our application were all coming from <code>Zend\EventManager\EventManager</code>, which is responsible for triggering and listening to events. We were a bit wary of this report being a false positive, as Xdebug's <em>time</em> category represents total execution time of a function, and not time spent executing it's own code (excluding calls to other functions). However, it was also high up on the list of <em>own time</em>, so it was still worth looking into. Here's what we found:

In our application, we were making heavy use of the <code>SharedEventManager</code> to handle logging, mailing, and other application-wide events. We also used wildcards (*) to listen to events across several different modules. Even worse, each time a logging or mailer event was triggered, our logging and mailing services were retrieved through the <code>ServiceManager</code>, which is also fairly slow. This led to there being hundreds of calls to the <code>ServiceManager</code> in a single request. Not good!

Of course in many instances, events can be useful to decouple code, so we didn't want to completely remove eventing in our application. Our solution was to instantiate any services we knew we were going to need on bootstrap, such as the <code>Logger</code>. We also cleaned up our <code>SharedEventManager</code> code blocks to make sure we only triggered/listened to events when absolutely necessary. In general, our advice here is to be aware that the <code>SharedEventManager</code> can be dangerous, and to use it wisely.
<h4><strong>Skinny Module.php</strong></h4>
Obviously, the <code>Module.php</code> file is a pretty important file in your application. Looking back at our profiling statistics, we found that the call to <code>onBootstrap()</code> in our main <code>Module.php</code> file was pretty slow (around 10th in the list of <em>time</em> spent).

That being said: keep your <code>Module.php</code> files skinny. In most tutorials and documentation, there's a habit of putting invokables, initializers, factories, etc. in <code>Module.php</code>. Don't! All of these closures will be evaluated for every request. We can avoid this by creating factories, and declaring them in our <code>module.config.php</code> file. Here's an example of creating an initializer for our <code>Config</code> service:

In <code>module.config.php</code>:

{% highlight ruby %}

'service_manager' => array(
    'initializers' => array(
        'Application\Config\ConfigAwareInterfaceInitializer',
    ),
 ),
'controllers' =>
    'initializers' => array(
        'Application\Config\ConfigAwareInterfaceInitializer',
    ),
),
{% endhighlight %}

And in <code>Application\Config\ConfigAwareInterfaceInitializer</code>:

{% highlight ruby %}

class ConfigAwareInterfaceInitializer implements InitializerInterface
{
    /**
     * Initialize
     *
     * @param $instance
     * @param ServiceLocatorInterface $serviceLocator
     * @return mixed
     */
    public function initialize($instance, ServiceLocatorInterface $serviceLocator)
    {
        if (get_class($serviceLocator) === 'Zend\Mvc\Controller\ControllerManager') {
            $serviceLocator = $serviceLocator-&gt;getServiceLocator();
        }

        if ($instance instanceof ConfigAwareInterface) {
            $instance-&gt;setConfig($serviceLocator-&gt;get('Config'));
        }
    }
}
{% endhighlight %}

With these two pieces of code, we allow any of our services or controllers to implement the <code>ConfigAwareInterface</code>, which will be responsible for injecting the <code>Config</code> service when the service/controller is created. Note that we didn't put any code in <code>Module.php</code> to make this happen. We eliminate the need to evaluate any closures when bootstrapping the module, and we're also able to cache the <code>module.config.php</code> file (more about this in the next section).

After refactoring our primary <code>Module.php</code> file, we found that the call to <code>onBootstrap()</code> dropped from 11th to 16th in the list of longest time spent in a function. Not Bad!
<h4><strong>Cache Settings</strong></h4>
A pretty obvious tip here, but if you're not familiar with all the config files in a ZF2 application, this one might be overlooked. In your <code>./config/application.config.php</code> file, there are a few values you can modify to enable config caching:

{% highlight ruby %}

return array(

    //...

    'module_listener_options' => array(

        //...

        'config_cache_enabled' => ($env == 'production'),

        'config_cache_key' => 'my_cache_key',

        'module_map_cache_enabled' => ($env == 'production'),

        'module_map_cache_key' => 'my_cache_key',

        //...
    ),
{% endhighlight %}

Notice that for the keys <code>config_cache_enabled</code> and <code>module_map_cache_enabled</code>, we set it to true if our environment is in production. When these are true, two cache files will be created: <code>module-classmap-cache.my_cache_key.php</code> and <code>module-config.cache.my_cache_key.php</code>. The <code>config_cache</code> will be a concatenation of all our modules config files, while the <code>module_map_cache</code> will be a map of module namespaces to their respective <code>Module.php</code> files. As with our previous examples, these are some quick fixes that should really be there by default.
<h4><strong>Doctrine Caching</strong></h4>
If you're using Doctrine in your application, there are some easy caching options available that will undoubtably improve your app's performance. We recommend taking a quick look at <a href="http://doctrine-orm.readthedocs.org/en/latest/reference/improving-performance.html">Doctrine's documentation</a> to get an idea of how important it is to use a cache with Doctrine.

With those warnings in mind, here's how to enable Doctrine caching in your ZF2 application:

We'll need edit three different files. First, we edit our local configuration file to tell doctrine what type of cache we're using. Next, we edit our global configuration to specify where the factory is for creating our caching service. Finally, we define a factory that is responsible for creating the caching service:

In <code>./config/autoload/local.php</code>:

{% highlight ruby %}

'doctrine' => array(
    'configuration' => array(
        'orm_default' => array(
            'metadata_cache'    => 'myApc',
            'query_cache'       => 'myApc',
            'result_cache'      => 'myApc',
            'generate_proxies'  => false,
        )
    )
),
{% endhighlight %}

In <code>./config/autoload/global.php</code>:

{% highlight ruby %}

'service_manager' => array(
    'factories' => array(
        'doctrine.cache.myApc' => 'Application\Cache\ApcFactory',
    ),
    'abstract_factories' => array(
        'Zend\Cache\Service\StorageCacheAbstractServiceFactory',
    )
),
{% endhighlight %}

In <code>ApcFactory.php</code>:

{% highlight ruby %}

namespace Application\Cache;

use Doctrine\Common\Cache\ApcCache;
use Zend\ServiceManager\FactoryInterface;
use Zend\ServiceManager\ServiceLocatorInterface;

class ApcFactory implements FactoryInterface
{
    /**
     * Create service
     *
     * @param ServiceLocatorInterface $serviceLocator
     * @return mixed
     */
    public function createService(ServiceLocatorInterface $serviceLocator)
    {
        $apcCache = new ApcCache();

        return $apcCache;
    }
}
{% endhighlight %}

In this example we're using ApcCache, but there are other options as well, such as Memcache. One more thing to point out is that in our <code>local.php</code>, we specified <code>'generate_proxies' => false</code>. By setting this to <code>false</code>, we're saying that we DON'T want doctrine to generate new proxy entity classes at runtime. See <a href="http://doctrine-orm.readthedocs.org/en/latest/reference/advanced-configuration.html">here</a> for more information on proxies. As with our classmap autoloading, we can generate these proxies once, before the application is loaded. Doctrine provides us with a simple script for generating these proxies from the command line:

{% highlight ruby %}

./vendor/bin/doctrine-module orm:generate-proxies
{% endhighlight %}

And we're done! With these steps, we now have doctrine caching entity metadata, php queries, and query results. We've also stopped Doctrine from generating entity proxies. I recommended reading Doctrine's documentation on caching and proxies for more information on how these will benefit you.
<h4><strong>Session Write Close</strong></h4>
While profiling our application, we noticed something strange. The call to <code>session_start()</code>, located in <code>Zend\Session\SessionManager</code>, was taking several seconds for some of our API calls. Surely, all this function needs to do is read the session file? Well, it turns out that PHP locks the session file until a request has been completed. That means that in cases where we make several API calls at once, each of them needing access to the session, they will each be blocking until their request has completed.

Luckily, we can fix this by calling <code>session_write_close()</code>. This will disable writing to the file, while still allowing us to read from it. Note that if you need to write to the session again, you must reopen it. For us, it was enough to make a call to <code>session_write_close()</code> once the API call has been authorized, thus allowing for any subsequent calls to open the session.
<h4><strong>Opinions + Conclusion</strong></h4>
Looking back at the changes we made, we've come to a couple of conclusions. Feel free to share your thoughts:

Our initial approach to architecting our application was to split everything up into different modules. We later realized that many of our modules should have been part of a main application module. Each module now had their own pile of boilerplate config and a scary file tree, they had their own classmap autoloading, bootstrapping functions, eventing structure, etc. Not only was this affecting our application load times, it was also slowing down development time and increasing complexity. Our suggestion: pull out modular code later, rather than sooner.

ZF2's has obviously taken an unopinionated approach to development. While this provides us with a lot of flexibility, it also allows us to make some pretty poor choices. A good example of this is allowing us to create invokables/initializers/factories from the <code>Module.php</code> file when it is almost always a bad idea to do so. This is compounded by the fact that ZF2 has rather poor documentation, something that is crucial for a framework that hopes to provide a lot of flexibility. Although there are fixes to most problems we've encountered, it all suggests that some parts of the framework could have benefit by being more restrictive/opinionated.

Although many more changes were made (adding queued jobs, encryption speed ups, etc.), the tips above can be used in almost any ZF2 application to improve performance. Keep in mind, though, that these are not a silver bullet to creating a high performance ZF2 application. As mentioned above, it is highly recommended to profile your code before you start and see where that takes you. Best of luck!
