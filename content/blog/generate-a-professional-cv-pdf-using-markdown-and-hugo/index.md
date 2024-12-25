+++
title = "Generate a Professional CV PDF Using Markdown and Hugo"
date = "2024-12-25T20:25:56+04:00"
cover = "/blog/generate-a-professional-cv-pdf-using-markdown-and-hugo/images/cover.png"
images = [
    "/blog/generate-a-professional-cv-pdf-using-markdown-and-hugo/images/cover.png",
    "/blog/generate-a-professional-cv-pdf-using-markdown-and-hugo/images/cv-in-website-layout.png",
    "/blog/generate-a-professional-cv-pdf-using-markdown-and-hugo/images/cv-in-own-layout.png",
    "/blog/generate-a-professional-cv-pdf-using-markdown-and-hugo/images/cv-pdf.png"
]
tags = ["Hugo", "Markdown", "CV", "PDF"]
keywords = ["Hugo CV PDF", "Markdown to PDF Hugo", "Generate CV with Hugo", "Convert Hugo content to PDF", "Build CV with Hugo and Markdown"]
description = "Learn how to generate a professional CV in PDF format using Markdown and Hugo. This guide walks you through creating a streamlined workflow to manage your content effortlessly, keeping your CV and website consistent and up-to-date."
showFullContent = false
readingTime = true
+++

Recently, I updated my personal website and rebuilt it using [Hugo](https://gohugo.io/). For those unfamiliar:

> Hugo is one of the most popular open-source static site generators. With its amazing speed and flexibility, Hugo makes building websites fun again.

My website includes various sections such as About, Work Experience, Projects, and more. However, this blog focuses specifically on the CV section — a PDF file containing a summary of my work experience, education, skills, projects, and more. Essentially, the CV is duplicated content from my website.

Maintaining two separate sources for my website and the CV seemed inefficient, so I considered generating the CV directly from my Hugo content. After some research, I discovered that Hugo does not natively support generating PDF files. But don’t worry — this story has a happy ending! I found a workaround to solve my problem.

<!--more-->

**Disclaimer**: This blog assumes you are familiar with the basic concepts of building a website with Hugo. I won’t dive into those details here. Instead, the focus is on converting a page into a PDF file.

## Planning the Structure

It all started with the planning phase. I decided that my CV would include the following sections: Summary, Skills, Experience, Education, and Open Source (to showcase some of my open-source projects).

To avoid duplication, I reused Hugo shortcodes I had previously created for sections like Work Experience, Education, and Skills. Here’s an example of the shortcode for displaying Working Experience:

```html
<div class="experience">
  {{ range .Site.Data.experience }}
  <div class="container">
    <span class="date">{{ .startDate }} - {{ .endDate }}</span>
    <div class="content">
      <div class="title">{{ .position }}</div>
      <div class="subtitle">
        <span>{{ .company }}</span> <span>{{ .location }}</span>
      </div>
    </div>
  </div>
  {{ end }}
</div>
```

With these shortcodes in place, the final outline for my `content/cv.md` file looked like this:

```md
## Summary

Some text about me...

## Skills

{{</* skills */>}}

## Experience

{{</* experience */>}}

## Education

{{</* education */>}}

## Open Source

{{</* oss-projects */>}}
```

With all the steps mentioned above, I now have a functional `/cv` page on my website that displays all the necessary information for my CV:

{{< figure src="/blog/generate-a-professional-cv-pdf-using-markdown-and-hugo/images/cv-in-website-layout.png" alt="CV in Website Layout" position="center" caption="CV in Website Layout" captionPosition="center" >}}

## CV Layout

The `/cv` page is currently a regular website page with the site’s header, footer, and overall styling. While my website is minimalistic thanks to the [Terminal](https://themes.gohugo.io/themes/hugo-theme-terminal/) theme — and this layout could work for the CV — your website might have a more complex design. A CV should be clean, minimalistic, and highly readable.

To achieve this, I created a separate layout specifically for the CV, stripping unnecessary elements like the header and footer and applying minimal styling. Here’s what my `layouts/_default/cv.html` file looks like:

```html
<!DOCTYPE html>
<html lang="{{ $.Site.Language }}">
  <head>
    <style>
      // Some styles.
    </style>
  </head>
  <body>
    <header>
      <h1>{{ .Site.Data.profile.cv.name }}</h1>
      <div class="contact">
        <div class="contact-info">
          <span>{{ .Site.Data.profile.cv.email }}</span>
          <span>{{ .Site.Data.profile.cv.phone }}</span>
          <span>{{ .Site.Data.profile.cv.address }}</span>
        </div>
        <div class="links">
          {{ range .Site.Data.profile.social }}
          <a href="{{ .url }}">{{ .label }}</a>
          {{ end }}
        </div>
      </div>
    </header>
    <main>{{ .Content }}</main>
  </body>
</html>
```

This gave me a clean and professional - looking CV page:

{{< figure src="/blog/generate-a-professional-cv-pdf-using-markdown-and-hugo/images/cv-in-own-layout.png" alt="CV in own Layout" position="center" caption="CV in own Layout" captionPosition="center" >}}

I can even directly share the link to my CV, such as <a href="https://abgeo.dev/cv" target="_blank">abgeo.dev/cv</a>, with recipients. However, an HTML page isn’t the most practical format for sharing a CV, especially in professional settings.


We still need to convert it to a more universally accepted format, like PDF. Stay tuned — we’re almost there!

## Converting CV to PDF

As I mentioned earlier, Hugo doesn’t natively support generating PDF files. While manually printing the `/cv` page to a PDF is an option, I wanted an automated solution. Ideally, I wanted to integrate this step into my Continuous Deployment (CD) pipeline to ensure any updates to my content are automatically reflected in the CV PDF.

That’s when I discovered [WeasyPrint](https://weasyprint.org/):

> WeasyPrint is a smart solution helping web developers to create PDF documents. It’s free and open source software that can be easily plugged to your applications and websites and turns simple HTML pages into gorgeous PDFs.

**WeasyPrint** comes with a CLI interface that can convert static HTML pages into PDF files. With a single command, I was able to generate a PDF representation of my CV page:

```bash
weasyprint public/cv/index.html cv.pdf
```

{{< figure src="/blog/generate-a-professional-cv-pdf-using-markdown-and-hugo/images/cv-pdf.png" alt="CV in PDF Format" position="center" caption="CV in PDF Format" captionPosition="center" >}}

## Conclusion

Bingo! This is exactly what I wanted to achieve: a clean and readable CV in PDF format, built entirely using Markdown and Hugo’s features like shortcodes.

This approach provides flexibility by allowing me to manage my content from a single source, ensuring consistency between my website and CV without duplication. Integrating the PDF generation into my workflow guarantees that any changes are automatically reflected in the final document.

Whether you’re managing a portfolio, resume, or any other content shared across multiple formats, this method offers an efficient and powerful solution. I hope this inspires you to explore similar workflows and streamline your content management. Happy building!
