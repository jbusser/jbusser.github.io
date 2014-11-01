---
layout: post
---

#### Use Poltergeist's `page.driver.network_traffic` collection to detect inline HTTP requests

We use Google Analytics at my job, and we have implemented [Universal
Analytics](https://developers.google.com/analytics/devguides/collection/analyticsjs/).
I expected that testing thest calls with Capybara would be straightforward: Implement
WebMock, stub a response, assert that the page made the expected XHR.

Nope.

The trouble is that `analytics.js`, Google Universal Analytics' library, makes inline HTTP
requests to a GIF to report user activity.

OK. So how do you detect that a browser has made an expected inline image request?

The answer lies in the Poltergeist docs, specifically the [NetworkTraffic
collection](http://www.rubydoc.info/github/jonleighton/poltergeist/Capybara/Poltergeist/NetworkTraffic).

Given that we have HTML in our templates that contains code like

<code data-gist-id="d9f4b1b2dca9505b97f7" data-gist-file="new.html.erb"></code>

How do we test it? Specifically, how can we test the function `categoryLabel()` so that we
know it does the right thing?

The function `categoryLabel()` uses `purl.js` to parse parts of the URL. That's what gives
us `$.url()`. The query param `ga_category_label` is optional, so we want to supply a
default should one not be passed in. To test this, we can write a Capybara spec like
this:

<code data-gist-id="d9f4b1b2dca9505b97f7" data-gist-file="users_sign_in_spec.rb" data-gist-highlight-line="22-26"></code>

The money code is lines 22-26. That's where we can inspect Poltergeist's collection of
inline HTTP requests. We pass that through `google_analytics_requests` to find only those
inline requests that go to Google Analytics, and we then determine whether any of those
requests contain the query param we expect.
