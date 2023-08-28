---
id: parcel-quicktemplate
title: Using Parcel with Go Templates
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
I later wanted to manage fonts better than just statically serving `${pkgs.jetbrains-mono}/share/fonts/truetype/JetBrainsMono[wght].ttf`, so a different solution was needed.


<tangent>
Tailwinds "I'm gonna search for class names I know" thing dosen't actually parse your html and just searches for classnames, so it even works fine with template engines
</tangent>


In the JavaScript SPA world the use of bundlers like webpack/vite/parcel is ubiquitous, so this is the story about how I adapted parcel to work with my blog engine.

## How my blog engine works

My blog engine is build around a SQLite database storing Posts, tags (that aren't used yet) and metadata (that also isn't used yet).
A go server reads the database and renders the markdown to HTML/RSS feeds.

<tangent>
[gomarkdown](https://github.com/gomarkdown/markdown) is a pretty great markdown renderer. you can easily modify it's parser/renderer to, for example, build such tangent blocks.

<tangent>
using a static site generator would 100% be enough for what my blog is now, but where's the fun in that


<tangent>
damn, I can stack tangents
</tangent>

</tangent>

</tangent>

The rendered markdown then gets templated into a [quicktemplate](https://github.com/valyala/quicktemplate) to generate the HTML sent to the reader.

<tangent>
quicktemplate is actually not a runtime templating engine, but a code generator. The code it generates is pretty equivalent to just `writer.Write("string")`, so it's really fast.
</tangent>

I wanted parcel to take the templates and modify them to include CSS, fonts and if I ever decide to integrate frontend JS, that as well.

## What is a bundler and why do I want to use it

A bundler takes all your JS/Fonts/CSS files and combines them to a minimum of files.

This reduces the amount of requests your browser has to make ad it makes caching easier.

It also allows the bundler to merge your code and it's dependencies, so for example if you do 

```css
@import 'npm:@fontsource-variable/jetbrains-mono/wght-italic.css';
```

in an included CSS file, your bundler can automagically also output the font into your application.

Parcel by default outputs files with hashes in their file name, so you can just tell your Web server to set its cache policy to forever and bam, easy caching.

<tangent>
This is so cool
</tangent>

## Getting parcel to play nice with templates

### How parcel works

To understand how parcel works, reading its [Plugin System Overview](https://parceljs.org/plugin-system/overview/) is probably the best resource.
But I'll try to give a short overview to describe where I needed to hook in to make it work with `qtpl`.

Parcel has Resolvers and Transformers to figure out, assets make up your project

**Resolvers:**

Resolvers turn dependency requests into absolute paths. So it'll and convert our `npm:@fontsource-variable/jetbrains-mono/wght-italic.css` import to `<project_dir>/node_modules/@fontsource-variable/jetbrains-mono/jetbrains-mono.css`

**Transformers:**

Transformers take a file and convert it somehow. So if you had for example a SCSS file, a transformer would convert it to CSS. Another Transformer might minimize an HTML file.
They also add dependencies to the asset graph for the resolvers to resolve. So our JetBrains-Mono gets added by a CSS transformer to the asset graph.


The Assets then get bundled (by Bundler plugins) to combine files where possible, named (by Namer plugins) to figure out file paths, and then they're written to the output directory.

<tangent>
There are other steps like Compressors or Validators but we'll ignore them here
</tangent>

### The Parcel Plugins I needed to write

Parcel plugins are their own JS Projects with their own package.json,...
Using yarn workspaces this wasn't even as painful as I had thought.

<tangent>
[now they even can be just JS Module files](https://parceljs.org/features/plugins/#relative-file-paths), that would have been so much easier
</tangent>

The main JS file of the Plugin just has to default export the Plugin class itself.

#### A resolver to ignore most kind of imports in `.qtpl` files

In the blog engine, links to posts/etc. get templated into the page at runtime.
Parcel tries to import anything, even links to template strings.

So I needed to build a resolver that just ignores imports from `.qtpl` files if the import isn't CSS or JS

<tangent>
if you want to adopt it to other templating engines like `html/template`, just change the file endings
</tangent>

That's the resolver:

```js
// packages/parcel-resolver-qtpl/src/index.js
const { Resolver } = require('@parcel/plugin'); // cjs is ugly but it just worked and I'm lazy

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

#### A Namer to place assets into a different directory

The default Namer just puts all your assets into the same directory. But as the output consists of both files to be read by the templating engine and assets, the files needed to be split into different directories.

- Templates go to `/templates` (I set this as the primary output path)
- anything else goes to `/statics/dist`

Writing Namers is also surprisingly simple. You just need to return the file path you want the file to have in the end (relative to the primary output path).

```js
// packages/parcel-namer-split/src/index.js
const { Namer } = require('@parcel/plugin');
const path = require('node:path');

exports.default = new Namer({
  name({ bundle }) {
    if (bundle.type != "qtpl") {
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

Now we just have to write a parcel configuration that combines all the plugins with the defaults.
A parcel configuration is just a JSON5 file describing what plugins to use.

If you don't have any `.parcelrc` it'll just use `@parcel/config-default` as its configuration.

We'll just extend `@parcel/config-default` because it does all the CSS transforming/... for us

```json
{
  "extends": "@parcel/config-default",
  "resolvers": ["parcel-resolver-qtpl", "..."],
  "transformers": {
    "*.qtpl": [
      "@parcel/transformer-posthtml", // the default html transformers
      "@parcel/transformer-html"
    ],
    "*.jsonld": ["@parcel/transformer-raw", "@parcel/transformer-inline-string"]
  },
  "packagers": {
    "*.qtpl": "@parcel/packager-html" // the default html packager
  },
  "namers": ["parcel-namer-split", "..." ],
}

```

<tangent>
`"..."` just includes the defaults
</tangent>

I needed to explicitly handle `jsonld` and tell parcel to do nothing with it, as Parcel will - by default - transform JSON-LD meta tags to resolve listed dependencies, etc.
I inject my JSON-LD at runtime, so it tried to parse the template string as JSON, without much success.

<tangent>
`@parcel/transformer-raw` just takes the input and returns it as an output file.

`@parcel/transformer-inline-string` takes an input and returns it as an inline string. The HTML transformer doesn't like to write files into itself
</tangent>


Including Tailwind CSS was as easy as just following [Tailwinds tutorial for PostCSS](https://tailwindcss.com/docs/installation/using-postcss)

## Building the whole thing with nix

<tangent>
You didn't *really* think you'll get a post without nix, did you?
</tangent>

My build process consists out of three parts:

1. Build the templates with parcel
2. Convert the templates to go code with quicktemplate
3. Build the go project

### Building the templates with parcel in nix

I'm using yarn right now, so I just tried using `yarn2nix` (included in nixpkgs) to build the yarn project with nix.

The parcel plugins are part of yarn workspaces, which we need to include manually with the `yarn.lock` of the root package.

```nix
# flake.nix (excerpt)
xnoblog_tmpl = pkgs.mkYarnPackage rec {
  pname = "xynoblog_tmpl";
  version = "0.0.1";
  workspaceDependencies =
    let
      deps = map
        (x:
          pkgs.mkYarnPackage { # generate a yarn package for everything
            src = "${./packages}/${x}";
            yarnLock = src + "/yarn.lock"; # use root lock file
            fixupPhase = "true";
            inherit version offlineCache; # inherit the parents version and cache
          }
        )
        (builtins.attrNames (builtins.readDir ./packages)); # import all packages in the packages directory
    in
    deps;
  offlineCache = pkgs.fetchYarnDeps { # this fetches yarn dependencies into nix
    yarnLock = src + "/yarn.lock";
    # sha256 = pkgs.lib.fakeSha256;
    sha256 = "sha256-ImagineARealSHA256Here/ItGetsGeneratedByNix="; # reproducible ✨
  };
  src = ./.;
  distPhase = "true"; # we do everything in the buildPhase
  installPhase = "true";
  fixupPhase = "true";
  buildPhase = ''
    export HOME=$(mktemp -d) # yarn needs $HOME to be set
    mkdir -p $out/templates # create output directory
    yarn --offline parcel build --dist-dir $out/templates # run parcel
  '';
};
```

Now all the build templates/assets are built into a nix derivation.

<tangent>
In `pkgs.fetchYarnDeps`, you get the right sha256 just like you do with `pkgs.buildGoModule`.

Setting it to `pkgs.lib.fakeSha256` and seeing onto which sha256 it mismatches
</tangent>

### Converting templates and building the application

I just put template copying/building into the derivation of the application itself.

```nix
# flake.nix (excerpt)
xynoblog = pkgs.buildGoModule rec {
    pname = "xynoblog";
    version = "0.0.1";
    src = ./.;
    nativeBuildInputs = [ pkgs.quicktemplate ... ];
    preConfigure = ''
      cp -r ${self.packages.${pkgs.system}.xynoblog_tmpl}/{statics,templates} . # copy the templates into application sources
      chmod +w -R ./{statics,templates} # we need to write to them
      qtc -dir=templates # run the code generator
    '';
    # in the buildPhase it'll turn into a normal go application

    ...
  };
```

<hr/>

That's how I use Parcel with a template engine.
If you want to read my blogs source code, it's open source and on [GitHub](https://github.com/thexyno/blog).

But please don't base your blog engine on it, and [just learn a new language, and write your own](https://xeiaso.net/blog/new-language-blog-backend-2022-03-02)

<br>

Thank you for reading, and a big thanks to
[Arson](https://chaos.social/@nzbr)
for their input and help in writing this post.
