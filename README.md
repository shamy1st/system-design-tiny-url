# TinyURL Design

Let's design a URL shortening service like TinyURL. This service will provide short aliases redirecting to long URLs. 

Similar services: bit.ly, goo.gl, qlink.me, etc.

Difficulty Level: Easy 

## What is TinyURL?

* URL shortening is used to create shorter aliases for long URLs.
* Users are redirected to the original URL when they hit these short links.
* Short links save a lot of space when displayed, printed, messaged, or tweeted.
* Additionally, users are less likely to mistype shorter URLs.
* The shortened URL is nearly one-third the size of the actual URL.
* URL shortening is used for optimizing links across devices, tracking individual links to analyze audience and campaign performance, and hiding affiliated original URLs.
* example:
      
      https://github.com/shamy1st/system-design-tiny-url/edit/main/README.md
      http://tinyurl.com/uxs6dlc

## 1. Requirements

### Functional Requirements

1. Given a URL, our service should generate a shorter and unique alias of it. This is called a short link. This link should be short enough to be easily copied and pasted into applications.
2. When users access a short link, our service should redirect them to the original link.
3. Users should optionally be able to pick a custom short link for their URL.
4. Links will expire after a standard default timespan. Users should be able to specify the expiration time.

### Non-functional Requirements

1. The system should be highly available. This is required because, if our service is down, all the URL redirections will start failing.
2. URL redirection should happen in real-time with minimal latency.
3. Shortened links should not be guessable (not predictable).

### Out of Scope Requirements

1. Analytics; e.g., how many times a redirection happened?
2. Our service should also be accessible through REST APIs by other services.

## 2. Design Considerations

## 3. Estimation

## 4. High-level Design

## 5. Database Model

## 6. System Interface


