# My URL To API

## Overview
This service provides stateless link routing and stateful URL shortening for the ecosystem.

## Instructions
Agents **do not** need to make a POST request to create a short link. Internal Go services will automatically handle the shortening of Stripe Links or other internal references when passed back to the user.

However, if an agent is specifically asked to shorten a URL on behalf of a user in a custom workflow, they should use the POST /url/shorten endpoint (not currently active for public API use).

When returning shortened URLs to users, the format will always be:
`https://api.myurlto.com/{short_code}`
