---
title: Efficiently resolving composer.lock merge conflicts
published: true
description: When working in a team on a PHP project using Composer, you have probably encountered a problem when multiple people added, removed or…
slug: efficiently-resolving-composer-lock-merge-conflicts
canonical_url: https://ionbazan.medium.com/efficiently-resolving-composer-lock-merge-conflicts-7ebc260c0337
tags: php, composer, git, conflict
author: Ion Bazan
cover: https://images.unsplash.com/photo-1545296664-39db56ad95bd
---

# Efficiently resolving composer.lock git merge conflicts

When working in a team on a PHP project using Composer, you have probably encountered a problem when multiple people added, removed or updated some packages in `composer.json` and `composer.lock` on `main` branch and GIT welcomed you with this message in the morning:

```
Auto-merging composer.lock
CONFLICT (content): Merge conflict in composer.lock
```

Resolving such conflict is not a straightforward operation due to the structure of the file which contains all locked dependencies information and a content hash.

In this article I will show you few methods to resolve such conflict and make your life a bit easier.

# TL;DR

```
git checkout --theirs composer.lock
composer update php # the "php" is important here
```

# What not to do?

I’ve seen people doing `git checkout --ours composer.lock` and pushing the changes… Since it’s overwriting all others’ changes with yours, it’s basically an equivalent of force-pushing changes to `main` branch — never do that!

Another bad (but not as destructive) idea would be to remove `composer.lock` file and recreate it using `composer update`. This might seem like a good idea if you don’t really care about your locked versions and trust your version constraints and tests 100%. In real life, this will most likely introduce some unexpected changes into your feature branch and unnecessarily increase the diff.

# Method 1: The fast & simple [my favorite!]

This method is suitable for most common scenarios, when some packages were added or updated in `main` branch and you are trying to merge it into your feature branch with some other packages added. Assuming there are no dependency conflicts, you are good to go with this solution after merging the `composer.json` file.

First, accept the changes from base (`main`) branch:

```
git checkout --theirs composer.lock
```

Then, trigger an update of the packages you added or removed:

```
composer update php
```

Please note that you may pass any installed (or even not installed) package name to the update command for it to work but passing `php` will only touch the packages you just added or removed, without unnecessarily updating any extra packages to keep your changes clean.

<img width="100%" style="width:100%" src="https://media.giphy.com/media/26FPnsRww5DbqoPuM/giphy.gif">

### Pros:

* Fast
* Simple
* Universal — you can use same commands, no matter what packages you add or remove

### Cons:

* Packages you added on your feature branch will be updated to most recent version matching the constraints (usually it should not be an issue)
* Any package updates (via `composer update`) from your feature branch will be gone — you should re-run the update afterwards if needed

# Method 2: Redoing your changes

This method is described in the [official Composer documentation](https://getcomposer.org/doc/articles/resolving-merge-conflicts.md). In some cases, when there might be some conflicting packages, it might be better to redo you package additions/removals on top of most recent changes from `main` branch — similarly as applying changes during a rebase.

First, accept the changes from base (`main`) branch:

```
git checkout --theirs composer.lock composer.json
```

Then, add all the packages you added or removed in your feature branch:

```
composer require package/a:^1.2
composer require package/b:^2.0
composer remove package/c
...
```

When you’re done, make sure your `composer.json` and `composer.lock` files are valid using `composer validate` command.

### Pros:

* Reports conflicting dependencies as you add each of your packages
* Quite fast

### Cons:

* You need to remember which packages you added or removed in your feature branch
* Repeating these actions on each merge might be tedious and error-prone
* Packages you added on your feature branch will be updated to most recent version matching the constraints (usually it should not be an issue)
* Any package updates (via `composer update`) from your feature branch will be gone — you should re-run the update afterwards if needed

# Method 3: Using external tool

Since above methods will not preserve the locked versions on packages that have been added on your feature branch, you may use an external tool such as [Composer Git Merge Driver](https://github.com/balbuf/composer-git-merge-driver) that automates this process, keeping your locked versions and minimizing the diff produced. Please refer to the [official repository](https://github.com/balbuf/composer-git-merge-driver) for installation and usage instructions.

<img width="100%" style="width:100%" src="https://media.giphy.com/media/NmerZ36iBkmKk/giphy.gif">

### Pros:

* Works magically — in most cases you will not even notice the conflict

### Cons:

* Installation might be a bit too complicated to enforce it in the team

# Additional notes

Please keep in mind that using Method 1 or 2 will discard the information about the exact package version added in your feature branch. This is because `composer` will lock the newest available version matching your constraints at the time of merge. If your package was constrained as `^1.0` and locked at `1.1.0` , it might be updated to `1.2.0` while performing these steps. Most of the time it should not produce any side effects, especially if your version constraints are valid.
