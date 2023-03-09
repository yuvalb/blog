# My Blog

This blog is based on [Cobalt](https://cobalt-org.github.io/) which is written in [Rust](https://www.rust-lang.org/).

## Development

Serving blog locally:

```sh
cobalt serve
```

Or with drafts:

```sh
cobalt serve --drafts
```

## Editing

### Add a page

Add a new page or post to your site:

```sh
$ cobalt new "Cats Around the World"
```

### Publish a post

Once your post is ready, you can publish it:

```sh
cobalt publish posts/cats-around-the-world.md
```

The page will no longer be a "draft" and the published_date will be set to today.

## Publishing

### Build the site

Once the post is in published state, build the site:

```sh
cobalt build
```

The site is sitting in \_site and ready to be uploaded!
