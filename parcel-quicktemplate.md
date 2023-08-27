---
id: parcel-quicktemplate
title: Using parcel with go templates
created: 2023-08-27
tags:
  - go
  - quicktemplate
  - tailwind
  - tailwindcss
  - parcel
---

When I built my blog engine 
(of because I build my own blog engine, because where is the fun in using a premade one)
I really wanted to try TailwindCSS for a first time.

It seemed like a nice way to not have to write too much CSS myself and everyone was talking about it at the time.

You can actually use tailwind just by itself, and it works, which is what I did at the start.
I later wanted to manage fonts better than just statically serving `${pkgs.jetbrains-mono}/share/fonts/truetype/JetBrainsMono[wght].ttf`, so a better solution was needed.


<tangent>
Tailwinds "I'm gonna search for class names I know" thing dosen't actually parse your html and just searches for classnames, so it even works fine with template engines
</tangent>


In the Javascript SPA world the use of bundlers like webpack/vite/parcel is ubiquitous, so this is the story about how I adapted parcel to work with my blog engine.

## How my blog engine works

My blog engine is build around a SQLite database storing Posts, tags and metadata.
A go server reads the database and renders the markdown to HTML/RSS

<tangent>
[gomarkdown](https://github.com/gomarkdown/markdown) is a pretty great markdown renderer. you can easily modify it's parser/renderer to for example build such tangent blocks.

<tangent>
using a static site generator would 100% be enough for what my blog is now, but where's the fun in that


<tangent>
damn, I can stack tangents
</tangent>

</tangent>

</tangent>

The rendered markdown then gets templated into a [quicktemplate](https://github.com/valyala/quicktemplate) to generate the HTML sent to the reader.

<tangent>
quicktemplate is actually not a runtime templating engine, but a code generator. it generates code that nearly just consists of something equivalent to `writer.Write("string")` so it's really fast.
</tangent>

I wanted parcel to take the templates and modify them to include CSS, fonts and if I ever decide to want to integrate frontend JS, the frontend JS.

## What is a bundler and why do I want to use it

A bundler takes all your JS/Fonts/CSS files and combines them to a minimum of files.

This reduces the amount of requests your brower has to make ad it makes caching easier.

It also allows the bundler to merge your code and it's dependencies, so for example if you do 

```css
@import 'npm:jetbrains-mono';
```

in an included CSS file, your bundler can automagically also output the font into your application.

Parcel by default outputs files with hashes in their file name, so you can just tell your Web server to set it's cache policy to forever and bam, easy caching.

<tangent>
This is so cool
</tangent>

## Getting parcel to play nice with templates

### How parcel works

To understand how parcel works, reading its [Plugin System Overview](https://parceljs.org/plugin-system/overview/) is probably the best resource.
But I'll try to give a short overview to describe where I needed to hook in to make it work with `qtpl`

Parcel has Resolvers and Transformers to figure out, out of which assets your Project consists

**Resolvers:**

Resolvers turn dependency requests into absolute paths. So it'll and convert our `npm:jetbrains-mono` import to `<project_dir>/node_modules/jetbrains-mono/css/jetbrains-mono.css`

**Transformers:**

Transformers take a file in one format and convert it to another. So if you had for example a SCSS file, a transformer would convert it to CSS.
They also add dependencies to the asset graph for the resolvers to resolve. So our `@import 'npm:jetbrains-mono';` gets added to the asset graph.


The Assets then get bundled (by a Bundler plugin) to combine files where possible, named (by a Namer plugin) to figure out file paths and then written.

### The Parcel Plugins I needed to write

Parcel plugins are their own JS Projects with their own package.json,...
Using yarn workspaces this wasn't even as painful as I had thought.

The main JS file of the Plugin just has to default export the Plugin class itself.

#### A resolver to ignore most kind of imports in `.qtpl` files

Parcel tries to import anything, even template strings.
I needed to build a resolver that just ignores imports from `.qtpl` files if the import isn't CSS or JS

That's the resolver:

```js
const { Resolver } = require('@parcel/plugin');

exports.default = new Resolver({
  async resolve(x) {
    if (!x.dependency | !x.dependency.sourcePath) return null; // dependency can be undefined
    // make sure only css and js files are included from qtpl files
    if (x.dependency.sourcePath.endsWith(".qtpl") &&
      !(x.specifier.endsWith(".css") || x.specifier.endsWith(".js"))) {
      return { isExcluded: true };
    }
    return null;
  }
});
```

simple, isn't it

#### A transformer that just does nothing

Parcel will - by default - transform JSON-LD meta tags to resolve listed dependencies, etc.

I inject my JSON-LD at runtime, so it tried to parse the template string as JSON, without much success.
It isn't even possible to say that an asset doesn't need a transformer (I think, if you know how to do that send me a message) so I needed a transformer that does nothing.

```js
const {Transformer} = require('@parcel/plugin');

exports.default = new Transformer({
        async transform({asset}) {
                return [asset];
        }
});
```

done

#### A Namer to place assets into a different directory

I bundle my whole blog into a single binary using `go:embed`, so the paths of the template and asset files needed to be different.

- Templates go to `/templates`
- anything else goes to `/statics/dist`

Writing Namers is also surprisingly simple. You just need to return the file name you want the file to have in the end.

```js
const {Namer} = require('@parcel/plugin');
const path = require('node:path');

exports.default = new Namer({
        name({bundle}) {
                if(bundle.type != "qtpl") {
                   let filePath = bundle.getMainEntry().filePath;
                   let bn = path.basename(filePath).split(".")
                   let hr = bundle.needsStableName ? "." : `${bundle.hashReference}.`
                   return `../statics/dist/${bn[0]}.${hr}${bn.slice(1).join("")}`;
                }
                return null;

        }
});
```

### Combining plugins to have a working parcel configuration
That's all the needed plugins. 
