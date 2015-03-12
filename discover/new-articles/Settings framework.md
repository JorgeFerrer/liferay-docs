# Overview of Settings framework

## Introduction

The Settings framework solves the needis intended to work with the configuration of Liferay portlets and services. This framework allows specifying the configuration at several scopes:
# System scope: The values are set one for the whole Liferay installation
# Portal Instance scope: Allows each portal instance to have different configuration values. This is meant for services that deal with data not associated to a site, such as UserService, GroupService, etc.
# Site scope: Allows each site to have different configuration values. This is meant for services that deal with data associated with a site, such as JournalService, MBMessageService, ...
# Portlet Instance scope: Allows each portlet instance to have its own configuration values (this is obviously only applicable to portlets and not to services).

In addition to the above, the Settings framework automatically deal with default values for each scope. For example, if you develop a wiki service which has a configuration property to select the markup used, you may want to allow selecting it per site. Once you do that, the settings framework allows setting the default for a portal instance, so that all new sites in the instance will use that value unless it is changed. It also automatically (with no effort from your side) allows setting the default value at the system scope, which will be used for all new portal instances unless it is changed.

The Settings framework is implemented leveraging the OSGi configuration capabilities, which allows for a high performance reading of properties while still allowing dynamic changes to the configuration even at the system scope.

### How does this compare with the solutions prior to Liferay 7?

If you have developed configurable portlets or services for Liferay in previous versions you might be wondering how things are changing. In versions prior to 7.0 the recommended approach was to use the PortletPreferences API from the portlet specification to retrieve and store the portlet configuration. The standard behaviour was altered with a set of elements in liferay-portlet.xml (preferences-owned-by-group, preferences-unique-per-layout, ...). There was no clear differentiation between portlet configuration and service configuration.

Under the hood, the dynamic storage was handled by the Liferay PortletPreferences API and PortalPreferences APIs. These APIs are used for a large variety of purposes and thus it is hard to understand and use.

In addition to the above it was also common to store portlet/service configuration in properties. That meant in portal.properties in the core or portlet.properties in plugins. The values in each of these files could be overridden by placing a file called portal-ext.properties or portlet-ext.properties in the core or plugin classpahts. The properties in these files some times acted as the final configuration which could not be changed in the UI and sometimes as the default values for a configuration UI.

In summary the configuration and its default values were stored in:

* portal.properties
* portal-ext.properties
* the database table PortalPreferences 
* the database table PortletPreferences

Each configuration value was designated by a key (a string) which could be different when stored in the DB or in properties files. For example: *emailFromUser* was stored as such in the database, but as *email.from.user* in properties files. 

Default values had to be provided in the client code, when calling the API, which made it difficult to track them.

To finish with, it was not clear which configured values were shared among different portlets or unique per portlet instance.

The Settings framework was designed to address all this limitations and make it possible for Liferay developers to make their portlets and services highly configurable with very little effort.

## Concepts

### Basic concepts (Settings and TypedSettings)
In the following sections we are introducing the root concepts involved in the Settings API. 

To begin with, we have the concept of **settings** objects, which are entities containing several configuration values, identified by a **key**, and with an actual **value** of a specific type (`String`, `boolean`, `int`, etc.). They work like a map and are defined in Java by the `Settings` interface.

Raw `Settings` only support `String` and `String[]` values. In case you need support for primitive values, you must create a `TypedSettings` object giving it the raw `Settings` object. `TypedSettings` adds support for values of type:

* `boolean`
* `double`
* `float`
* `int`
* `long`
* `LocalizedValuesMap`

The `LocalizedValuesMap` type is a special structured type that allows having several string values for one single configuration key indexed by locale. Also, a **default value** is defined for locales which don't have a specific value set.

### Retrieving settings (settings id and levels)
Settings objects are identified by a **settings id**, which is a `String` containing an OSGi symbolic name (a Java package like name), a portlet id, or a portlet instance id.

Settings objects live at different **settings level**. The settings level is the scope where settings are stored. It affects the instantiability and cardinality of settings instances and how the value of keys are resolved.

There are several settings scopes:

* System: settings at this level are unique per server installation.
* Company service: settings at this level have one different value per company.
* Group service: settings at this level have one different value per group.
* Portlet instance: settings at this level have one different value per portlet instance.

Depending on the settings level where a configuration is stored its values are resolved with a different algorithm, that is explained in the following section.

### Reading and writing settings (lookup chain)
Configuration values stored in `Settings` objects are read-only by default (though it is evident that someone has to write them if we want them to be useful). To be able to modify a settings object you need to ask for its associated `ModifiableSettings` object, made available through a specific getter.

The need to distinguish between read-only and modifiable settings is due to the chaining of configurations. The concept of chaining implies looking in several places (**lookup chain**) for configuration values until they are found. 

For instance, if you have a configuration for a portlet instance and ask for a value, the framework will look for it at the portlet instance level, but if it is not found there, it will look for it again at the group level, then company level, then portal instance level, and so on, until portal.properties or the OSGi Configuration Admin service is hit.

Thus, when you ask a `Settings` object at portlet instance level for a key it can give you a value which, in reality, is set at a lower level. But then, if you ask for that value to its `ModifiableSettings`, the lookup chain won't be applied and the value will be returned if and only if it is set at that level. 

Also, if you set the value of that key, it is set at the portlet instance level so that the next time you ask for it, it is retrieved from that level.

We define different chains per settings level. Currently the following lookup chains are used for each settings level:

* System (for an specific *settingsId*):
	1. OSGi configuration with *configurationPid* == *settingsId*
	2. The usual portal.properties chain

* Company service (for an specific *companyId* and *settingsId*):
	2. The `PortletPreferences` with company owner type, given *companyId*, and *portletId* == *settingsId*
	3. The `PortalPreferences` with company owner type, and given *companyId*
	4. OSGi configuration with *configurationPid* == *settingsId*
	5. The usual portal.properties chain

* Group service (for an specific *groupId* and *settingsId*):
	1. The `PortletPreferences` with group owner type, given *groupId*, and *portletId* == *settingsId*
	2. The `PortletPreferences` with company owner type, the *companyId* associated to the given *groupId*, and *portletId* == *settingsId*
	3. The `PortalPreferences` with company owner type, and *companyId* == associated *companyId* of the given *groupId*
	4. OSGi configuration with *configurationPid* == *settingsId*
	5. The usual portal.properties chain

* Portlet instance (for an specific *portletId* and *layout*)
	1. The `PortletPreferences` with layout owner type, the *companyId* and *plid* associated to the given *layout*, and the given *portletId*
	2. The `PortletPreferences` with group owner type, the *companyId* and *groupId* associated to the given *layout*, and the given *portletId*
	3. The `PortletPreferences` with company owner type, the *companyId* associated to the given *layout*, and the given *portletId*
	4. The `PortalPreferences` with company owner type, and *companyId* == associated *companyId* of the given *layout*
	5. OSGi configuration with *configurationPid* == *portletId*
	6. The usual portal.properties chain

### Getting snapshots of portlet instance settings (ArchivedSettings)
Some Liferay configuration dialogs offer the possibility to save the current configured settings with a name. These saved configurations may be retrieved, deleted and written from the Settings API by means of an `ArchivedSettings` object which is a decoration of the standard `ModifiableSettings` object.

### Typesafe settings (high level settings)
When manipulating configuration through the `Settings` interface, all values must be referenced by their key. This is difficult to maintain and error prone, what we would like to do is to manipulate the configuration by using a real Java object that implements some interface describing the structure of the configuration.

Fortunately, OSGi has an specification called *Metatype* which allows to define this type of interfaces. Settings API is fully compatible with this specification and may emulate a typesafe configuration given its *settingsId* and the interface to coerce the configuration into.

These interfaces are called **high level settings** and are syntactic sugar to convert code like:

```
	Setting settings = ...;
	
	String emailAddress = settings.getValue("emailAddress");
	
	...
	
	return emailAddress;
```

into:


```
	MyHighLevelSettings settings = ...;
	
	String emailAddress = settings.emailAddress();
	
	...
	
	return emailAddress;
```

The *Metatype* specification allows defining default values too, which is useful to gather the configuration structure and its default values into just one Java class.

Please have a look at the Metatype specification to learn how to define high level settings interfaces.

### Declaring high level settings (SettingsDefinition)
Declaring the existence of a high level settings class is as easy as deploying an *immediate* OSGi component that implements the `SettingsDefinition` interface.

Please refer to the *Defining a module configuration with Settings API* section to learn how to deploy a new configuration.

### Retrieving high level settings (settings providers)
For each declared high level settings, an accompanying **settings provider** is registered as an OSGi component which can be used to retrieve the high level settings. 

There are several settings provider interfaces, one per settings level, named:

* **TODO[ ServerSettingsProvider ]**
* **TODO[ CompanyServiceSettingsProvider ]**
* **GroupServiceSettingsProvider**: for configurations at group service level.
* **PortletInstanceSettingsProvider**: for configurations at portlet instance level.

All of them are generified so that they return the correct high level settings type. 

Also, when retrieved as OSGi services, they have a `class.name` property specifying the high level settings type they return (which must correspond with the generics one).

See the section on *Consuming a module configuration* to learn more about how to use settings providers.

### Configuration bean
Each high level settings object has an associated OSGi *configuration bean* which is managed using the `Configuration Admin` service. 

This configuration bean stores the default values for all high level settings objects and is a singleton. This means that, even if the high level settings is stored at the portlet instance level, there only exist one configuration bean associated to it.

Configuration beans are the equivalent to the old *portal.properties* but they are better because they are modularized and can be managed using standard OSGi services like, for example, the *Felix Web Console*.

### Special configuration values (location variables)
Usually, configuration keys are assigned static values but, under some circumstances (usually default values) it may be useful to set a configuration value to a variable value. 

For this purpose, the settings API supports **location variables**, which are values with the format:

```
	${URL}
```	

where *URL* must have one of the following formats:

* `resource:path/to/static/resource/file/in/class/namespace/file.properties`

	Resolves to a file retrieved with the classloader where the high level settings interface is defined.


* `	file:///absolute/path/to/a/file/in/the/filesystem/file.properties`

	Resolves to a file stored in the local file system.


* `server-property://settings.id/referenced.property.key`

	Resolves to another configuration property defined at server level and designated by the given *settingsId* and *key*. 

# Usage of Settings API

## Defining a module configuration

### Step 1: Define the structure of the configuration
The first thing we need to do is to create a Java interface describing the fields that apply to the configuration. 

Let's see an example of a group service level configuration:

```
@Meta.OCD(id = "com.liferay.example.configuration.ExampleGroupServiceConfiguration")
public interface ExampleGroupServiceConfiguration {

	@Meta.AD(
		deflt = "jpg", required = false
	)
	public String defaultFormat();

	@Meta.AD(
		deflt = "${server-property://com.liferay.portal/admin.email.from.address}",
		required = false
	)
	public String emailFromAddress();
	
}
```

In this example we are declaring that the configuration will have two `String` fields named *defaultFormat* and *emailFromAddress* with the given default values. 

Also, the `Meta.OCD` annotation is declaring under which *configurationPid* we are going to store the configuration in *Configuration Admin* service.

### Step 2: Declare the high level settings interface
In addition to the previous interface, we need to define another one which, basically, has the same methods as the configuration but defines the high level settings object associated to this OSGi configuration. 

Thus, we do:

```
public interface ExampleGroupServiceSettings
	extends GroupServiceSettings, ExampleGroupServiceConfiguration
{
}
```

Although it may seem redundant to declare this interface it is important to do it for two reasons:

1. By extending `GroupServiceSettings` we are stating that this configuration lives at group service level.
2. By having this interface we can distinguish between the objects retrieved directly from OSGi using the *Configuration Admin* service, and the ones retrieved from the Settings API which, obviously, are different entities although they have the same methods.

### Optional step 3: Add extra methods to the high level settings
This step is not necessary but it gives you an extra level of customization if you need it. It allows you to define methods which will be implemented by the high level settings object (`ExampleGroupServiceSettings` in our example) but not by the OSGi configuration (`ExampleGroupServiceConfiguration` in our example).

Let's say you want to add a new method to get the `emailFromAddress`'s domain part and make it available to all `ExampleGroupServiceSettings` objects. To do that you need to:

* Declare a new interface with the extra methods:

```
public interface ExampleGroupServiceSettingsExtra {

	public String emailFromAddressDomain();

}
```

* Implement the interface:

```
public class ExampleGroupServiceSettingsExtraImpl
	implements ExampleGroupServiceSettingsExtra {

	public ExampleGroupServiceSettingsExtraImpl(TypedSettings typedSettings) {
		_typedSettings = typedSettings;
	}

	@Override
	public String emailFromAddressDomain() {
		String emailFromAddress = _typedSettings.getValue("emailFromAddress");

		int i = emailFromAddress.indexOf("@");

		return emailFromAddress.substring(i+1);
	}
	
}
``` 

Note that the implementation receives a `TypedSettings` object in the constructor. This is the low level settings object which contains the configuration. It allows you to retrieve configuration values referenced by keys whether or not those keys are declared in the `ExampleGroupServiceConfiguration` interface.

### Step 4: Implement the SettingsDefinition
In this final step you just need to create an OSGi component, set its `immediate` property to true and implement the `SettingsDefinition` interface.

In our example:

```
@Component(immediate = true)
public class ExampleGroupServiceSettingsDefinition 
	implements SettingsDefinition<ExampleGroupServiceSettings, ExampleGroupServiceConfiguration> {

	@Override
	public Class<ExampleGroupServiceConfiguration> getConfigurationBeanClass() {
		return ExampleGroupServiceConfiguration.class;
	}

	@Override
	public Class<ExampleGroupServiceSettings> getSettingsClass() {
		return ExampleGroupServiceSettings.class;
	}

	@Override
	public Class<?> getSettingsExtraClass() {
		return ExampleGroupServiceSettingsExtraImpl.class;
	}

	@Override
	public String[] getSettingsIds() {
		return new String[] { "com.liferay.example" };
	}

}
```

In case you haven't defined any extra methods you can return `null` in the `getSettingsExtraClass()` method.

## Consuming a module configuration
To consume a module configuration you just need to grab a reference to its settings provider via OSGi, ask for the given configuration object and start using it.

In our example you would have code like this:

```
@Component
public class ExampleGroupServiceSettingsConsumer {

	public void consumeSettingsForGroupId(long groupId) throws PortalException {
		ExampleGroupServiceSettings groupServiceSettings =
			_groupServiceSettingsProvider.getGroupServiceSettings(groupId);

		System.out.println(
			"Consuming an extra method from settings: " +
				groupServiceSettings.emailFromAddressDomain());
	}

	@Reference(
		target = "(class.name=com.liferay.example.ExampleGroupServiceSettings)"
	)
	protected void setGroupServiceSettingsProvider(
		GroupServiceSettingsProvider<ExampleGroupServiceSettings>
			groupServiceSettingsProvider ) {

		_groupServiceSettingsProvider = groupServiceSettingsProvider;
	}

	private GroupServiceSettingsProvider<ExampleGroupServiceSettings>
		_groupServiceSettingsProvider;

}
```

# Deployment of Settings API
The Settings API is implemented at two levels: 

* The low level API inside portal's core: allows you to use low level `Settings` objects
* The high level API inside its own `portal-settings` module: this module listens for `SettingsDefinition` registrations and registers its settings provider and all necessary services to make it work.

In particular, whenever a new `SettingsDefinition` is found, three services are registered with OSGi:

1. A `ManagedService` as defined by OSGi's *Configuration Admin* which listens to changes in the configuration so that it stays always fresh inside the runtime.
2. A service under the same interface of the configuration bean which can be used to access OSGi's configuration without any need to talk to *Configuration Admin*. Also, this component is automatically refreshed whenever the configuration changes so that consumers don't need to bother with changes: they always see the current configuration refreshed in real time.
3. A settings provider under the correct interface and its `class.name` property set to the high level settings object it is providing.

For our example, we would have three services named:

1. `org.osgi.service.cm.ManagedService` with properties `{service.pid=com.liferay.example}`.
2. `com.liferay.example.ExampleGroupServiceConfiguration` without properties.
3. `com.liferay.portal.kernel.settings.GroupServiceSettingsProvider` with properties `{class.name=com.liferay.example.ExampleGroupServiceSettings}`.

