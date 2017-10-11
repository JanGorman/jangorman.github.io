## Octopress CLI Commands

Here are the subcommands for Octopress.

```
init <PATH>         # Adds Octopress scaffolding to your site
new <PATH>          # Like `jekyll new` + `octopress init`
new post <TITLE>    # Add a new post to your site
new page <PATH>     # Add a new page to your site
new draft <TITLE>   # Add a new draft post to your site
publish <POST>      # Publish a draft from _drafts to _posts
unpublish <POST>    # Search for a post and convert it into a draft
isolate [POST]      # Stash all posts but the one you're working on for a faster build
integrate           # Restores all posts, doing the opposite of the isolate command
deploy              # deploy your site via S3, Rsync, or to GitHub pages.
```

### New Post

This automates the creation of a new post.

```sh
$ octopress new post "My Title"
```

#### Command options

| Option               | Description                             |
|:---------------------|:----------------------------------------|
| `--template PATH`    | Use a template from <path>              |
| `--date DATE`        | The date for the post. Should be parseable by [Time#parse](http://ruby-doc.org/stdlib-2.1.0/libdoc/time/rdoc/Time.html#method-i-parse) |
| `--slug SLUG`        | Slug for the new post.                  |
| `--dir DIR`          | Create post at _posts/DIR/.             |
| `--force`            | Overwrite existing file.                |

### New Page

Creating a new page is easy, you can use the default file name extension (.html), pass a specific extension, or end with a `/` to create
an index.html document.

```
$ octopress new page some-page           # ./some-page.html
$ octopress new page about.md            # ./about.md
$ octopress new page docs/               # ./docs/index.html
```

### New Draft

This will create a new post in your `_drafts` directory.

```sh
$ octopress new draft "My Title"
```

| Option             | Description                               |
|:-------------------|:------------------------------------------|
| `--template PATH`  | Use a template from <path>                |
| `--date DATE`      | The date for the draft. Should be parseable by [Time#parse](http://ruby-doc.org/stdlib-2.1.0/libdoc/time/rdoc/Time.html#method-i-parse) (defaults to Time.now) |
| `--slug SLUG`      | The slug for the new post.                |
| `--force`          | Overwrite existing file.                  |

### Publish a draft

Use the `publish` command to publish a draft to the `_posts` folder. This will also rename the file with the proper date format.

```sh
$ octopress publish _drafts/some-cool-post.md
$ octopress publish cool
```
In the first example, a draft is published using the path. The publish command can also search for a post by filename. The second command
would work the same as the first. If other drafts match your search, you will be prompted to select them from a menu. This is often much
faster than typing out the full path.

| Option             | Description                               |
|:-------------------|:------------------------------------------|
| `--date DATE`      | The date for the post. Should be parseable by [Time#parse](http://ruby-doc.org/stdlib-2.1.0/libdoc/time/rdoc/Time.html#method-i-parse) |
| `--slug SLUG`      | Change the slug for the new post.         |
| `--dir DIR`        | Create post at _posts/DIR/.               |
| `--force`          | Overwrite existing file.                  |

When publishing a draft, the new post will use the draft's date. Pass the option `--date now` to the publish command to set the new post date from your system clock. As usual, you can pass any compatible date string as well.

### Unpublish a post

Use the `unpublish` command to move a post to the `_drafts` directory, renaming the file according to the drafts convention.

```sh
$ octopress unpublish _posts/2015-01-10-some-post.md
$ octopress unpublish some post
```

Just like the publish command, you can either pass a path or a search string to match the file name. If more than one match is found, you
will be prompted to select from a menu of posts.

### Deploying your site

The Octopress gem comes with [octopress-deploy](https://github.com/octopress/deploy) which allows you to easily deploy your site with Rsync, on S3 or Cloudfront, to GitHub pages, or other Git based deployment hosting platforms.

Once you've built your site (with `jekyll build`) you can deploy it like this:

```
$ octopress deploy
```

#### Generate Deployment configuration

**Remember to add your configuration to `.gitignore` to be sure
you never commit sensitive information to your repository.**


Octopress can generate a deployment configuration file for you using the `octopress deploy init` command.

```
$ octopress deploy init git git@github.com:user/project
```

