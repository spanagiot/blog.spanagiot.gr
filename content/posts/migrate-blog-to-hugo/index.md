---
title: "Breaking Up with Ghost: My Move to Hugo"
date: 2023-04-22T18:47:38+03:00
draft: true
---

When I started this blog, I chose Ghost as the backend because it appeared to be a fresher alternative to the bloated WordPress. I didn't need all the features offered by WordPress, and a simpler platform seemed like the obvious choice.
I installed it, created a couple of posts and I was pretty happy. I made sure to keep the backend up-to-date by installing the latest security patches and reviewing the release notes to avoid any compatibility issues with newer versions. A couple of times the updates broke the theme I used so I had to wait for the maintainer to release a fix.
However, I realized that this level of maintenance may be excessive for a blog that I update infrequently, with only a few (or hopefully in the future, a dozen) posts. Additionally, Ghost's focus on catering to creators looking to monetize their readers and the features that were released didn't align with my expectations. As a result, I am exploring different platforms to use.

This is where [Hugo](https://gohugo.io/) comes into play. [Hugo](https://gohugo.io/) is a static site generator that converts markdown files to HTML pages. It's pretty fast (written with Go) and easily extensible with themes and templates. The procedure to convert my existing blog to Hugo was easy, and the lack of content made it easier.

All you need is to install Hugo by following this [link](https://gohugo.io/installation/). If you already have Go installed you can use download Hugo by running
```
go install github.com/gohugoio/hugo@latest
```

Congratulations, you can now use Hugo to create your first website. I created this website with
```
hugo new site blog.spanagiot.gr
```

With this command Hugo created a new folder named *blog.spanagiot.gr* and populated it with the bare minimum configuration.

You will need to install a theme because Hugo doesn't include one by default. Move to the folder and run

```
git init
git submodule add https://github.com/theNewDynamic/gohugo-theme-ananke themes/ananke
echo "theme = 'ananke'" >> config.toml
```

to install the [Ananke](https://github.com/theNewDynamic/gohugo-theme-ananke) theme, the theme that is recommended by Hugo quick start guide.

Then run
```
hugo server -D
```

and you should see the local server started and by navigating to `http://localhost:1313` you are greeted by an empty website.

To create your first post, run
```
hugo new posts/my-first-post.md
```

For more information and explanations you can read the quick-start guide of Hugo [here](https://gohugo.io/getting-started/quick-start/).

So, after experimenting a bit witg Hugo and see how it works and how it is configured, it was time to export my existing Ghost blog and import it to Hugo.
For this, I will use the [ghostToHugo](https://github.com/jbarone/ghostToHugo)
I installed it with
```
go install github.com/jbarone/ghostToHugo@latest
```

I needed to export the existing data by going to https://blog.spanagiot.gr/ghost/settings/labs and press the *Export your content* button. This will download a JSON file with all the blog content that I will feed to **ghostToHugo**.

**ghostToHugo** wants to generate a new, empty site to load the data so I navigated to `/tmp` and created the site there with
```
ghostToHugo  -v ~/Downloads/blog-spanagiot-gr.ghost.2023-04-08-09-35-29.json
```

A new folder named **newhugosite** has been created. The **content/posts** folder contains all the posts that were exported from the Ghost blog.

Migrating from Ghost to Hugo is not a painless procedure. Images were not automatically exported and I had to manually grab them and include them to the markdown files.

In Hugo, there are 2 ways of organizing your posts folder. You can either create single Markdown files inside the posts folder or you can have a *page bundle* where each folder has the post name, an *index.md* exists inside the post folder and all the assets of it reside inside.
I decided to follow the *page bundle* solution so I can keep the assets and images of all posts organized in their own folder.

I created the bundle with the existing posts using the following bash script inside the **content/posts** folder.
```bash
#!/bin/bash

# loop through each file in the directory
for file in ./*; do
  # get the file name without the extension
  filename=$(basename -- "$file")
  name="${filename%.*}"

  # create a folder with the same name as the file
  mkdir "./$name"

  # move the file inside the newly created folder and rename it to index.md
  mv "$file" "./$name/index.md"
done
```

After this, I had to decide on the theme that I want for my updated blog. I navigated to https://themes.gohugo.io/ and decided to go with [PaperMode](https://adityatelange.github.io/hugo-PaperMod/).
You can install [PaperMode](https://adityatelange.github.io/hugo-PaperMod/) with

```
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

Then, configure Hugo to use it by appending it to the configuration file 
```
echo "theme = 'PaperMod'" >> config.toml
```

> You can configure the *PaperMod* theme.
> All the options are presented [here](https://adityatelange.github.io/hugo-PaperMod/posts/papermod/papermod-features/)

The next step was to move all the exported posts (*page bundles*) to my new blog folder and see how they look.

The only major issue that I encountered was the way each platform handled images. To include an image in a Hugo page you need to use the following snippet
```
{{</* figure align=center src="images/image-7.png" title="The MAC address from our application" */>}}
```

This will load the **image-7.png** located in the **image** folder inside the current *page bundle*, align it to the center of the page and present a nice caption below the image. In this specific case, the folder stucture looks like this
```
content/posts
├── get-mac-address-for-ios
│   ├── images
│   │   ├── image-4.png
│   │   ├── image-6.png
│   │   └── image-7.png
│   └── index.md
```

I started the server and navigated around the website to check that the content was showing properly. After confirming that it didn't have any issue, it was time to publish my site.

I edited the `baseURL` value inside the `config.toml` file to match the domain I am using and also changed the `title` to the new title of this blog, **DevOps Diaries: A Personal Blog**

Finally, all I needed to do was to run
```
hugo
```

and all the entire static site was generated inside the **public** folder, including HTML and all the assets.