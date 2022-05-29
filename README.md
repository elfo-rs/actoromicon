# Actoromicon

[![Build Status][actions-badge]][actions-url]

[actions-badge]: https://github.com/elfo-rs/actoromicon/workflows/CI/badge.svg
[actions-url]: https://github.com/elfo-rs/actoromicon/actions?query=workflow%3ACI+branch%3Amaster

## Contributing

Install [mdBook](https://phaiax.github.io/mdBook/README.html):

```sh
cargo install mdbook
```

To build the book's HTML:

```sh
mdbook build
```

And to edit the book in pseudo-interactive "watch" mode:

```sh
mdbook watch --open # local files
mdbook serve --open # server at localhost:3000
```

### `linkcheck`

We use the `linkcheck` tool to find broken links.
To run it locally (requires the nightly toolchain):

```sh
curl -sSLo linkcheck.sh https://raw.githubusercontent.com/rust-lang/rust/master/src/tools/linkchecker/linkcheck.sh
sh linkcheck.sh book
