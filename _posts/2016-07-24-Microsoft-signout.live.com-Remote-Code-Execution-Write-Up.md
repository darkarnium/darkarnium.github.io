---
layout: post
title:  Remote Code Execution (RCE) on Microsoft's 'signout.live.com'
date: 2016-07-24 00:00:00.000000000 -08:00
author: Peter Adkins
image:
---

**TL;DR:** The combination of a less than great vulnerability handling processes by Adobe, and the use of default credentials by Microsoft yielded remote code execution on the `signout.live.com` domain.

The following remote code execution vulnerability in the `signout.live.com` service was reported to the Microsoft Security Response Center in late 2015 and has since been patched. This vulnerability was the result of an operational configuration error, as well as another vulnerability inside of the Adobe Experience Manager (AEM) installation used to provide this service.

Due to the circumstances around this RCE, details of a previously unpublished and potentially 'silently patched' vulnerability in the Adobe Experience Manager (AEM) Dispatch module will also be covered (CVE-2016-0957).

## AEM Overview

Adobe Experience Manager (AEM) is an 'enterprise grade' content management system sold and maintained by Adobe Systems. The core components of this system run inside of a JVM, with an optional Apache HTTP server module provided for 'caching and/or load-balancing'.

### AEM Architecture.

Upon first encounter, AEM appears to be a fairly typical enterprise application made of pain, suffering and Java. These feelings are quickly realized upon running the `quickstart` installation package which consists of a single several-hundred megabyte JAR that unpacks neatly into a tree of hatred and self loathing.

Under the hood, this 'stock' AEM deployment consists of a vast array of open source products and some Adobe brand glue. Rather than the services which comprise AEM being deployed in a 'traditional' manner, they are instead implemented as components inside of an Apache Felix based Open Services Gateway initiative (OSGi) framework.

![A view of the AEM internal architecture, taken from the AEM 5.6.1 documentation.](/assets/article_images/2016/aem-layout.png)

The advantage of this system is that these components, known as OSGi 'bundles', can be installed, restarted, or re-configured without the need to restart the OSGi framework or underlying JVM. In addition, this architecture allows for the extension of AEM through the development and installation of custom OSGi bundles.

### Deployment Topology

A 'typical' AEM deployment consists of three distinct tiers:

* Author
  * Provides an authoring environment for content (and other data).
  * Publishes data to Publish nodes via replication queues (push).
  * Stores content in a JCR compliant content repository.
  * Runs in a JVM.
* Publish
  * Receives published content from the Author nodes.
  * Serves published content to Dispatch nodes.
  * Stores content in a JCR compliant content repository.
  * Runs in a JVM.
* Dispatch
  * Serves and caches content from Publish nodes to the end-user.
  * 'Proxies' requests back to the Publish farm if objects are not in cache (pull).
  * Caches content on disk as rendered objects.
  * Runs as an Apache HTTP Server module.

An example diagram of this style of tiered deployment can be found below:

![A common three tier deployment, taken from the AEM 5.6.1 documentation.](/assets/article_images/2016/aem-flow.png)

In order to improve the security posture of an AEM installation, these tiers are typically deployed with the Author and Publish tiers protected from the world through network segmentation and/or access controls. The Dispatch tier is generally the only tier 'open' to the internet, providing a mechanism to retrieve and cache content from the Publish tier (in a manner not unlike that of a reverse-proxy).

### Dispatch Filtering

As a result of the Dispatch tier pulling data from upstream Publish nodes, the Dispatch module implements a 'filtering' mechanism in order to mitigate abuse. This filter is especially important given that nodes in the Publish tier serve both content and administrative resources via the same Apache Sling service.

As an example of why this filtering is required, the following URLs on the Publish node `publish.example.org` are able to be accessed without any authentication:

* `https://publish.example.org/etc/reports/diskusage.html`
  * Provides a browsable view of all data in the content repository.
* `https://publish.example.org/content/www-example-org/en_US/example.html`
  * Renders an example page for the public 'www.example.org' website.

However, if accessed via the Dispatch tier - assuming a default Dispatch configuration with an empty cache - the following should be true:

* `https://Dispatch.example.org/etc/reports/disusage.html`
  * Filtered by the Dispatch tier, with an HTTP 404 served to the requestor.
* `https://publish.example.org/content/www-example-org/en_US/example.html`
  * Fetches a rendered example page for 'www.example.org' from the Publish tier.
  * Serves the fetched page to the requestor.

In order to implement these restrictions, the *default* AEM Dispatch module configuration contains a set of filters which operate in a default 'deny' manner: If a resource hasn't been explicitly allowed inside of a `filter` block, requests for that resource would be denied.

In order to better demonstrate this configuration, an excerpt from an example Dispatch configuration file - taken from the AEM 5.6.1 'security checklist' - has been included below:

```
 # only handle the requests in the following acl. default is 'none'
 # the glob pattern is matched against the first request line
 /filter
   {
   # deny everything and allow specific entries
   /0001 { /type "deny"  /glob "*" }
   /0023 { /type "allow" /glob "* /content*" }
   ...
   # enable specific mime types in non-public content directories
   /0041 { /type "allow" /glob "* *.css *"   }  # enable css
   /0042 { /type "allow" /glob "* *.gif *"   }  # enable gifs
   ...
  }
```

The end result of this configuration is that the ability to pull Publish tier administrative resources through the Dispatch tier should be prevented.

...or perhaps not?

## CVE-2016-0957

CVE-2016-0957 is a very simple vulnerability brought about by the unexpected and improper behavior of the `glob` filter inside of the AEM Dispatch module. The net result of this vulnerability is that `glob` filters can be trivially 'coerced' into returning an `allow` match for resources which may otherwise be denied. This coercion is possible due to `glob` filters matching on not only the requested resource URL, but also on any included HTTP query parameters.

Exploiting this vulnerability is as simple as appending a known-allowed resource path onto a filtered URL as an HTTP query parameter.

An example of this bypass can be found below; assuming the use of a configuration similar to that listed above:

* `https://Dispatch.example.org/system/console`
  * Implicitly denied by the Dispatch filter due to rule `0001`.
  * Does not match any subsequent rules.
  * Access is denied.
* `https://Dispatch.example.org/system/console?.css`
  * Implicitly denied by the Dispatch filter due to rule `0001`.
  * The `.css` URL query parameter coerces the `glob` engine into matching rule `0041`.
  * Access is permitted.

### Impact

Depending on the version and configuration of the affected AEM installation, the above vulnerability could expose the Publish tier to a number of vulnerabilities, including:

* `/libs/opensocial/proxy`
  * Provides a proxy which is able to be used to perform arbitrary server-side requests.
* `/etc/mobile/useragent-test.html`
  * Exposes a reflected Cross-Site Scripting (XSS) vulnerability in older versions of AEM 5.X.
* `/etc/reports/diskusage.html`
  * Exposes an unauthenticated, browsable view of all content in the repository which may lead to information disclosure.

### Reporting.

This behavior was initially observed inside of an AEM 5.X environment which utilized a default Dispatch configuration towards the end of 2015. When discovered, this issue was reported to the Adobe PSIRT as a potential security vulnerability. A number of days after this report was submitted, the Adobe PSIRT advised that this was a known issue with the Dispatch module and had been 'addressed' in version 4.1.5 onwards.

As the reported behavior was observed in version 4.1.9 of the Dispatch module, a subsequent email was sent to the Adobe PSIRT in order to request additional information.

After some time, the Adobe PSIRT detailed that the reported issue had been previously discovered internally and that version 4.1.5 of the Dispatch module onwards contains a `url` filter directive which should be used in place of `glob` filters.

In order to confirm suspicions that this issue had been 'silently patched' by Adobe, all security advisories and release notes for the Dispatch module were reviewed. In the end, only a single-line statement relating to this change was able to be found - which was found in a `CHANGELOG` file, inside of the 4.1.5 Dispatch module release tarball:

> DISP-407 - Security Checklist: Default Dispatch rules can be circumvented by query-string

Further to this statement, no additional information appeared to have been published relating to this vulnerability.

As a result of these findings, an additional email was sent to the Adobe PSIRT expressing concerns related to the handling of this vulnerability and a retrospective security advisory was requested.

On February 9 of 2016, Adobe raised APSB16-05 which formally allocated CVE-2016-0957 to this vulnerability, and disclosed that 'Dispatch 4.1.5 and higher resolves a URL filter bypass vulnerability that could be used to circumvent Dispatch rules'.

Unfortunately, due to the nature of this vulnerability, simply upgrading the Dispatch module does not appear to mitigate this vulnerability. In order to mitigate, the Dispatch module must not only be updated to at least version 4.1.5, but any `glob` filters defined in the Dispatch configuration should be replaced with `url` filters.

## The world's lamest RCE.

With an overview of both AEM and CVE-2016-0957 out of the way, the following section describes an example where the combination of this filter bypass, and the misconfiguration of the AEM Publish nodes used by `signout.live.com` were able to be used together in order to allow for the execution of arbitrary code.

The discovery of this issue came about through regular interaction with the Microsoft Live service, rather than through active testing. At the time, the `signout.live.com` domain appeared to be used as a logout 'landing' page for the Microsoft Live service.

![The `signout.live.com` landing page.](/assets/article_images/2016/signout-landing.png)

When this `signout` redirect was first encountered, it was noticed that the URL structure looked suspiciously like it may have been generated by an AEM. In order to confirm these suspicions, the body of the rendered HTML page was examined for the presence of a number of common AEM components. The presence of a handful of Javascript libraries indicated that this page was at least generated by AEM.

As this was following the discussed interactions with the Adobe PSIRT, an attempt was immediately made to see whether the default `glob` style filters were in use. This was done by requesting the URL for the AEM OSGi console with an HTTP query parameter of `.css` appended:

> https://signout.live.com/system/console?.css

As this request was met with an HTTP 401, a subsequent request without any HTTP query parameters was performed. This second request being met with an HTTP 404 confirmed suspicions that the first request had successfully bypassed the Dispatch filters.

In order to verify that this was correct, this same URL was accessed using a web browser both with and without the query parameter. As expected, the former request successfully bypassed the Dispatch filter and resulted in an HTTP Basic authentication prompt:

![The `signout.live.com` OSGi console login.](/assets/article_images/2016/aem-osgi-login.png)

Given that it was possible to bypass the Dispatch filters, it was initially thought that it may have been possible to brute force credentials for an AEM built-in administrative accounts in order to gain access to the OSGi console. However, before getting that far, and in a "*what if..?*" moment, the default credentials of `admin / admin` were attempted.

![Wat.](/assets/article_images/2016/aem-uhoh.png)

In a moment of utter disbeleief, it appeared that these default credentials had been accepted and full-administrative access to the AEM Publish nodes' OSGi console had been granted. In order to confirm that this was valid, a number of subseqent requests inside of the OSGi console were performed, all of which completed successfully.

![Double wat.](/assets/article_images/2016/aem-licence.png)

At this stage, it would have been possible to execute code inside of the JVM through the upload of a custom OSGi bundle. However, the question was whether it was possible to escalate access further - as a purely hypothetical and 'off instance' exercise, as no code was attempted to be loaded into the Microsoft system at any time.

As part of this exercise, a list of loaded OSGi bundles in a generic AEM 5.X deployment was reviewed, where it was noted that `org.apache.commons.exec` was loaded. This bundle appeared to be an implementation of the Apache Commons Exec library, which provides a method to '...reliably execute external processes from within the JVM'.

In order to confirm whether this library was able to be used, a quick proof-of-concept which utilized both this library, as well as the OSGi `BundleActivator` interface was developed. This OSGi bundle was configured in such a way that when loaded, the `org.apache.commons.exec` library would be called and a `ping` command would be fired against an external server.

Due to the nature of the AEM OSGi framework, once installed, this module would persist inside of the system and would be loaded automatically at system start-up.

Ignoring POM files and associated boilerplate, a simple 'command executor' OSGi bundle was able to be implemented in as few as 18 lines of Java - and likely in less by someone who knew what they were doing :)

```Java
public class ProviderActivator implements BundleActivator {
    String staticCommand = "C:\\Windows\\System32\\ping.exe www.example.org";

    @Override
    public void start(BundleContext bundleContext) throws Exception {
        CommandLine cmdLine = CommandLine.parse(staticCommand);
        DefaultExecutor executor = new DefaultExecutor();

        try {
            executor.execute(cmdLine);
        } catch(java.io.IOException ex) {
            ex.printStackTrace();
        }
    }

    @Override
    public void stop(BundleContext bundleContext) throws Exception {
    }
}
```

**Note:** As mentioned above, no code was loaded into the Microsoft system **at any time**. The proof-of-concept code prepared as part of this exercise was compiled and provided as part of the report to Microsoft, but was never loaded into the system.

### Reporting.

This vulnerability was reported to the MSRC on the 3rd of December 2015, and was both confirmed and assigned a case manager within 24-hours. After some time and back-and-forth with the MSRC, this vulnerability was confirmed to have been resolved on the 3rd of May 2016.

Unfortunately, on the 4th of May 2016 it was confirmed by the MSRC that this report was not eligible for a monetary reward under the Microsoft Online Services Bug Bounty program as the affected domain was not explicitly listed as in-scope.

Such is life! :)

### Thanks.

I'd like to extend thanks to all of the MSRC staff who were involved in this case. Although this case took quite some time to be resolved, all of the MSRC staff encountered throughout were a pleasure to work with.
