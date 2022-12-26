# Blog

The source for my blog, and a chance to learn Hugo.

My blog is at https://d1zozjsyfqzng.cloudfront.net/.

## Hugo

Start Hugo's development server, including draft content. The site will be running on `http://localhost:1313/`.
```
hugo server -D
```

Add content:
```
hugo new posts/my-new-post.md
```

Build the site:
```
hugo
```
This will publish the site to the `public` directory.

Deploy:
```
hugo deploy
```
