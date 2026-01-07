# Actoromicon

[![CI][actions-badge]][actions-url]

[actions-badge]: https://github.com/elfo-rs/actoromicon/actions/workflows/ci.yml/badge.svg
[actions-url]: https://github.com/elfo-rs/actoromicon/actions/workflows/ci.yml

## Contributing

Install [mdbook](https://phaiax.github.io/mdBook) and [rumdl](https://github.com/rvben/rumdl):

```sh
cargo install mdbook rumdl
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

To lint (check rumdl docs to integrate with your editor):

```sh
rumdl check
```
