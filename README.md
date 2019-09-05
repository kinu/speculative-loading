# [WIP] Cross-Origin Speculative Loading and Privacy Implications

Kinuko Yasuda, Sep 2019, (c) Google

_(Note: This isn't a proposal that's thoroughly thought out or stamped with the Google Seal of Approval, but more about trying to write down a collection of threat model and mitigation ideas that are found during the discussion)_

**TL;DR:** Examine what threat model cross-origin speculative loading (e.g. `prefetch`, `prerender` and `portal`) could have and reconsider how it could work with the new privacy goals on the web.

- [Terminology](#terminology)
- [Background](#background)
- [Threat Model](#threat-model)
- [Plausible Changes](#plausible-changes)
- [Acknowledgements](#acknowledgements)

## Terminology

- **Speculative loading** -- web platform features that are proposed and designed for speculatively fetching resources in order to accelerate next navigations.  Namely, pre* features that are defined in [resource-hints](https://w3c.github.io/resource-hints/) (`prefetch`, `prerender` etc) and emerging features like [Portals](https://github.com/WICG/portals) (where a page can be previewed before being navigated into). This document often just use the term 'speculative loading' for cross-origin speculative loading.
- **Referrer page** -- a source page that can initiate speculative loading by inserting a relevant HTML tag, e.g. `<prefetch>`, for the resource that is likely used by the next navigation.  Typically the page would have an anchor tag pointing to the same page for navigation.
- **Target page** -- a target page that can be navigated from the referrer page.  Its resource might be speculatively fetched by a speculative loading before the user actually navigates to the page, and if that's the case the navigation can finish more quickly.

## Background

(You can skip to the [threat model](#threat-model) section if you know the context well)

### Why Speculative Loading Exists
Speculative loading can be used to speculatively fetch resources that are likely used for next navigations, therefore a site can potentially accelerate next navigations. For example, when a referrer page A.com thinks that the user likely visits B.com, it can insert an HTML tag for speculative loading, e.g. <prefetch>, right next to an anchor tag to the target page B.com. If everything works as expected, when the user clicks the link for B.com some of the critical resources are already (speculatively) fetched and therefore the navigation finishes quickly.

### Speculative Loading and Double-Keyed Caching
However, as was pointed out in several places (e.g. [resource-hints/issues/82](https://github.com/w3c/resource-hints/issues/82)) this model does not work very well with the recent privacy goals on the web, especially for cross-origin use cases. For example, major modern browsers like Safari, Chrome and FireFox has shipped or has started experimenting with [Double-Keyed caching](https://github.com/whatwg/fetch/issues/904), where subresources are cached in a separate HTTP cache partitions so that the user’s browsing history on the current site (of the top-level page) is not shared with other sites, or vice versa. This partitioning clearly breaks how cross-origin prefetch works today, and considering other similar privacy implications it looks any cross-origin speculative loading would need to be re-considered.

## Threat Model

Cross-origin pre* must not enable any additional user tracking across sites.  User tracking that can happen between the referrer and target page upon actual navigation is out-of-scope of this document.

### Scenarios

(**Note** we think the threat model discussed here should also be examined and agreed to make sure we want to avoid these on the web. Many of these are possible today without using `prefetch` with 3p cookies and other technologies, but they are also being discussed if they should be like that forever.)

- The referrer site can prefetch a set of subresources (e.g. target.com/1, target.com/3, target.com/4) for next navigation, and then when a user navigates to the target site, it can learn that the user has the tracker ID 0b11010 from the set of subresources that are loaded from the cache (i.e. via timing info).
  - **Mitigation**: Top-level navigation only
  - **Alternate Mitigation**: Subresources that are to be loaded must be determined only by the target site (e.g. via a link header) but not by the referrer site

- The referrer site can create prefetch with a URL like “https://target.com/?from=referrer&id=12345”, then the target site can learn that a particular user on target.com has the tracker ID 12345 on the referrer site
  - **Mitigation**: Only allow uncredentialed requests
  - **Alternate mitigation that was considered**: Use the same code path as normal navigation so that prefetch inherits link decoration defense as it's added to normal navigation. For example, if normal navigation becomes credential-less when there are query parameters, that should carry over to prefetch. If normal navigation with query parameters deletes first-party cookies after a day, prefetch needs to arrange that the same flag is set in the event that it's "committed". Note that this is likely **insufficient** for user-did-not-click cases as the ID is shared without any user actions.

- The referrer site can prefetch multiple prefetches like “https://target.com/?id=12345” and “https://target.com/”, and then when a user navigates to one of the prefetched URL the target site can learn that the user who just navigated to target.com has the tracker ID 12345 on the referrer site (from the shared socket pool / TLS stack knowledge).
  - **Mitigation**: Use a separate, unique network stack (NIK, network isolation key) for pre* fetches. (This needs to be combined with the other mitigation for credentials, e.g. request needs to be uncredentialed)

- The referrer site can prefetch multiple links, then the tracker site can create N iframes or can cycle through the N sites (e.g. via redirects) to get N bits of timing info (i.e. by seeing which sites can be connected faster than others) to track the user.
  - **Mitigation**: the pre* fetched resources can be used only by the immediate next navigation

 - Timing attack: The referrer creates `<img src="https://referrer.example/about/to/prefetch/target.com/path">` and `<link rel=prefetch href="https://target.com/path">` at the same time. Later, the two servers correlate their logs and match up user IDs sent in cookies when those requests come in close to each other.
   - **Mitigation**: Similar to other threats. Needs uncredentialed requests and separate network stack.

- The referrer site prefetches a page that goes through multiple sites via a redirect. Later, the redirected sites and prefetched sites can correlate their logs and match up user IDs to associate the user information.
  - **Mitigation**: Similar to other threats. Needs uncredentialed requests and separate network stack.
  - **Alternative mitigation that was considered**: no redirects

- The referrer prefetches “https://target.com/foo/”, and observe onerror/onload to see if the resource is cached for https://target.com by timing information, which allows the referrer to retrieve the user’s cross-origin browsing history.
  - **Mitigation**: Similar to other threats. Needs uncredentialed requests and separate network stack.
  - **Alternative mitigation**: Stop firing onerror/onload for speculative loading.

## Plausible Changes

If we consider all the scenarios listed in the previous section are valid and need to be avoided, we think following requirements will need to be met:

- **No referrer**
- Requests must be **uncredentialed**
- Prefetched resources must be available **only to the immediate next top-level navigation**
- Requests must **not share network connections and state** (e.g. https://github.com/whatwg/fetch/issues/917)

(Here this also assumes the UA implements some form of [Double-keyed or partitioned HTTP cache](https://github.com/whatwg/fetch/issues/904))

### Opt-in Mechanism for Uncredentialed Navigations

Given that navigations are credentialed by default while we think uncredentialed requests will be needed for speculative loading, we’ll need a mechanism for a site to explicitly express that their document can be loaded in this way without credentials and yet to be used for a next navigation.  Otherwise a referrer site can force a victim page to be loaded without credentials to make the page appear as if broken.

One plausible way is to address this conflict is to add a response header that can tell the UA that the page can be safely prefetched and loaded without credentials, e.g. `Allow-Uncredentialed-Navigations`.

## Acknowledgements

Thanks a lot for sharing a lot of insights on the threat model section:
Jeffrey Yasskin, David Benjamin, Matt Menke, Josh Karlin, Dominic Farolino, Yoav Weiss, Brad Lassey, (probably more)
