+++
date = '2025-09-28T12:59:28-04:00'
draft = false
title = 'Building my Blog'
+++

In this post, I'm going to talk in more depth about why and how I built this blog.

<!--more-->

# How it Started

Many months ago, I started watching a YouTuber named **ThePrimeAgen**. Every now and then, he talks about Golang and expresses his enthusiasm for the language. Intrigued by his insights, I decided to give Golang a try to see what it was all about.

I began experimenting with Golang about six months ago, solving some [BeeCrowd](https://judge.beecrowd.com/) problems just to get the hang of it. Soon after, I built a few simple REST API servers, as Golang is very well-suited for this kind of software.

But why create a blog?

I'm a big fan of tech blogs, and since most of my knowledge has come from reading them, I thought it would be cool to have my own. As I mentioned in my [previous post](https://blog.luizgustavojunqueira.com/post/my-blog#why-i-made-this-blog), I also wanted to test my skills, deploy an application beyond localhost, and improve my writing.

## Technologies

For building this blog, I decided to use only technologies I had never used before, so I chose the GOTTH stack:

- **Golang**: to serve the web app
- **Templ**: for HTML templating
- **TailwindCSS**: for styling and beautiful design
- **HTMX**: for better interactivity and navigation
- **AlpineJS**: for interactive animations

I also used PostgreSQL as the database, though I plan to switch to SQLite in the future.

## How it works

### The Backend

I developed the backend as a package so that it could be reused by anyone who wants a simple blog without much hassle. Currently, the only customization options available are for the blog title, page title, and admin user.

I plan to add more customization options — such as color schemes and layout adjustments — in the future once I address other priorities.

The package is named Blogo (a blend of "Blog" and "Go") and requires the following configuration:

```go
type BlogoConfig struct {
	BlogName   string
	Title      string
	Port       string
	DB         *sql.DB
	AuthConfig *auth.AuthConfig
	Logger     *log.Logger
	Location   *time.Location
	Queries    *repository.Queries
}
```

You can run the server with:

```go
blog.Start()
```

A more detailed example is available in my [repo](https://github.com/luizgustavojunqueira/Blogo/blob/main/cmd/blog/main.go), where you can see my personal blog configuration.

#### PostHandler

The server consists of a PostHandler and an AuthHandler. The authentication part is straightforward, so I'll focus on the PostHandler, which comprises seven functions to manage posts:

- **GetPosts**: Renders a list of post cards as HTML using a Templ template.
- **ViewPost**: Renders the full post.
- **DeletePost**: Deletes the specified post.
- **Editor**: Renders a simple editor where I can input the post title, slug, description, and Markdown content (it's pretty basic at the moment, so no images).
- **Edit**: Edits an existing post.
  **CreatePost**: Generates the table of contents, converts the Markdown content into HTML, and saves the post.
- **ParseMarkdown**: Similar to CreatePost but returns the HTML content instead—this is used for live reloading in the editor.

### The Frontend

As mentioned earlier, the frontend is essentially pure HTML with a small amount of JavaScript provided by **HTMX** and **AlpineJS**, and styled with **TailwindCSS**.

#### Why Templ?

Templ allows for creating reusable HTML components that integrate seamlessly with Golang code. For example, consider the MainPage component that lists the post cards:

```go
templ MainPage(blogname string, posts []repository.Post, authenticated bool) {
	if authenticated {
		@components.Header(blogname, []string{"New Post", "Logout"}, []string{"/editor", "/logout"})
	} else {
		@components.Header(blogname, []string{"Login"}, []string{"/login"})
	}
	<ul id="posts-list" class="min-h-screen flex flex-col items-center">
		for _, post := range posts {
			@components.PostCard(post, authenticated)
		}
	</ul>
}
```

> Note: The Header and PostCard components are defined in the components package.

The Golang code for rendering the page is straightforward:

```go
mainPage := pages.MainPage(h.blogName, h.pagetitle, posts, authenticated)
mainPage.Render(ctx, w) // w is the http.ResponseWriter
```

#### HTMX and AlpineJS

I used a small amount of **HTMX** primarily for making HTTP requests—such as deleting a post, handling login, and enabling live reload in the editor.

**AlpineJS** is employed mainly to toggle the visibility of the table of contents (TOC) and to switch the color scheme.

### Database

Since this is a relatively simple app, I chose not to use an ORM and instead went with SQLC—a tool that compiles SQL queries into Go code, ensuring type safety.

For database migrations, I'm using golang-migrate.

# Conclusion

As you can see, there is still some work to be done (which I plan to address someday, hopefully), but for now the blog meets my needs.

Regarding the technologies, I really enjoyed working with them. It was a great experience developing this app using the **GOTTH** stack, as all the tools have supportive communities and excellent documentation.

I liked it so much that I'm planning to build a simple web app for organizing personal finances (e.g., tracking monthly expenses), since I haven't found a good free app available. Perhaps someday I'll write a post about that as well.

Until then, thanks for taking the time to read this.

**_Check out my [Portfólio](https://portfolio.luizgustavojunqueira.com) too. There's a post coming about it soon too._**
