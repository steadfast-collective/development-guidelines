# Statamic

## Starter Kits

Whenever we’re starting out on a new Statamic builder, we should use the Peak Starter Kit the base.

Peak is an opinionated starter kit that comes pre-packaged with things like SEO tooling, cookie-consent notices & partials for things like responsive images and buttons.

Out of the box, Peak uses Tailwind CSS & Alpine.js - in short, this is the SALT stack!

## Code Style

When you’re developing with VS Code, you’ll probably want to install the [Antlers Toolbox](https://marketplace.visualstudio.com/items?itemName=stillat-llc.vscode-antlers) extension.

It’ll give you syntax highlighting in `.antlers.html` files, code suggestions based on the fields in blueprints.

If you’re using PhpStorm, you can use [this plugin](https://plugins.jetbrains.com/plugin/19203-antlers-language-support) instead. However, it only covers syntax highlighting.

If you’re using VS Code, there’s one setting you should change in your VS Code Settings which has to do with code formatting.

Change `HTML > Format: Wrap Attributes` to `preserve`.

This will ensure when writing Antlers the HTML attributes aren’t wrapped in a weird way which keeps the HTML clear and readable.

## Configuring Statamic’s Git Integration

Statamic’s Git Integration means that any content changes made on the live site are automatically committed & pushed to GitHub.

This means that we can see what’s changed and roll any changes back by using the power of Git!

There’s a couple of steps to setting up the Git Integration.

The first thing you’ll want to do is append `[BOT]` to the end of commits made by the Git integration.

```php
// config/statamic/git.php

'commands' => [
    'git add {{ paths }}',
    'git -c "user.name={{ name }}" -c "user.email={{ email }}" commit -m "{{ message }} [BOT]"',
],
```

Next, you’ll want to add a few deployment steps to Envoyer. One at the very start of the deployment process to cancel out the deploy if the commit was authored by the Git integration:

```bash
if [[ "{{ message }}" =~ "[BOT]" ]]; then
    echo 'AUTO-COMMITTED ON PRODUCTION. NOTHING TO DEPLOY.'
    exit 1
fi
```

And another to setup the Git repository (as sites using Envoyer don’t already include the Git repository so we need to add it manually):

```bash
set -e
git init
git branch -m main
git remote add origin git@github.com:your/remote-repository.git
git fetch
git branch --track main origin/main
git reset HEAD
```

If the git integration does not work after adding the script above, an extra step may be needed in the script.

```bash
# ...
git branch --track main origin/main
git commit --allow-empty -n -m "Initial commit." # extra step
git reset HEAD
```

The one last change you need to make is to actually enable the Git integration in the site’s `.env` file:

```bash
STATAMIC_GIT_ENABLED=true
STATAMIC_GIT_AUTOMATIC=true
STATAMIC_GIT_PUSH=true
```

You may wish to [install Laravel Horizon](https://laravel.com/docs/10.x/horizon#main-content) to ensure the Git commits & pushing is done in the background and doesn’t “slow down” the entry save request.

## Setting up on a server

We’re still waiting on Positive Internet to document the process of how we can create sites ourselves.

However, we will need to follow the steps outlined here as the configuration for Laravel Storage & forms needs to be tweaked: <https://statamic.dev/tips/zero-downtime-deployments>

## Static Caching

Static Caching essentially means that once a page has been viewed once, any subsequent visits come from the “static cache” - users are just served static HTML, PHP/Statamic isn’t touched during the request.

Ideally, every site we build should be using static caching in production. It helps improve the performance of the site for users and also reduces the load on our servers since there’s less work being done per request.

In order to use static caching, there’s a couple of things that’ll need to be done for it to work smoothly.

### Invalidation

#### When content is updated

Whenever entries or taxonomy terms are created or updated in Statamic, the URLs of that entry or term will be invalidated automatically (meaning you should see the change on the live site pretty quickly).

However, for some collections, you may wish for an update to an entry to invalidate other pages, like a blog index page.

You can configure these rules in the `statamic/static-caching.php` config file. There’s [documentation on doing so here](https://statamic.dev/static-caching#when-saving).

You may wish to [install Laravel Horizon](https://laravel.com/docs/10.x/horizon#main-content) to ensure static caching invalidation is done in the background and doesn’t “slow down” the entry save request.

#### Rebuilding the whole cache

Statamic includes a `php please static:warm` command which allows you to “rebuild” the entire static cache if needed.

Behind the scenes, Statamic is visiting the URL of every single entry in your site. Because it’s doing that so quickly one after the other, it can sometimes cause the server to struggle load-wise.

To workaround this, there’s a custom command you can copy into your project which will visit all the URLs for your entries, with a few seconds of a gap between them.

Example: <https://github.com/steadfast-collective/enclave/blob/develop/app/Console/Commands/RebuildStaticCache.php>

#### Clearing pages on demand

Sometimes, instead of rebuilding the entire static cache, you may wish to only clear specific pages.

You can install the [Static Cache Manager](https://github.com/duncanmcclean/static-cache-manager) addon to do this.

Once installed, it can be found under ‘Utilities -> Static Cache Manager’. You can enter as many URLs as you want and it’ll clear them (+ any versions of the page in the cache with query parameters).

### Server Configuration

When a site goes live, you’ll need to configure static caching in two places:

1. In your `.env`, you’ll need to enable it:

```bash
STATAMIC_STATIC_CACHING_STRATEGY=full
```

2. In the Nginx config file for the site, you’ll need to make an adjustment to the config for `location /`.

This change will point Nginx to initially look for cached files in the `public/static` folder first before resorting to load the page from Statamic.

```
location / {
  try_files /static${uri}_${args}.html $uri /index.php?$args;
}
```

## Control Panel

We spend a lot of time working on the front-end of the sites we develop. However, it’s worth spending a little bit of time, possibly near the end of a project tidying up the Control Panel experience after all, that’s what our clients will be using to edit the website we’ve built for them.

There’s a few touches, which if done before the end of the project can improve the Control Panel experience massively.

### Roles & Permissions

Obviously, whenever we’re using the Control Panel, we have access to everything: configuring collections, editing blueprints, updating Statamic, etc.

However, we probably don’t want our clients to have access to do these same things. We don’t want a client to accidentally break something by deleting a field on a blueprint or updating Statamic on production (which, if it fails, would bring down the site).

To prevent this, we can create a user role in Statamic to limit access to what the client can see.

Usually, this will be a list of:

- Entries
- Terms
- Globals
- Utilities -> Static Cache Manager
- Users
- Access form submissions

You can then assign roles to users when creating or updating users.

If the client has different types of people updating the website - for example: their marketing team, an SEO agency, operations team, you may wish to create a role for each of these different “types” of users giving them control over just what they need to do their job.

### Dashboard

By default, when you login to the Control Panel, you’ll be redirected to a ‘Dashboard’ page.

The Dashboard allows for the placement of [widgets](https://statamic.dev/widgets#content). Out of the box, Peak ships with widgets for images with missing alt information, list of pages, recent form submissions.

If it’s useful for the site, you may choose to keep the Dashboard & adjust the widgets to display information useful for content editors.

However, sometimes there might be nothing especially useful to display as widgets. In that case, you can disable the Dashboard and redirect users to another page (like the pages listing page) when users login.

```php
// config/statamic/cp.php

'start_page' => 'collection/pages',
```

### White Labelling

Another good thing to do is white labelling the CMS to make it feel a little more bespoke.

For example: changing the logo in the Control Panel to the client’s logo, changing the support URL to the Trello board & disabling links to the Statamic documentation on production/staging environments.

All of these changes can be made in the `config/statamic/cp.php` file - you may also [view Statamic’s documentation](https://statamic.dev/white-labeling#content) on white labelling the Control Panel for more information.

### Customising the Control Panel Navigation

From Statamic 3.4 onwards, you can customise the layout of the Navigation in the Control Panel.

You can customise the Nav for everyone, just for a specific user role or for just yourself.

To customise, go to the Preferences cog at the top right of the Control Panel and click ‘CP Nav’. Then choose who you’d like to customise the nav for.

You can re-arrange the nav however you’d like - rename links, put things in different sections, whatever works best for the site!

![](/api/attachments.redirect?id=f43ae738-bc59-43a5-8f1c-17ae356778a9 " =499x453")

For most sites, it might be worth moving the Control Panel pages the client will use most often to the top-level or under ‘Content’ so they don’t need to toggle Collections to see their pages/blog posts/etc.

Apart from that, the navigation items can be left as the default.

### Customising the Listing Tables

On the listing tables for collections & taxonomies, you’re able to customise which columns are displayed to the user.

For example, in addition to just showing the entry title you could show a date an entry was published, subtitles, featured images, etc.

![Example of the Listing Table of the Articles collection on Enclave](/api/attachments.redirect?id=8bfb3642-e9a9-49d1-a688-8e2176a33276 " =736x384")

You can configure the columns for yourself using the 'cogs’ button on the right-hand side of the table - toggling the columns you want to be displayed, and dragging them into the right order.

Once you’re happy with the columns, you can copy the preferences to all users by following this process:

1. In your user’s YAML file (in the `users` folder), you’ll find a preferences array containing the updated columns.

   ![](/api/attachments.redirect?id=e60b1ac2-5540-4a9b-ba49-ea2e9d935d77 " =368x240")

2. Next, create a `preferences.yaml` file inside the `resources` folder. Inside it, copy anything from inside the `preferences` array you wish to be the default for all users.

### Live Preview

Statamic’s Live Preview functionality is great for allowing clients to edit content, then immediately see the effect their changes make to the page without the need to publish changes after every edit.

When any edits are made when using Live Preview, the previewed page will reload with the changed content.

This can sometimes be annoying for things like animations & cookie banners which may popup after every change.

Thankfully, it’s easy to check in Antlers whether the page is being loaded in Live Preview or not:

```antlers
{{ if !live_preview }}
    {{ partial:cookie-notice }}
{{ /if }}
```

## Storing users in flat-files vs in the database

By default, Statamic will store all users as flat-files in the site’s `users` folder. It’s safe to include this folder in the site’s Git repository as all passwords are hashed (just like they would be in the database).

Statamic does give you the option of moving users into the database - which makes it work the same way as users do in Laravel apps.

Unless the site is going to have lots of users or the site’s primarily a web application where the Statamic portion of the site is secondary, then just use flat-file users.

It’s easy to [switch from flat-file users to database users](https://statamic.dev/tips/storing-users-in-a-database#content) if requirements change in the future.

## Resources

- [Laracasts - Learn Statamic with Jack](https://laracasts.com/series/learn-statamic-with-jack)
- [Statamic Documentation](https://statamic.dev/)
- [Scribehow: Guide for clients purchasing Statamic licenses](https://scribehow.com/shared/Purchasing_a_Statamic_License__TrjO7h8iRF6xzXeRTchhew)
