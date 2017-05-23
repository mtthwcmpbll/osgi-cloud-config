# OSGi Cloud Config

This repository provides an OSGi bundle and associated feature that can be loaded
into an OSGi container to parse JSON data held within an environment variable.  This
pattern is used by platforms such as CloudFoundry to expose application and bound
service information to the application instance.

The primary actor in this bundle is the `EnvJsonProperties` class, which can be configured to
parse a particular environment variable and bind it's contents to OSGi configuration
with a particular prefix.  Doing this in a bundle that is loaded early in the container
lifecycle allows downstream bundles to pull the parsed configuration using normal
Configuration services as well as blueprint `<properties-placeholser>`.

The provided `osgi-vcap-config` bundle automatically parses and exposes CloudFoundry's `VCAP_APPLICATON`
with the `vcap.application` prefix and `VCAP_SERVICES` with the `vcap.service` prefix.
See that bundle's blueprint for an example of how you could use this to expose other
environment variables as needed.

## Installation

In order to use this feature, you'll need to install the bundle and/or feature into
your OSGi container. For example, to use this in Apache Karaf, you can install the bundle
and feature into your local Maven cache with `mvn clean install`, install the feature repository
using the karaf console's `feature:repo-add mvn:com.snowfort/osgi-cloud-config-features/1.0.0-SNAPSHOT/xml/features`,
and then install the provided VCAP-parsing feature with `feature:install osgi-vcap-config`.

## Usage

With the `osgi-vcap-config` feature installed, the bundle will start early enough in
the OSGi startup lifecycle that your own bundles should be able to parse vcap data. An
example is provided in `osgi-vcap-config/src/test/resources/vcap-services-properties-test.xml` - this
is a blueprint-only bundle that extracts values from the following example `VCAP_SERVICES`:

```
{
  "cleardb": [
    {
      "credentials": {
        "hostname": "us-cdbr-iron-east-03.cleardb.net",
        "jdbcUrl": "jdbc:mysql://us-cdbr-iron-east-03.cleardb.net/ad_b9eb3f3f82ef228?user=b69bed0a151d22&password=3a5ac732",
        "name": "ad_b9eb3f3f82ef228",
        "password": "3a5ac732",
        "port": "3306",
        "uri": "mysql://b69bed0a151d22:3a5ac732@us-cdbr-iron-east-03.cleardb.net:3306/ad_b9eb3f3f82ef228?reconnect=true",
        "username": "b69bed0a151d22"
      },
      "label": "cleardb",
      "name": "test-mysql",
      "plan": "spark",
      "provider": null,
      "syslog_drain_url": null,
      "tags": [
        "Cloud Databases",
        "Data Stores",
        "Web-based",
        "Online Backup & Storage",
        "Single Sign-On",
        "Cloud Security and Monitoring",
        "Certified Applications",
        "Developer Tools",
        "Data Store",
        "Development and Test Tools",
        "Buyable",
        "relational",
        "mysql"
      ],
      "volume_mounts": []
    }
  ]
}
```

This is done by setting up a `property-placeholder` with a persistent-id that matches
the value provided to `EnvJsonProperties` and property names in a JSON path-like
format.  Note that array items are indexed with a suffix `_i`, where _i_ is the item's
index in the array.

```
<cm:property-placeholder persistent-id="vcap.services">
    <cm:default-properties>
        <cm:property name="cleardb_0.credentials.jdbcUrl" value="defaultJdbc"/>
        <cm:property name="cleardb_0.credentials.name" value="defaultUsername"/>
        <cm:property name="cleardb_0.credentials.password" value="defaultPassword"/>
        <cm:property name="cleardb_0.credentials.missingProp" value="defaultValue"/>
    </cm:default-properties>
</cm:property-placeholder>
```

You can then use these values just like any other configuration value.  If a value
exists with that path in an environment variable handled by the loading bundle,
you'll see your value logged.  In the case of a missing value (`missingProp` in this case),
it'll fall back on the fault provided in `property-placeholder`.

```
<camelContext xmlns="http://camel.apache.org/schema/blueprint">
    <route>
        <from uri="timer:test" />
        <log message="vcap.services.cleardb_0.credentials.jdbcUrl:    {{cleardb_0.credentials.jdbcUrl}}" loggingLevel="INFO"/>
        <log message="vcap.services.cleardb_0.credentials.name:       {{cleardb_0.credentials.name}}" loggingLevel="INFO"/>
        <log message="vcap.services.cleardb_0.credentials.password:       {{cleardb_0.credentials.password}}" loggingLevel="INFO"/>
        <log message="vcap.services.cleardb_0.credentials.missingProp:       {{cleardb_0.credentials.missingProp}}" loggingLevel="INFO"/>
    </route>
</camelContext>
```
