App Protocol Langpacks Spec
===========================

This document outlines the design of the language package support implemented 
via the `app:` protocol handler.  The design revolves around special webapps 
which can register themselves to provide localization resources for multiple 
apps and multiple languages at once.

There are two independent parts of the language package system:

 - language negotiation, which requires the localization library to be able to 
   query the platform for languages available for the current app, both 
   bundled with the app and installed by the user as langpacks,

 - localization resources IO, which fetches the resources either from the 
   current app or a user-installed langpack.

This proposal relies on the app installation process to set Gecko prefs which 
store the list of languages available for the given app, and the app protocol 
handler to redirect l10n resource IO to langpacks, if necessary.

In this first version of the spec, we propose that langpacks be certified apps 
installed by partners to configure and easily extend language coverage of 
Firefox OS builds without having to rebuild the entire system.  This also 
allows the first implementation to ignore version checking.  In the future 
versions of the spec we plan to lift the certification restriction and allow 
langpacks to be privileged apps installed from the marketplace by the user.  
For this reason, the current spec includes versioning data.

(In the following examples we use the Settings app to illustrate the design.)


App's HTML
----------

In HTML, language resources are linked like this:

    <link 
      rel="localization"
      href="app://{locale}.l10n.settings.gaiamobile.org/locales/settings.{locale}.properties">

    Q. Do we need the `l10n` part following the `{locale}` variable?


App's Manifest
--------------

In the manifest, the app registers itself as a provider of its own bundled 
languages:

    "name": "Gaia Settings",
    "version": "2.2",
    "languages": {
      "default": "en-US",
      "available": {
        "en-US": "2.2-1",
        "de": "2.2-1"
      }
    },
    "overrides": {
      "en-US.l10n.settings.gaiamobile.org": "/",
      "de.l10n.settings.gaiamobile.org": "/"
    }

The `languages` field is used for language negotiation.  The `overrides` field 
is used for resource fetching.

The `overrides` field is optional (in which case the link elements in the HTML 
need to point to the same domain as the current app's).  It's a mapping of all 
possible per-language domains to the directory in which to look for the 
corresponding localization files.  It can be generated on build time from the 
`languages.available` field and the URL templates collected from the app's HTML 
files.

    Q. What is a good name for the `overrides` field?  Proposed: `overrides`, 
    `resources`, `domains`, `hosts`, `origins`, `alias`.

    NB. This design makes it possible to also specify the resource override 
    like so:

        "en-US.l10n.settings.gaiamobile.org": "/locales/"

    …and to link to localization resources using the following URL:

        app://{locale}.l10n.settings.gaiamobile.org/settings.{locale}.properties

    NB. See also the section at the end of this document about generalizing 
    this design to support other types of resources.


App Installation
----------------

During app installation (or on buildtime) the following Gecko prefs are set 
as per the manifest:

    'webapps.languages.settings.gaiamobile.org':
      '{"default":"en-US","available":{"en-US": "2.2-1","de": "2.2-1"}}'
    'webapps.overrides.en-US.l10n.settings.gaiamobile.org':
      'settings.gaiamobile.org'
    'webapps.overrides.de.l10n.settings.gaiamobile.org':
      'settings.gaiamobile.org'

    Q. Are Gecko prefs the right place to store this data?


App.getLanguages
----------------

When a document is opened the l10n.js library is tasked with negotiating the 
correct language to display the UI in to the user.  It uses navigator.languages 
to get the list of user's preferred langs and the App.getLanguages API (in case 
ofwebapps) or `meta` elements (in case of regular webpages) to get the list of 
languages available for the current app, as well as the default language.  The 
result of the negotiation is a list of supported locales.

For webapps, the information about the languages available for the current app 
is stored in a Gecko pref and can be accessed via the `mozApps` API.

    App.getLanguages();

This method returns a `DOMRequest` which resolves to the JSON-parsed value of 
the `'webapps.languages.' + appId` Gecko pref.

For regular webpages, the same kind of information is stored in meta elements 
and can be accessed synchronously.  There is no special localization manifest 
for regular webpages.

An example code in l10n.js using this API could look like this:

    function getAppLanguages() {
      return new Promise(function(resolve, reject) [
        var appReq = navigator.mozApps.getSelf();
        appReq.onerror = reject;
        appReq.onsuccess = function() {
          var app = appReq.result;
          if (app === null) {
            return resolve(null);
          }

          var langsReq = app.getLanguages();
          langsReq.onerror = reject;
          langsReq.onsuccess = function() {
            resolve(langsReq.result);
          };
        };
      });
    }

    getAppLanguages().then(function(langs) {
      if (langs) {
        // Negotiate supported languages using navigator.languages, 
        // langs.default and langs.available.
      } else {
        // We're not in an app or there are no registered languages.
        // Try to find <meta name="availableLanguages"> and 
        // <meta name="defaultLanguage"> and continue with language 
        // negotiation.
      }
    });

    Q. How do we handle changes in available languages that happen during app's 
    runtime?  Can an open app dynamically react to them?  Should there be an 
    `availablelanguageschange` event and which object should emit it 
    (`navigator.mozApps`?)?  Can l10n.js listen to events about other apps 
    being installed/uninstalled and react if the app in question is a langpack?

    NB. An alternative approach would be to create a new native API, e.g.  
    `navigator.mozLanguageSupport.get(appOrigin)`. The idea is that additional 
    locales for an appOrigin are stored by the navigator, not the app itself.  
    This API could be then used to emit the `availablelanguageschange` events 
    as well as provide methods for resource fetching (instead of the app: 
    protocol approach described below).



Resource IO
-----------

Once the l10n.js library has negotiated the list of supported languages, it can 
resolve the URL templates from `link` elements and fetch the resources in the 
correct language.  For instance, if the first language on the negotiated list 
is German, the following URL template:

    app://{locale}.l10n.settings.gaiamobile.org/locales/settings.{locale}.properties

…becomes:

    app://de.l10n.settings.gaiamobile.org/locales/settings.de.properties

This URI is used to perform an XHR request by the l10n.js library.  The app 
protocol handler gets the value of the Gecko pref corresponding to the host of 
the requested URI:

    'webapps.overrides.de.l10n.settings.gaiamobile.org':
      'settings.gaiamobile.org'

…and uses this value as the host name to create a new, valid URI:

    app://settings.gaiamobile.org/locales/settings.de.properties

…which points to an existing resource.



Langpack App Manifest
---------------------

Langpacks are webapp with a special 'langpack' role.  They're hidden from the 
Homescreen and can't be launched.  They act as bundles of resources which 
override requests to certain origins.  A single langpack can contain resources 
for multiple apps and multiple languages.

    "role": "langpack",
    "languages-provided": {
      "system.gaiamobile.org": {
        "de": "2.2-4"
      },
      "settings.gaiamobile.org": {
        "de": "2.2-4",
        "pl": "2.2-7"
      }
    }
    "overrides": {
      "de.l10n.system.gaiamobile.org": "/system",
      "de.l10n.settings.gaiamobile.org": "/settings"
      "pl.l10n.settings.gaiamobile.org": "/settings"
    }

    Q. Can we enforce that overrides are defined only for subdomains of apps 
    listed in `languages-provided`?

A special `languages-provided` field defines languages and their versions for 
app origins.  This information is used for language negotiation.


Langpack Installation
---------------------

When a langpack is installed, corresponding Gecko prefs are modified __if and 
only if__ the language pack's version is newer then the already installed.

    'webapps.languages.settings.gaiamobile.org':
      '{"default":"en-US",
        "available":{"en-US":"2.2-1","de":"2.2-4","pl":"2.2-7"}}'
    'webapps.overrides.en-US.l10n.settings.gaiamobile.org':
      'settings.gaiamobile.org'
    'webapps.overrides.de.l10n.settings.gaiamobile.org':
      'my-langpack.gaiamobile.org/settings'
    'webapps.overrides.pl.l10n.settings.gaiamobile.org':
      'my-langpack.gaiamobile.org/settings'

    Q. What happens when a langpack is uninstalled?

    Q. What happens when the target app is uninstalled?

In order to support uninstallation and allow to revert the values of prefs it 
might be necessary to keep the original value of each pref if a langpack wants 
to overwrite it.  Maybe `nsIPrefService.getBranch()` and `.getDefaultBranch` 
could be used?

    http://mxr.mozilla.org/mozilla-central/source/modules/libpref/nsIPrefService.idl
    

Language Negotiation and Resource IO, with Langpacks
----------------------------------------------------

`App.getLanguages()` for a given app now returns the newer version of `de`, as 
well as adds `pl` to the list of available languages.  The l10n.js library can 
negotiate the UI language using user's preferred languages stored in 
`navigator.languages`.

Once the negotiation has completed, specific resources can be fetched.  Again, 
`{locale}` is replaced by l10n.js with the result of the negotiation.

    app://{locale}.l10n.settings.gaiamobile.org/locales/settings.{locale}.properties

…becomes:

    app://pl.l10n.settings.gaiamobile.org/locales/settings.pl.properties

This URI is used to perform an XHR request by the l10n.js library.  The app 
protocol handler gets the value of the Gecko pref corresponding to the host of 
the requested URI:

    'webapps.overrides.pl.l10n.settings.gaiamobile.org':
      'my-langpack.gaiamobile.org/settings'

…and uses this value as the host name to create a new, valid URI:

    app://my-langpack.gaiamobile.org/settings/locales/settings.de.properties

…which points to an existing resource in the langpack.


Langpack Uninstallation
-----------------------

TBD.


App Upgrade & Uninstallation
----------------------------

TBD.


Generalization to other resources
---------------------------------

Settings HTML:

    <link 
      rel="stylesheet"
      href="app://theme.gaiamobile.org/css/theme.css">

Settings `manifest.webapp`:

    "name": "Gaia Settings",
    "overrides": {
      "theme.gaiamobile.org": "/",
    }

Saved pref on app install:

    'webapps.overrides.theme.gaiamobile.org':
      'settings.gaiamobile.org'

Theme pack (origin: my-theme.gaiamobile.org) manifest:

    "role": "themepack",
    "overrides": {
      "theme.gaiamobile.org": "/"
    }

Modified pref on theme pack install:

    'webapps.overrides.theme.gaiamobile.org':
      'my-theme.gaiamobile.org'
