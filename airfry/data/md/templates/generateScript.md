---
title: Generating Scripts
---

# Generate Scripts

If you want to render several pages using the same template, specify a generate script at the end of your template.

A generator script is written in javascript as a promise that you must resolve or reject

Your script has access to the following:

## Inputs

### require

Allows you to import node.js compatible npm modules that you have added to your project. Use at your own risk.

### resolve

Call with the result of your script. See below for data you can pass when your script resolves.

### reject

Call with a failure message if something goes wrong.

### inputs

- **triggeredBy**: If the script is being called because an individual data file changed, this will be set to the path of the file so that you can proceed to render only that file.
- **frontMatter**: Allows you access to the template's front matter data.

### global

If you create data in your [pre generate script](/docs/templates/preGenerate/) it will be accessible here.

### getDataFileNames

A convenience function supplied by Airfry to get the absolute file names in folders relative to the data directory you specified when starting Airfry using the standard "glob" pattern matching.

Pass in a glob string or an array of glob strings and all of the matching file paths will be returned as an array.

### cache

The current cache. See [caching](/docs/performance/cache/) for details.

### log

Call to log info to the airfry console. This can be helpful when you want to debug your scripts.

### frontMatterParse

Airfry passes an instance of [frontMatterParse](https://github.com/jxson/front-matter) to your generate scripts which you can use in case you want to use front matter in your data files. This is what's happening in the following example, but is entirely optional.

### dataDir

The absolute path for your data directory in case you need it for any reason.

## Resolve Data

- **cache**: Data that you want to update or add to the cache. See [caching](/docs/performance/cache/) for details.
- **siteFiles**: An object which creates output files wrt your output directory. The keys are the file names and the values will be stringified with _JSON.stringify_ and written to the key specified path.
- **generate**: An array of [page generation requests](#pageGenerationRequests).
- **watchFiles**: request watching for changes to these files, Airfry will call this script with inputs.TriggeredBy set to the file path that change.
- **watchGlobs**: Tell Airfry to watch glob patterns and if they change, call this script with inputs.TriggerBy set to the path that changed.
- **outData**: Output data for all generate scripts will be collected and passed to your postGenerate script if it exists. See [post generate](/docs/templates/postGenerate/) for details.

## Page Generation Request Array

When your generate script resolves, you should return a list of pages and data to run through the template. Simply return an array of objects with properites path & data.

- **path**: The sub path to generate the template, which will replace the '\*' in your generate script path.
- **data**: The data to supply to the template to be used as page variables.

Example:

```html
---
generate: posts/*
---
<%= description %>

<script generate>
resolve(
  [
    {
      path: "first",
      data: {
        description: "first post",
      }
    },
    {
      path: "second",
      data: {
        description: "second post",
      }
    },
  ])
```

The above template will generate two posts:

```
└── output/posts/first
    └── index.html
└── output/posts/second
    └── index.html
```

## Advanced example of a generate script:

```html
<script generate>
  var md = require("markdown-it")();
  const fs = require("fs");
  let docs;

  if (!inputs.triggeredBy) {
    // Airfry will call your script at launch and when your template
    // file changes.  getDataFileNames is convenience utility supplied
    // by Airfry to get the absolute file names in folders relative
    // to the data directory you specified when starting Airfry
    // using the standard "glob" pattern matching.
    docs = getDataFileNames("md/**/*.md");
  } else {
    // Airfry will also call your script if one of the files you asked
    // for changes while Airfry is still running.  This lets you re-render
    // a single file, rather than rendering all data files again!
    docs = [inputs.triggeredBy];
  }

  const mapped = docs.map((filepath) => {
    const raw = fs.readFileSync(filepath, "utf8");
    const content = frontMatterParse(raw);
    const html = md.render(content.body);
    const title = content.attributes.title;
    // Use our data paths to specify output paths.
    // dataDir is supplied by Airfry and is the absolute path to our data dir
    // We also truncate the '.md' of path names.
    const relPath = filepath.split(dataDir + "/md/")[1].slice(0, -3);
    return {
      path: relPath,
      data: {
        title: title,
        html: html,
      },
    };
  });

  resolve({
    siteFiles: {
      ["README.md"]: "*This will get generated!*",
    },
    postData: {
      someData: "This will be passed to your post generate script.",
      otherData: "This will also be passed to your post generate script.",
    },
    // the list of pages to generate through this template:
    generate: mapped,
    // Tell airfry which data files to watch.
    // If they change after the initial render, this generate script
    // will be called with "inputs.triggeredBy" set to the file that changed.
    watchGlobs: ["md/**/*.md"],
  });
</script>
```

### Reusing Generate Scripts

To re-use a generate script which already exists in your project, use **generate-use** as follows

```
<script generate-use:"templateName">
</script>
```

A common pattern is to create a template with only the generate script specified.

You can see an example of how this works in the docs for this site:

[docsMD.ejs](https://github.com/jaunt/airfryDocs/blob/main/airfry/templates/generators/docsMD.ejs).

The above generate script is referenced from this template:
[pages.ejs](https://github.com/jaunt/airfryDocs/blob/main/airfry/templates/pages.ejs)