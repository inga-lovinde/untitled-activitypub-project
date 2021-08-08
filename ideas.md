
- [Vision](#vision)
- [Social and federation features](#social-and-federation-features)
  * [Administration](#administration)
  * [Federation between instances](#federation-between-instances)
  * [User relationships](#user-relationships)
  * [Posts visibility (just a collection of thoughts; further discussion required)](#posts-visibility--just-a-collection-of-thoughts--further-discussion-required-)
  * [Timelines](#timelines)
    + [Community](#community)
    + [Friends](#friends)
    + [Mutuals](#mutuals)
    + [Custom timelines](#custom-timelines)
    + [Untagged](#untagged)
  * [Posts auto-removal](#posts-auto-removal)
  * [Handling username and userpic changes](#handling-username-and-userpic-changes)
  * [Additional features](#additional-features)
- [Technical and privacy considerations](#technical-and-privacy-considerations)
  * [Storage](#storage)
  * [Server-client architecture](#server-client-architecture)
    + [Basics](#basics)
    + [Telegram or Matrix bridge](#telegram-or-matrix-bridge)
    + [Email slow mode](#email-slow-mode)
    + [Web UI](#web-ui)
    + [Pitfalls](#pitfalls)
  * [Technical details](#technical-details)
    + [Resource usage](#resource-usage)
      - [Executions number](#executions-number)
      - [Execution time](#execution-time)
      - [Bandwidth](#bandwidth)
      - [Table Storage operations](#table-storage-operations)
      - [Table storage, blob storage](#table-storage--blob-storage)
    + [DB structure](#db-structure)
    + [Public pages](#public-pages)
    + [Technical tools](#technical-tools)
    + [Management UI, security](#management-ui--security)

## Vision

This part of fedi could look like this:
Tightly-knit communities (polycule, WG etc) run their own instance, so everybody personally knows their admin (who is just maintaining the instance).
Communities connect to each other, forming larger fuzzy structures.
Every instance is several (up to ten or maybe twenty) users only.

Federated social network should be social (as in actual people interacting with each other, not crossposter bots or large corporations), and federated with opt-in consent, where an instance consents to participate in some specific federation, and the other federation members agree.

Server software should be extremely lightweight; everybody should be able to self-host an instance on their existing computer (including Raspberry Pi etc) without any significant performance degradation.
Or, alternatively, they should be able to not do any manual management and to host it elsewhere for no more than $1/month.

It should be simple to install and use, and have as little dependencies as possible.

There is no need for a formal in-instance moderation, because everybody within an instance know each other.
Instances with bad actors can either solve their bad actors problem or be defederated from.
By default, a new instance has all instances quarantined.

## Social and federation features

### Administration

Ideally, all instance members could be its admins, but of course not everybody could be willing to spend time doing so.

References to "admin" in the following text are supposed to mean "whoever wants to deal with admin stuff",
not "a privileged user who decides how things are run for everybody else".

An useful metaphor is a home.
Several people live there; some of them are doing the chores and some aren't, but everybody gets a say in home matters.
And they might invite some guests to live there for a while, or uninvite them.

### Federation between instances

Considering specific instance: all instances known to it fall under four categories:

* Community
* Connected
* Quarantined
* Disconnected

By default any other instance gets into "Quarantined" category, meaning that all events it sends to our instance are getting quarantined;
for example, an user won't see a mention in their "Mentions" if it comes from a previously unknown instance;
they'll see it in their "Quarantined mentions" if they are willing to check it.

An instance admin can move a remote instance into "connected" or "disconnected" categories.
If it's disconnected, no events from it will be handled; additionally, no reposts of its activities will be handled.
If it's connected, all events it sends will get into timelines and "Mentions" etc.

Additionally, if a remote instance runs the same software, our and remote instance's admins can agree to form a community,
so that their instance will be in "Community" category for our, and our will be in "Community category" for their.

_The "Community" relationship is non-transitive; B might be in a "Community" category for A, and C in a "Community" category for B; yet C can be in any category for "A"_

Additionally, as a stopgap measure, an user can move a remote quarantined instance into "Connected",
or remote connected or quarantined instance into "Disconnected".
This change will affect this user only.
Ultimately and ideally, an instance-wide decision should be made, and all per-user overrides are supposed to be temporary.

The "Community" category brings us to timelines (see below).

### User relationships

All accounts are locked by default (so all follows have to be approved).

In addition to "following" and "follower", there will be a "friend" relationship.

An user can follow someone in an ordinary way, or alternatively "friend"-follow them.
If an user "friend"-follows someone, follow requests from that someone will be auto-approved.

Four kinds of relationships are recognized by an instance: "following", "follower", "mutual" and "friend".

(_Question: should "friend" always require a mutual following relationship? If I try to friend-follow someone, should they only be considered my friend if they follow me?_)

### Posts visibility (just a collection of thoughts; further discussion required)

There are three primary dimensions to the post visibility:

* On what timelines will it appear when posted?
* What users will be able to see it (mostly applicable to instances running other kind of software; see technical and privacy considerations below)
* Is it boostable?

With vanilla Mastodon you basically have (for simplicity leaving out the universal fact that everybody mentioned will see a post in their mentions):
* Public:
  * It appears on home timelines of everybody who follows you, plus on federated timelines of their instances, plus on your local timeline
  * Everybody are able to see it
  * It is boostable
* Unlisted:
  * It appears on home timelines of everybody who follows you
  * Everybody are able to see it
  * It is not boostable
* Followers-only:
  * It appears on home timelines of everybody who follows you
  * Only they are able to see it
  * It is not boostable
* Direct:
  * It appears on home timelines of people mentioned in the post, as long as they follow you
  * Only they are able to see it
  * It is not boostable

Hometown and glitch-soc add another option (which does not interact with activitypub in any way because these posts are not federated)
* Local
  * It appears on home timelines of everybody from your instance who follows you, plus on your local timeline
  * Only people from your instance are able to see it
  * It is not boostable (or is it?)

Ideally, an user would be able to choose:
* On what home timelines it will appear?
  * Followers
  * Mutuals
  * Friends
  * Local followers
  * Local mutuals (do we need this one, considering how small an instance would be?)
  * Local friends (do we need this one?)
* On what other timelines it will appear?
  * Should it appear on the local timeline?
  * Should it appear on the "Community" timeline?
  * It should probably never appear on "federated timeline" of instances running other software (such as Mastodon), so they'll probably always have a legacy "Unlisted" flag set
* Who will be able to see it?
  * Only those who will get it into their home timelines
  * Everybody (meaning people on instances running other software, as long as this post federated to them; see technical and privacy considerations below)
* Is it boostable?
  * If it's not boostable, we can set the legacy "followers-only" flag so that even on Mastodon instances only followers will be able to see it
  * If it's boostable, all visibility bets are off.

However, there probably are limitations to this caused by activitypub limitations and logic of other instances (such as Mastodon) so it requires further analysis.

### Timelines

#### Community

The society probably progressed past the need for federated timeline in its present form.
However, a community timeline would probably be useful.
Which will contain all listed posts from all the instances from the "Community" category.

This also implies that all listed non-local posts made on local instance will federate to all "Community" instances.

(Note that only instances supporting the "Community" feature can be in "Community" category; so Mastodon instances will never get there.)

#### Friends

In addition to having a home timeline (posts by people you follow), there will be a "Friends" timeline (posts by friends)

#### Mutuals

Should there be another timeline for mutuals, similar to the one for friends?

#### Custom timelines

An user is able to add tags for someone they follow; and to subscribe to a timeline with a corresponding tag (similar to what Mastodon has).

#### Untagged

Home timeline except for everybody who is tagged.

_A question: should it also exclude friends and mutuals? In other words, should it be "everybody who is not tagged" or "everybody who is not on other timelines"?_

### Posts auto-removal

With Mastodon, people are using all kinds of external apps to delete their old posts, and there is a pull request into vanilla mastodon to make it a built-in feature: https://github.com/mastodon/mastodon/pull/16529

_Question: is this feature needed here? How would it look like? Could it be that the reasons for its popularity in mastodon are not applicable here? (See technical and privacy considerations)_

### Handling username and userpic changes

As described in technical and privacy considerations, some of the clients will probably not even display an userpic.

Additionally, as opposed to mastodon, there is no central user registry with display names and userpics; every post is supplied with the detailed user information.
(This information will be text-only though, so it will contain an URL of the userpic on the remote instance.)

So probably an user will be able to sign off with a specific person's name or to choose a specific pronouns every time they post, which should be convenient to plural users or to those whose pronouns may change frequently.

_Question: should an user be able to pick an entirely new display name on every post, or does it make sense to limit it to postfixes only (e.g. an "user name" can post a specific post as "user name (fae/faer)" or "user name. -Person Name")? Changing the entire username will probably beneficial to some, but on the other hand it might make their posts less accessible to others, who might have a difficulty recognizing an user. And if that's the point, then maybe these users should register themselves separate accounts for every display name instead (and anyway there is an option of changing one's display name; this question only refers to the ability to modify a display name in the post compose UI)? Thanks to 🐉 for the idea_

### Additional features

Similar to username and userpic, a client will receive a relationship status along with a post, and will be able to display it.

(Similar feature request for mastodon: https://github.com/mastodon/mastodon/issues/15403)

## Technical and privacy considerations

### Storage

Server should only store local posts, and never store posts from other instances.
The only data from the other instances server should store is the data needed to manage follow relationships.

Not only will that improve privacy; it will also reduce the storage requirements.

### Server-client architecture

#### Basics

The idea is to have a server software that will deal with ActivityPub and user relationships, and clients (which also have their backends) actually presenting timelines to users.

A server would be an activitypub instance, with all the activitypub stuff implemented.
It would handle all the activitypub requests, it would know who follows who, and so it would know who is supposed to get this remote activity into their timeline, and what instances are supposed to get this activity made by a local user.

It would also provide public user profiles, but it would _not_ display any user posts.

A client can subscribe to push notifications for a specific timeline for a specific user, this is the only way for clients to get data from the server.
So whenever server gets any incoming activitypub activity, it knows what clients should get this activity, and it will send relevant requests to these clients.

Everybody can write a client; server knows nothing about specific clients.

Some possible examples of clients would be:

#### Telegram or Matrix bridge

Scenario: an user sets up a chat room involving a Telegram or Matrix bot; connects a bot in this room to their instance using oauth; subscribes to some timelines.
For example, an user could have one chatroom with their home timeline, and another (with notifications enabled) with their mentions, and a third with quarantined mentions.
Or subscribe to all of these from a single chatroom.

An user would also be able to reply to the posts, or star them, or boost them within that chat room, or follow users, etc.

That way, an user would be able to use their fedi account on any platform that has a Telegram client; would have infinite timelines; and would have the ability to search within timelines, something that existing ActivityPub software lacks.

(Right now it is impossible to implement with Mastodon; relevant issue is https://github.com/mastodon/mastodon/issues/15638 )

Such a client would have negligible storage requirements; it would only need to associate chat rooms with oauth sessions and subscriptions, which is a handful of DB entries per active user.

It will not support custom emojis (instead only displaying a text code).

This bridge will be implemented as a first client, among with the server.

#### Email slow mode

Scenario: once a day an user (who only has internet access for one minute per day) receives an email digest of what has happened on the fediverse during this day.
They'll be able to compose a response to some of the posts from the digest while being in offline mode, and the next day, when they're online again, send an email message containing these responses, which will then be handled as appropriate.

Such a client would have to store a day worth of posts in order to build a digest at the end of a day.

(Thanks to cadence for the inspiration)

#### Web UI

One could also implement something similar to the Mastodon Web UI or Pleroma-FE.

However, such a client would have to store a content of the entire timeline in order to display it, because activitypub server does not store posts coming from the other instances and does not support client fetches.

Maybe such a client could store a content of the entire timeline and expose Mastodon API, so that the existing frontends (Mastodon Web UI, Pleroma-FE, Pinafore, Sengi, Tusky, etc) would be able to run on top of it.

#### Pitfalls

_Question: how do we handle remote status deletion? Of course, clients should be getting push notification related to these events, but what if email client has already sent the digest to the user? Or how would telegram bot know what telegram message to delete unless it maintains a database storing the correspondence between activitypub posts and telegram messages?_

_Question: all images should have descriptions when possible, and all clients should warn users prior to boosting posts with uncaptioned images. Clients should also warn users prior to posting posts with uncaptioned images (and making captioning a part of a workflow), but some users cannot caption images, so there should probably be an option to disable these warnings. Should it be per-user or per-client?_

### Technical details

#### Resource usage

The goal is to be able to run an instance of a reasonable size for less than $1/month with managed providers.

$1/month rules out VPSes automatically, only leaving pay-per-use cloud providers such as Azure, with serverless functions (or "cgi scripts running on other people's computers", as 🦉 would say).

And with them, $1/month rules out relational databases automatically. But with the proposed storage scope, relational databases are not required.

With Azure, $1/month gives you any of the following:
* 20GB of Table Storage
* 27M Table Storage operations
* 50GB of Blob Storage
* 50GB of intra-continental or 20GB of inter-continental bandwidth (for instances hosted in Europe and North America)
or any combination of these (for example, one-fifth of each), and additionally:

* 400K GB-s execution time for serverless functions (free tier)
* 1 million executions of serverless functions (free tier)

I'm getting about a thousand of posts in my timeline per day. An instance with ten users like that would get around 300K posts per month. Including other activitypub activities (likes etc), it probably comes to 400K activities per month, just to be on a safe side.
Every activity should be handled.

Additionally, there is probably going to be a similar number of outgoing activities times instances they're supposed to reach.

##### Executions number

400K incoming requests probably prevents using any kind of a queue (receive incoming activity, determine which users should know about it, put relevant entries into these users' queues etc), in order to fit within 1 million executions.

So every incoming request should be handled in one go, by a function that accepts a request and sends push notifications to clients.

##### Execution time

Additionally, 400K GB-s execution time for 400K requests means 1GB-s for a single request, but that's the best case scenario, and we should try to further limit ourselves.
Fully handling an incoming activitypub activity will imply several requests to the database (in order to determine what clients should get push notifications about it), and that might take up to several seconds.
Adding telegram/matrix bots on top of that (if they'll run in the same acocunt) means that memory and time budget is quite limited, and that just to be on a safe side we cannot afford languages with garbage collectors or dynamically compiled languages etc; Rust will probably be the best option here.

##### Bandwidth

400K push notifications, at 4GB of inter-continental bandwidth (just to be on a safe side), is 10KB per push notification, which is somewhat reasonable for most posts (plus user information, minus userpic), hopefully.
With Telegram bots, images do not have to be sent over the wire, just an url of the image is enough.

##### Table Storage operations

We also have a 5M Table Storage operations budget, or 12 operations per incoming activity.
One to get a category of the instance.
One to retrieve a list of clients subscribed to the "Community" timeline (if needed).
One to retrieve follow relationships for the poster.
And for every relationship, one to retrieve a list of clients subscribed to the relevant timelines. Hopefully there will be at most nine relationships, on average.

We'll also have outgoing activities, but they're much less in number.

##### Table storage, blob storage

What remains is 4GB Table Storage budget and 10GB Blob Storage budget. Hopefully enough to store all the posts made by local users!
(At 10 users and 2KB storage/post that's 200K posts per user; with 500KB per post with media (but not video) that's 2K media posts per user.

We should probably give an instance administrator some toggle to allow or disallow video attachments.

#### DB structure

Only local posts are stored in DB.

Relationships are stored with data duplication:

* For every local user, we need to know who they follow (in order to allow managing follow relationships);
* Additionally, we need to know who follows them (in order to determine what instances/users to send posts created by a local user);
* And additionally, for every known remote user, we need to known what local users follow them (in order to determine what local users should get push notifications for posts created by remote users).

#### Public pages

Every fedi instance has to have some public pages (user profile, a page of the specific public post, etc).

It probably makes sense to make these static, and to store a precompiled HTML somewhere, so that no server code is executed when one of these is requested.

#### Technical tools

As mentioned above, the server will probably be written in Rust, should have an easy installer, and should have builds with support for Azure Table Storage and for at least one NoSQL database that runs on Raspberry Pi (MongoDB?)

It should support running as an Azure Function or as a cgi-bin behind some web server (nginx?).

#### Management UI, security

There should be some form of UI to handle admin actions, oauth etc.

Preferably it should be on a separate domain.

For simplicity, all authentication could be done by using HTTPS and client certificates.

A public key fingerprint of an administrator would be stored in an app config.
Using that key an administrator would be able to log into management UI and to invite a new user (by their public key).

All the everyday functionality should be available in clients (including allowing or blocking an instance on an user-level; and including notifying admins about moderation reports).

However, some less used functionality (instance-wide changing the category of a remote instance? emoji management? revoking oauth tokens? canceling subscriptions? something else?) will be managed from this UI.
