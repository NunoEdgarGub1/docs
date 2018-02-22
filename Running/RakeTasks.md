# List of Rake tasks
The list of Rake tasks.

If you want to confirm all Rake tasks from command line interface, you can run the following.

```sh
bundle exec rails rake -T
```

**General usage note:** Remember that each command will be invoked in a particular environment (configuration, database). The default environment when nothing is specified is "development". In production, you usually prepend all commands with `RAILS_ENV=production` to opt into the production environment. This is not necessary if you're using the Docker images, because that environment variable is set in the image for you. Similarly, prepending `bundle exec` is not necessary when using Docker. The following invocations are equivalent:

Standalone: `RAILS_ENV=production bundle exec rake etherhive:users:admins`  
Docker: `docker-compose run --rm web rake etherhive:users:admins`

Furthermore, in the command, `rake` is interchangeable with `rails`

#### Administrative

|Task|Description|Usage|
|----|-----------|-----|
|etherhive:make_admin|Turn a user into an admin|USERNAME=yourname|
|etherhive:make_mod|Turn a user into an moderator|USERNAME=yourname|
|etherhive:revoke_staff|Revoke admin or moderator privileges of a specified user|USERNAME=yourname|
|etherhive:confirm_email|Confirm a user manually|USER_EMAIL=your@email|
|etherhive:add_user|Create new user|(Interactive)|
|etherhive:users:admins|List e-mails of all admins|
|etherhive:settings:open_registrations|Open up registrations|
|etherhive:settings:close_registrations|Close down registrations|

#### Media Storage

|Task|Description|Usage|
|----|-----------|-----|
|etherhive:media:remove_silenced|Purge all media uploads by silenced accounts. Completely removes records, as if their statuses never had attachments|
|etherhive:media:remove_remote|Remove local cache of remote media attachments older than some time period (defaults to 7 days). Removes only cache, record of an attachment existing remains|NUM_DAYS=7|

#### E-mails

|Task|Description|Usage|
|----|-----------|-----|
|etherhive:emails:digest|Sends out a personal digest to all eligible inactive users. Digest includes mentions since the last time the user was active. No e-mail is sent if there is no new content since last digest or user activity|

#### Misc

|Task|Description|Usage|
|----|-----------|-----|
|etherhive:push:clear| Normally, a subscription to a once-followed user remains forever, in case the user gets re-followed later. You can purge such subscriptions with 0 followers if you wish|
|etherhive:feeds:clear_all|Purge all home timelines from redis. Useful only for troubleshooting, when e.g. resetting database during development|
|etherhive:feeds:build|Regenerates home timelines for all active users. Useful only for troubleshooting, e.g. if you lost your redis database|
|etherhive:webpush:generate_vapid_key|Generates secrets for WebPush notifications|
 
#### Data migrations

These are one-off tasks for updating from one version of EtherHive to another. They are noted in the particular release's upgrade notes and are not relevant outside of that.

|Task|Description|Usage|
|----|-----------|-----|
|etherhive:media:set_unknown|Only relevant for a past release|
|etherhive:maintenance:update_counter_caches|Only relevant for a past release|
|etherhive:maintenance:add_static_avatars|Only relevant for a past release|
|etherhive:maintenance:prepare_for_foreign_keys|Only relevant for a past release|