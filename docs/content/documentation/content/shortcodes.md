+++
title = "Shortcodes"
weight = 40
+++

Zola borrows the concept of [shortcodes](https://codex.wordpress.org/Shortcode_API) from WordPress.
In our case, a shortcode corresponds to a template defined in the `templates/shortcodes` directory that can be used in a Markdown file.

Broadly speaking, Zola's shortcodes cover two distinct use cases:

* Inject more complex HTML: Markdown is good for writing, but it isn't great when you need to add inline HTML or styling.
* Ease repetitive data based tasks: when you have [external data](@/documentation/templates/overview.md#load-data) that you
  want to display in your page's body.

The latter may also be solved by writing HTML, however Zola allows the use of Markdown based shortcodes which end in `.md`
rather than `.html`. This may be particularly useful if you want to include headings generated by the shortcode in the
[table of contents](@/documentation/content/table-of-contents.md).

If you want to use something similar to shortcodes in your templates, you can use [Tera macros](https://keats.github.io/tera/docs#macros). They are functions or components that you can call to return some text.

## Writing a shortcode
Let's write a shortcode to embed YouTube videos as an example.
In a file called `youtube.html` in the `templates/shortcodes` directory, paste the
following:

```jinja2
<div {% if class %}class="{{class}}"{% endif %}>
    <iframe
        src="https://www.youtube.com/embed/{{id}}{% if autoplay %}?autoplay=1{% endif %}"
        webkitallowfullscreen
        mozallowfullscreen
        allowfullscreen>
    </iframe>
</div>
```

This template is very straightforward: an iframe pointing to the YouTube embed URL wrapped in a `<div>`.
In terms of input, this shortcode expects at least one variable: `id` ([example here](#shortcodes-without-body)).
Because the other variables are in an `if` statement, they are optional.

That's it. Zola will now recognise this template as a shortcode named `youtube` (the filename minus the `.html` extension).

The Markdown renderer will wrap an inline HTML node such as `<a>` or `<span>` into a paragraph.
If you want to disable this behaviour, wrap your shortcode in a `<div>`.

A Markdown based shortcode in turn will be treated as if what it returned was part of the page's body. If we create
`books.md` in `templates/shortcodes` for example:

```jinja2
{% set data = load_data(path=path) -%}
{% for book in data.books %}
### {{ book.title }}

{{ book.description | safe }}
{% endfor %}
```

This will create a shortcode `books` with the argument `path` pointing to a `.toml` file where it loads lists of books with
titles and descriptions. They will flow with the rest of the document in which `books` is called.

Shortcodes are rendered before the page's Markdown is parsed so they don't have access to the page's table of contents.
Because of that, you also cannot use the [`get_page`](@/documentation/templates/overview.md#get-page) / [`get_section`](@/documentation/templates/overview.md#get-section) / [`get_taxonomy`](@/documentation/templates/overview.md#get-taxonomy) / [`get_taxonomy_term`](@/documentation/templates/overview.md#get-taxonomy-term) global functions. It might work while
running `zola serve` because it has been loaded but it will fail during `zola build`.

## Using shortcodes

There are two kinds of shortcodes:

- ones that do not take a body, such as the YouTube example above
- ones that do, such as one that styles a quote

In both cases, the arguments must be named and they will all be passed to the template. 
Parentheses are mandatory even if there are no arguments.

Note that while shortcodes look like normal Tera expressions, they are not Tera at all -- they can
pretty much just shuttle arguments to their template. Several limitations of note are:

- All arguments are required
- The shortcode cannot reference Tera variables
- Concatenation and other operators are unavailable

If the shortcode is invalid, it will not be interpreted by the markdown parser and will instead
get rendered directly into the final HTML.

Lastly, a shortcode name (and thus the corresponding `.html` file) as well as any argument names
can only contain numbers, letters and underscores, and must start with a letter or underscore.
In Regex terms, `^[A-Za-z_][0-9A-Za-z_]+$`.

Argument values can be of one of five types:

- string: surrounded by double quotes, single quotes or backticks
- bool: `true` or `false`
- float: a number with a decimal point (e.g., 1.2)
- integer: a whole number or its negative counterpart (e.g., 3)
- array: an array of any kind of value, except arrays

Malformed values will be silently ignored.

Both types of shortcode will also get either a `page` or `section` variable depending on where they were used
and a `config` variable. These values will overwrite any arguments passed to a shortcode so these variable names
should not be used as argument names in shortcodes.

### Shortcodes without body

Simply call the shortcode as if it was a Tera function in a variable block.

```md
Here is a YouTube video:

{{/* youtube(id="dQw4w9WgXcQ") */}}

{{/* youtube(id="dQw4w9WgXcQ", autoplay=true) */}}

An inline {{/* youtube(id="dQw4w9WgXcQ", autoplay=true, class="youtube") */}} shortcode
```

Note that if you want to have some content that looks like a shortcode but not have Zola try to render it,
you will need to escape it by using `{{/*` and `*/}}` instead of `{{` and `}}`.

### Shortcodes with body
Let's imagine that we have the following shortcode `quote.html` template:

```jinja2
<blockquote>
    {{ body }} <br>
    -- {{ author}}
</blockquote>
```

We could use it in our Markdown file like so:

```md
As someone said:

{%/* quote(author="Vincent") */%}
A quote
{%/* end */%}
```

The body of the shortcode will be automatically passed down to the rendering context as the `body` variable and needs
to be on a new line.

### Shortcodes with no arguments
Note that for both cases that the parentheses for shortcodes are necessary. 
A shortcode without the parentheses will render as plaintext and no warning will be emitted.

As an example, this is how an `aside` shortcode-with-body with no arguments would be defined in `aside.html`:
```jinja2
<aside>
    {{ body }}
</aside>
```

We could use it in our Markdown file like so:

```md
Readers can refer to the aside for more information.

{%/* aside() */%}
An aside
{%/* end */%}
```

### Content similar to shortcodes

If you want to have some content that looks like a shortcode but not have Zola try to render it,
you will need to escape it by using `{%/*` and `*/%}` instead of `{%` and `%}`. You won't need to escape
anything else until the closing tag.

## Shortcode context

Every shortcode can access some variables, beyond what you explicitly passed as parameter. These variables are explained in the following subsections:

- invocation count (`nth`)
- current language (`lang`), unless called from the `markdown` template filter (in which case it will always be the same value as `default_language` in configuration, or `en` when it is unset)
- `colocated_path`

When one of these variables conflict with a variable passed as argument, the argument value will be used.

### `nth`: invocation count

Every shortcode context is passed in a variable named `nth` that tracks how many times a particular shortcode has
been invoked in the current Markdown file. Given a shortcode `true_statement.html` template:

```jinja2
<p id="number{{ nth }}">{{ value }} is equal to {{ nth }}.</p>
```

It could be used in our Markdown as follows:

```md
{{/* true_statement(value=1) */}}
{{/* true_statement(value=2) */}}
```

This is useful when implementing custom markup for features such as sidenotes or end notes.

### `lang`: current language
**NOTE:** When calling a shortcode from within the `markdown` template filter, the `lang` variable will always be `en`. 
If you feel like you need that, please consider using template macros instead. 
If you really need that, you can rewrite your Markdown content to pass `lang` as argument to the shortcode.

Every shortcode can access the current language in the `lang` variable in the context. 
This is useful for presenting/filtering information in a shortcode depending in a per-language manner. For example, to display a per-language book cover for the current page in a shortcode called `bookcover.md`:

```jinja2
![Book cover in {{ lang }}](cover.{{ lang }}.png)
```

### `page` or `section`
You can access a slightly stripped down version of the equivalent variables in the normal templates.
The following attributes will be empty:

- translations
- backlinks
- pages

(Note: this is because the rendering of markdown is done before populating the sections)

A useful attribute to `page` in shortcodes is `colocated_path`.
This is used when you want to pass the name of some assets to shortcodes without repeating the full folders path.
Mostly useful when combined with `load_data` or `resize_image`.

```jinja2
{% set resized = resize_image(format="jpg", path=page.colocated_path ~ img_name, width=width, op="fit_width") %}
<img alt="{{ alt }}" src="{{ resized.url | safe }}" />
```

## Examples

Here are some shortcodes for inspiration.

### YouTube

Embed a responsive player for a YouTube video.

The arguments are:

- `id`: the video id (mandatory)
- `playlist`: the playlist id (optional)
- `class`: a class to add to the `<div>` surrounding the iframe
- `autoplay`: when set to "true", the video autoplays on load

Code:

```
<div {% if class %}class="{{class}}"{% endif %}>
    <iframe src="https://www.youtube-nocookie.com/embed/{{id}}{% if playlist %}?list={{playlist}}{% endif %}{% if autoplay %}?autoplay=1{% endif %}" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>
</div>
```

Usage example:

```md
{{/* youtube(id="dCKeXuVHl1o") */}}

{{/* youtube(id="dCKeXuVHl1o", playlist="RDdQw4w9WgXcQ") */}}

{{/* youtube(id="dCKeXuVHl1o", autoplay=true) */}}

{{/* youtube(id="dCKeXuVHl1o", autoplay=true, class="youtube") */}}
```

### Image Gallery

See [content processing page](@/documentation/content/image-processing/index.md#creating-picture-galleries) for code and example.
