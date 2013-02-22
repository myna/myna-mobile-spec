# Myna Mobile API

This document describes forthcoming mobile client libraries for Myna. The same basic template will be used for client libraries for iOS, Android, and HTML 5 mobile apps.

This is a work-in-progress and we welcome feedback. Please send feedback opening an issue on Github or emailing us at hello@mynaweb.com.

The main purpose of these libraries is to abstract away the problems of communication with the Myna servers. The client libraries will download experiment data and cache it on the device. Tests will be run locally without the need for constant communication with the Myna servers. Test results will be synced back to the servers as and when a network connection is available.

The examples below are written using Javascript pseudocode. The native client libraries for Android and iOS will follow a similar pattern in Java and Objective C respectively.

**Note for existing Myna users:** These clients will use a new version of the Myna API, documentation for which can be found on [Apiary](http://docs.mynaweb.apiary.io).

## Authentication

The main access point for Myna is a `MynaClient` object, which is constructed with a set of API credentials:

    // By API key:
    var client = Myna.getClient({ apiKey: "aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa" });

    // By account email and password:
    var client = Myna.getClient({ email: "dave@mynaweb.com", password: "password" });

## Accessing experiments

There are two ways of accessing experiments from the Myna servers: clients can download all experiments in the user's account or access a single experiment by ID. Each operation can be performed using a blocking or non-blocking call:

    // All experiments:

    // Blocking:
    var expts = client.getExperiments();

    // Non-blocking:
    client.getExperimentsAsync(function(expts) {
        var expt = expts["dashboard"];
        // ...
    });

    // Single experiment by ID:

    // Blocking
    var expt = client.getExperiment("dashboard");

    // Non-blocking
    client.getExperimentAsync("dashboard", function(expt) {
        // ...
    });

Experiment data is downloaded and cached on the device, allowing multiple views and conversions to be recorded without worrying further about asynchronous code.

**Note for existing Myna users:** In the current version of Myna, each experiment is automatically assigned a UUID-style ID. In the new version of Myna, users will be able to assign their own IDs on the Myna dashboard. This makes it easier to access and debug experiments in client libraries.

## Accessing variants

Once obtained, experiments can be queried to yield a variant to display to the user:

    // Allow Myna to suggest the variant:
    var variant = expt.viewVariant();

    // Alternatively, forcably select a particular variant:
    var variant = experiment.viewVariant("variant1");

In both of the above cases, a "view" is recorded against the variant in question. This information is queued for asynchronous transmission back to the Myna servers.

Each variant has an *ID* and a set of JSON-like *metadata* that the developer can query to set up the view of the variant. For example:

    // Determining a button colour programatically using the variant ID:

    var buttonColor = "gold";
    id(variant.id == "variant1") {
        buttonColor = "red";
    } else if(variant.id == "variant2") {
        buttonColor = "green";
    }

    // Querying variant metadata to fetch the button colour:

    var buttonColor = variant.getMetadata("button.color");

    // The experiment object also contains metadata,
    // which is useful for storing defaults:

    var buttonColor = variant.getMetadata("button.color") ||
                      expt.getMetadata("button.defaultColor");

Developers are strongly encouraged to use variant metadata wherever possible. Metadata can be edited on the Myna dashboard and synchronised automatically to the device, allowing developers to add, edit, and remove variants without deploying a new version of the app.

## Recording conversions

Conversions can be recorded via the variant object:

    variant.recordConversion();

An optional second argument can be added to specify a "reward amount" between 0 and 1:

    variant.recordConversion(0.8);

Results are queued for asynchronous transmission to the Myna servers in the background.

## Synchronizing results back to the Myna servers

Whenever an app views a variant or records a conversion, the Myna client records an *event* and attempts to transmit it back to the Myna servers. Events are queued until a network connection is available and *synchronized* to clear the queue.

Synchronization is attempted at the following moments:

 - whenever `viewVariant` is called;

 - whenever `recordConversion` is called;

 - whenever a Myna client is created and queued events are available
   to transmit.

 - whenever a network connection becomes available after a period of
   disconnection, and queued events are available to transmit.

## Cached views and conversions

By default, when `viewVariant` is called for the first time, the variant suggested by the server is cached in local storage on the device. The cached variant is reloaded by future calls to `viewVariant` without sending further events to the server. Similarly, the first call to `rewardConversion` adds data to the cache to prevent duplicate rewards.

Cached views and conversions can be cleared at an experiment or client level as follows:

    // Clear cached data for a single experiment:
    expt.clearCache();

    // Clear cached data for all experiments:
    client.clearCache();

Once the cache is cleared:

 - the next call to `viewVariant` will yield a new variant and transmit
   a new view event back to the server;

 - the next call to `recordConversion` will transmit a new conversion
   event back to the server.

## App updates

The cache is sensitive to changes in the application's version number. Changes to the version number will cause the cache to be cleared and new data to be retrieved from the server.
