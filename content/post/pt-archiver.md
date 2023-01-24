---
title: "PR Archiver" # Title of the blog post.
date: 2011-12-14T09:57:07Z # Date of post creation.
draft: false # Sets whether to render this page. Draft of true will not be rendered.
toc: false # Controls if a table of contents should be generated for first-level links automatically.
usePageBundles: false # Set to true to group assets like images in the same folder as this post.
codeMaxLines: 10 # Override global value for how many lines within a code block before auto-collapsing.
codeLineNumbers: false # Override global value for showing of line numbers within code block.
figurePositionShow: true # Override global value for showing the figure label.
# comment: false # Disable comment if false.
---

**I’ve started to use the Percona Toolkit quite a lot and have recently found pt-archiver useful for 2 things.**

## Purging Large amounts of data

```bash
pt-archiver   –source h=$HOST, 
              D=$DB,t=$TABLE,P=$PORT 
              –where “$WHERE” –no-check-charset 
              –skip-foreign-key-checks –progress 1000 –purge
```

the only bit that probably needs explaining is the ```$WHERE``` it is the where clause e.g. ```creation_date < '2010-10-11'``


## Archiving Data


```bash
pt-archiver  –source h=$SOURCE_HOST, 
             D=$SOURCE_DB,t=$SOURCE_TABLE,P=$SOURCE_PORT 
             –dest h=$DEST_HOST, 
             D=$DEST_DB,P=$DEST_PORT 
             –where “$WHERE” –no-check-charset 
             –skip-foreign-key-checks –progress 1000 
```

Both of these use low impact methods to either purge or archive the data. It needs a primary key to work from. It’s got lots of options I have not played with yet [more info](http://www.percona.com/doc/percona-toolkit/pt-archiver.html)
