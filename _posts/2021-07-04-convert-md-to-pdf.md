---
layout: post
title: Convert MD to pdf  
tags: linux
---

Making notes is easier on MD, so that it can be uploaded to git easily. However if these need to be shared as individual files, it is good to convert to pdf and send it across. 

using the tool mdpdf. it is super easy

```
https://pypi.org/project/mdpdf/
```
## Install mdpdf
```
pip install mdpdf
```

## Usage
```
mdpdf -o <output file name> <input md file > 
```

## help
```
mdpdf --help


➜  _posts git:(master) ✗ mdpdf --help
Usage: mdpdf [OPTIONS] [INPUTS]...

  Convert Markdown to PDF.

  For options below, <template> is a quoted, comma-
  delimited string, containing the left, centre,
  and right, header/footer fields. Format is

    "[left],[middle],[right]"

  Possible values to put here are:
    - Empty string
    - Arbitrary text
    - {page} current page number
    - {header} current top-level body text heading
    - {date} current date

Options:
  -o, --output FILE        Destination for file output.  [required]
  -h, --header <template>  Sets the header template.
  -f, --footer <template>  Footer template.
  -t, --title TEXT         PDF title.
  -s, --subject TEXT       PDF subject.
  -a, --author TEXT        PDF author.
  -k, --keywords TEXT      PDF keywords.
  -p, --paper [letter|A4]  Paper size (default letter).
  --version                Show the version and exit.
  --help                   Show this message and exit.
```
