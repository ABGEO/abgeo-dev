# abgeo-dev

My personal website built with [Hugo](https://gohugo.io)

## Features

- Static site generation using Hugo
- Organized content structure for easy updates
- Optimized for performance and SEO
- Lightweight, fast, and responsive design using [Terminal](https://themes.gohugo.io/themes/hugo-theme-terminal/) theme (modified)

## Running the Website Locally

### Prerequisites

- [Hugo](https://gohugo.io/getting-started/installing/) installed
- A compatible browser to view the site locally


To start a local development server, use:
```bash
hugo server -D
```

Access the website at http://localhost:1313

## Building the Website for Production

To build the website, run the following command:
```bash
hugo
```

This will generate the static files in the public directory.

## Generate CV

### Prerequisites

- [weasyprint](https://doc.courtbouillon.org/weasyprint/stable/first_steps.html) installed

```bash
hugo
weasyprint public/cv/index.html cv.pdf
```

This will generate the CV PDF file in the current directory.

## License

Copyright (c) 2024 [Temuri Takalandze](https://www.abgeo.dev).  
Released under the [MIT](LICENSE) license.
