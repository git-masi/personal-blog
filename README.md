# Eric Masi - Personal Blog

This repo contains the source files for the personal blo/website of Eric Masi. It is built using the [Hextra](https://github.com/imfing/hextra) template for [Hugo](https://gohugo.io/).

## Local Development

Pre-requisites: [Hugo](https://gohugo.io/getting-started/installing/), [Go](https://golang.org/doc/install), [Git](https://git-scm.com), and [Task](https://taskfile.dev/)

### After cloning

```sh
hugo mod tidy
```

### Update theme

```sh
hugo mod get -u
hugo mod tidy
```

See [Update modules](https://gohugo.io/hugo-modules/use-modules/#update-modules) for more details.

### Start dev server

```sh
# Start the dev server
task dev
```

## Custom CSS

Custom styles can be added to `./assets/css/custom.css`.

### Syntax highlighting

This repo uses Chroma for syntax highlighting. You can generate custom themes using the following:

```sh
hugo gen chromastyles --style=github > assets/css/github.css
```

From there you can place relevant styles in `./assets/css/custom.css`. It is probably a good idea to create both dark and light styles.
