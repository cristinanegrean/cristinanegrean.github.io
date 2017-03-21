# cristinanegrean.github.io
Personal Blog cristina.tech

## Setting up Jekyll on your computer
- Execute following commands:
    - `(sudo) gem install rake`
    - `(sudo) gem install rake-jekyll`
    - `(sudo) gem install jekyll`
    - `(sudo) gem install jekyll-paginate`
- Run `jekyll serve` in the root directory of the tech blog
- Browse to http://localhost:4000/

## Add yourself as an author
- Add yourself to the **_data/authors.yml** file
- Add a picture of yourself to the **img** folder

## Create a new blogpost
- Add a blogpost image to the **img** directory
- Use any (online) **Markdown editor** (for example [brackets](http://brackets.io) or [classeur.io](http://classeur.io))
- Add following [**FrontMatter**](http://jekyllrb.com/docs/frontmatter/) tags on the top of your post (you can also copy-paste this from another post)
    - layout: post
    - author: {author}
    - title: {title}
    - image: {image_path}
    - tags: {tags}
    - category: {category}
    - comments: true
- Save file in the **_post** directory using the following file format: *{year}-{month}-{day}-{title}.md*
	- **example**: *2015-11-09-Awesome-Blogpost.md*
- Publish as **plain text**! (Jekyll will generate the static HTML for us)
- Tweak post and commit until you feel satisfied with it
