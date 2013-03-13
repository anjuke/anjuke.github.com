# Anjuke Engineering Blog

http://arch.corp.anjuke.com

----

## How to contribute

You should have **ruby** and **bundler** installed in order to generate the static pages.

I'm using **ruby-2.0.0p0**.

Please refer the [official page](http://gembundler.com/) for installing bundler.

### Preparing

Clone and checkout `source` branch.

```
$ git clone git@github.com:anjuke/anjuke.github.com.git
$ cd anjuke.github.com
$ git checkout -B source origin/source
```

Now bootstrap and install dependencies.

```
$ script/boostrap
```

### Generate pages

Save you new post to `source/_posts` directory. You may also attach some small media files to `medias` directory.

When you're done type the following in the terminal.

```
$ rake generate
```

### Preview

Before deploy you may preview the site in a web browser locally.

```
$ rake preview
```

Open [http://localhost:4000](http://localhost:4000) in your browser.

### Deploy

Everything looks OK? Cool, you are able to live the site by the following

```
$ rake deploy
```

Open [http://arch.corp.anjuke.com](http://arch.corp.anjuke.com) to check online.

Remember to push your `source` branch to origin repo otherwise other people may overwrite you works.

### Pull-request

If you don't have **push** permission of this repo, you are unable to deploy the site directly. But you can still use [pull-request](https://help.github.com/articles/using-pull-requests) to contribute.

[pull-request is preferred](http://codeinthehole.com/writing/pull-requests-and-other-good-practices-for-teams-using-github/).

