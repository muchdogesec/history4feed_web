# history4feed Web

## Overview

history4feed Web is a paid version of [history4feed](https://github.com/muchdogesec/history4feed) Web is a paid version of history4feed.

history4feed Web is ultimatley designed to be a multi-user version of history4feed.

High level mockups and designs can be found here: https://miro.com/app/board/uXjVNnKRE_U=/?share_link_id=597498628214

## Enhancements over history4feed

### Introduce outside services

* Auth0: Users and Roles
* Stripe: Payments and Subscriptions
* Send In Blue: Mailing list

### Introduce Authentication and Users (Customers)

Anyone can sign up to history4feed Web.

All sign ups are handled by Auth0.

A user must confirm their email before being granted access to the web app. They will continue to see a verify email page (with link to resend).

This is managed by Auth0: https://auth0.com/docs/manage-users/user-accounts/verify-emails

On sign up user can opt into mailing list (Send in Blue), this is managed by Auth0 on sign up (https://auth0.com/blog/using-auth0-to-collect-consent-for-newsletter-signups/) (user who marks as false, is added to list as unsubscribed), and managed directly in the web app with send in blue (my account).

A user can change their email at any time: https://community.auth0.com/t/let-users-change-their-email-address/109271 . However, they must confirm email verification before history4feed Web treats is at their email (they can resend verification at anytime)

A user can change their password at anytime: https://auth0.com/docs/authenticate/database-connections/password-change (if 2fa enabled, they must enter code)

All users can optionally enable 2FA at anytime in Auth0: https://auth0.com/docs/secure/multi-factor-authentication (use device codes only, not SMS)

A user can delete their account. This will:

* delete history4feed web data
* cancel any h4f Stripe subsciptions, and delete the Stripe customer (delete only happens if no other DOGESEC subscriptions exist)
* delete any h4f Auth0 permissions and delete the Auth0 customer (deletion of Auth0 happens if no other DOGESEC subscriptions exist)
* delete customer from h4f mailing list (Send in Blue), if subscribed

### Introduce products, product subscriptions and payments

Almost all of this logic can be handled by Stripe because our subscription/customer/product is very typical of SaaS products.

As such we can also use dj-stripe to make this super simple: https://dj-stripe.dev/2.8/

![](https://b.stripecdn.com/docs-statics-srv/assets/abstractions.c0365799e62eac96eed3e9e746e3b65b.svg)

https://docs.stripe.com/billing/subscriptions/overview

We use Stripes flat rate pricing model, with users choosing to pay monthly / yearly.

https://docs.stripe.com/products-prices/pricing-models#flat-rate

![](https://b.stripecdn.com/docs-statics-srv/assets/pricing_model-flat-rate.4f63dae2c4f7078ae10f30324539b0cc.png)

On sign up, we create a customer record for the user in Stripe. To support this we will offer a no-cost product that user (customer) is automatically signed up to on registration (you can see how this products is set later in this doc).

A user can change plan at anytime: https://docs.stripe.com/billing/subscriptions/upgrade-downgrade?locale=en-GB

If they upgrade, prorated payment for remaining cost increase for billing period will be taken. If downgrade, user will stay on same plan until end of billing period (in other words; there are no refunds).

Here is a rough idea of the products we will create in Stripe

* product 1
	* name: Free
	* description: Free tier
	* price monthly: $0.00
* product 2
	* name: Starter
	* description: Starter tier
	* price monthly: $10.00
	* price yearly: $70.00
* product 3
	* name: Pro
	* description: Pro tier
	* price monthly: $15.00
	* price yearly: $105.00

Note, the free product is always charged monthly.

We need to add afew extra attibutes into dj-stripe for products:

* is_default_products (boolean, must be assigned to one product, but no more than one product): defines the product user will be subscribe to on registration
* is_visible_in_ui (boolean): defines if selectable in the UI. Useful for hiding old products/custom products from users in UI
* is_archive (boolean): defines if product is archive (also useful for managing old products)

### Introduce roles

To charge users we have a simple pricing model. A user subscribes to a product based on the number of feeds they want to describe to. This defined as a role.

Roles are manages in Auth0: https://auth0.com/docs/manage-users/access-control/configure-core-rbac/rbac-users/assign-roles-to-users

When user changes Products, the roles linked to them in Auth0 are updated accordingly.

Roles are super simple at this point in time (although might be extended in the future). They define the number of feeds a user can access.

A product must have one Role and only one Role.

### Introduce feed subscriptions

The history4feed Web model assumes all feeds added have no owner.

A user in history4feed Web can subscribe and unsubscribe to a feed in history4feed as they want.

Some notes on this logic:

* a feed URL always has a unique record. i.e. if more than one user subscribe to the same feed url, they are subscribed to the same entry
* once a feed is added, a feed always exists in history4feed Web and continues to be polled, even if all users have unsubscribed
* a user can only see feeds they are subscribed to (this is managed by a seperate DB in web app). They can add a feed at anytime. If the feed already exists in history4feed Web, they are simply subscribed to the existing record for it. If it does not exist, a new record is added and backfill job triggered

### Introduce automated feed polling

history4feed requires users to manually run a request to `PATCH /api/v1/feeds/<feed_id>`.

history4feed Web will automatically update each blog on a set schedule.

Schedules can be set on a per feed level. The schedule will define when queries to this endpoint are run for the feed to ensure the post are kept updated at a suitable schedule. The default schedule is 4 hours.

Note, in history4feed Web a user cannot run requests to this endpoint directly and must wait for the default schedule to run.

## Public API

A user sees a similar version to the backend API, with some minor changes (and addition of one endpoint).

### Authentication

All queries to the API require authentication.

Authentication defines the Role of the user, and thus the data that can be returned to them.

All requests require an `X-API-Key` in the header (unless otherwise stated). User API keys can be generated in the user interface.

### Available public endpoints

#### GET Feeds

Same as history4feed, but also includes:

* API Router adds filter to the request to limit only feeds they are subscribed to.

#### POST Feeds (Subscribe)

Same as history4feed, but first checks

* if user has enough remaining feeds
* if feed exists already in db, if not creates it
* subscribes a user to that feed in the web db

#### POST Feeds (Unsubscribe)

Unlike core history4feed, a user cannot delete a feed (as feeds do not belong to a user in history4feed_web).

```shell
POST /api/v1/feeds/{feed_id}/unsubscribe
```

Will unsubscribe authenticated user from the feed in the web db.

#### GET Feed ID

Same as history4feed, but also includes:

* will only ever return results for feed ids users are subscribed to

#### GET Posts (JSON)

Same as history4feed, but also includes:

* will only ever return results for feed ids users are subscribed to

#### GET Posts by ID

Same as history4feed, but also includes:

* will only ever return results for feed ids users are subscribed to

#### GET Jobs

Same as history4feed, but also includes:

* will only ever return results for feed ids users are subscribed to

#### GET Job

Same as history4feed, but also includes:

* will only ever return results for feed ids users are subscribed to

## CI/CD

This repository stores keys in Github.

It uses Github actions to deploy automatically.

It is deployed using docker.