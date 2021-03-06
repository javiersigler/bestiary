# Templates

The Go Standard Library offers templating of both text and html content. The html templates offer a superset of the text template features, including contextual escaping, but the interface can be a little confusing.

## The missing docs

There is extensive documentation of the template packages, but most of it is in the [text/template](https://golang.org/pkg/text/template/) package, not the [html/template](https://golang.org/pkg/html/template/) package, so be sure to read that as well, even if you intend to focus on using html/template. The html template package extends the text/template package and uses much of its functionality, so you need to read the documentation for both.  

## Delimiters

If you have problems with the go template delimiters clashing with other libraries, you can use the Delims function in order to set the delimiters to some other string. Note this must be done first, before files are parsed:

```go
// Set the delimiters on this template
tmpl, err := template.New("").Delims("[[", "]]").ParseFiles("foo.tmpl", "bar.tmpl")
```

## Dot

The dot character in go templates represents the data context, and is set on Execution. Within functions like range the dot is set within that scope to the successive elements of the array. To access the parent context within a range use `$.`.

## Inline templates

While normally templates are complex enough to require their own files, you may find it useful to define templates as strings inline in your go code, particularly if sharing them on the playground. When templates are over a few lines this can become cumbersome and you might want to consider storing them in files and reading them in to a template set. 

```go
    // Define a simple template to print data
    inline := `Hello, {{.}}`

    // Set data to the string world!
	data := `world!`
	
    // Load the template
	t, err := template.New("t").Parse(inline)
	if err != nil {
		panic(err)
	}

	// Print the template 't' to stdout
	// using data as the context
	err = t.ExecuteTemplate(os.Stdout, "t", data)
```

Prints the template "Hello, " + the data "world!"
> Hello, world!

## Escaping 

The Go template package assumes that template authors (and by extension the templates) are trusted, but that the data inserted into the template is not. The HTML templates offer contextual escaping, so the package knows about the structure of html and the context in which data is escaped matters - the same data inside a url, an attribute or text content may be escaped differently. 

For example the text "O'Reilly & co" is escaped differently depending on whether it is found inside script tags, an attribute or an href attribute.

```go
    tmpl := `Hello, <a class="{{.}}" href="{{.}}"> {{.}}</a><script>{{.}}</script>`
	data := `O'Reilly >`
	
    // Load the template
	t, err := template.New("t").Parse(tmpl)
	if err != nil {
		panic(err)
	}

    // Execute the template
	err = t.ExecuteTemplate(os.Stdout, "t", data)
```

> `Hello, <a class="O&#39;Reilly &amp; co" href="O%27Reilly%20&amp;%20co"> O&#39;Reilly &amp; co</a><script>"O'Reilly \u0026 co"</script>`

If you need to bypass escaping, you can use the appropriate [typed string](https://golang.org/pkg/html/template/#hdr-Typed_Strings) like template.HTML in a function or field, to indicate to the package that this is trusted content. Be very careful not to use user content in such cases, or to escape it properly before use.  

## Structs, fields and methods

There is a uniform interface for fields and methods of a struct - the . operator will call either a field or a method. The same is true of map entries. You can also use the built-in function index to access members of Maps and Slices.

```go
{{ index mymap 0 }}
```

## Func Maps

Templates have a set of global functions available, which are documented in the [text/template](https://golang.org/pkg/text/template/#hdr-Functions) package, these offer logical operations, printing and a few other conveniences. 

To extend these Templates offer a FuncMap, which allows you to add functions to templates if you need to format data in a particular way (for example formatting currencies), or provide global data like the title of your website or string translations without threading it through all the different handlers and setting it explicitly as part of the data provided to templates. Don't get too carried away adding functions to your templates though - they are deliberately pared down version of go so that most of the logic resides in your .go files, rather than in template files, and to keep the attack surface to a minimum. Templates are deliberately minimalist in Go and you should strive to keep them so. 

## Prefix operators 

The boolean operators available in templates are all functions, which means they use prefix notation. Use _and_ x y, not x _and_ y. If you're coming from Ruby this may come as a surprise, as using _and_ with one argument will appear to work, but does not behave as you might expect. 

```go
{{/* Don't do this, it will always be true if x is non nil */}}
{{ if x and y}}

{{ end }}

{{/* Do this instead */}}
{{ if and x y}}

{{ end }}
```

## Range functions 

Go templates also allow you to range, but unlike the range in Go code, using this function with one argument yields the value, not the index.  

```go
{{ $v := range .values }}
```

There are numerous small differences between Go templates and Go code like this, so don't assume that the template language reflects Go norms.

## Writing comma separated lists

If using range in templates to write a list, which must be comma separated (JSON for example requires commas on all array items, but not the last), the neatest way to do this is to use if $i immediately after the range call:

```go 
// Print ',' before every element but the first one
{{ $i, $v := range .values }}{{if $i}}, {{end}}
<a href="/{{.}}"> {{.}} </a>
{{ end }}
```

## Printing values 

The printf function is available in templates to print values as strings:  

```go
{{ printf "%s %d %d" "hello" 1 1 }}
```

## The line eater

If you need to add newlines in your template for legibility (for example when outputting many columns of csv), you can invoke the line eater with the - symbol beside the braces (either start or end braces). The template:

```go
{{- .One }}
{{- .Two -}}
```

will output the values with whitespace removed:

> foobar

## Template Sets

There are several different ways to define templates and name them - they can be named inline in the template, or named by one of the ParseFiles or ParseGlob functions, or named on creation with template.New("t"). If using ParseGlob you'd have to make sure the file names used are always unique, which can be problematic in a large project. Another approach is to use the relative path of template files as the name, to ensure uniqueness, and make it clear when reading templates which file on disk is being included. 

Templates can only include to those in the same set, so you may find it convenient to load all templates into the same set with distinct names (perahps using the location of the templates loaded) so that they can be pulled in to any template by path.

## Rendering Nested Templates 

To render a template within another template, assuming they are in the same set, use the template function, which takes the name of the content to place, and the data context to render it with (use the dot operator to obtain the current context). 

```go
<body>
<h1>Hello</h1>
{{ template "views/users/content.html.got" . }}
</body>
```

This will render the template with name "views/users/content.html.got" inside the main template above.

## Template Blocks

Template blocks, introduced in Go 1.6, can be used to define areas of a template to replace with other content by overlaying it with another template. The master template should be created as normal, then Clone used to copy it with an overlay template. This may seem counter-intuitive at first. 

Another approach to achieve a similar result is to create a layout template with a `.content` key, which can then have content (itself rendered from a separate template) inserted into it using this key.

```go
<body>
<h1>Hello</h1>
{{block "content" .}}
<p>Default content</p>
{{ end }}
</body>
```


