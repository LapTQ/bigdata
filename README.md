# Big Data Storage and Processing

## Exporting a markdown book to PDF with Pandoc

### Install dependencies

```
$ sudo apt install pandoc
$ sudo apt install texlive-latex-base
```

### Structure

```
├── book
│   ├── Chapter1
│   │   ├── Scene1.md
│   │   └── Scene2.md
│   └── Chapter2
│       └── Scene1.md
├── images
│   └── lostbook.jpg
└── title.txt
```

### Compile

The below command will add table of contents, output to book.pdf, get title info from title.txt and grab three markdown files.

```
$ pandoc --toc -o book.pdf title.txt .\book\Chapter1\Scene1.md .\book\Chapter1\Scene2.md .\book\Chapter2\Scene1.md
```

### References

* https://github.com/TheFern2/markdown-book
* https://www.markdownguide.org/basic-syntax/