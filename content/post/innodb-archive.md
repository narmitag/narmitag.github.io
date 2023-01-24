---
title: "Converting Innodb Tables to Archive" # Title of the blog post.
date: 2012-01-14T09:57:07Z # Date of post creation.
draft: false # Sets whether to render this page. Draft of true will not be rendered.
toc: false # Controls if a table of contents should be generated for first-level links automatically.
usePageBundles: false # Set to true to group assets like images in the same folder as this post.
codeMaxLines: 10 # Override global value for how many lines within a code block before auto-collapsing.
codeLineNumbers: false # Override global value for showing of line numbers within code block.
figurePositionShow: true # Override global value for showing the figure label.
# comment: false # Disable comment if false.
---

I recently had to set up a lot of Archive storage engine tables for several 100 InnoDB tables.

To do this conversion I had to script off all the indexes and in some cases the primary keys. I did a MySQLDUMP and ran the following commands against the dump file.

```bash
sed ‘s/ENGINE=InnoDB/ENGINE=archive/g’  dump.dmp > dump2.dmp
cat dump2.dmp | grep  -v ‘^  UNIQUE KEY’ > dump3.dmp 
cat dump3.dmp |grep -v ‘^  KEY’ >dump4.dmp
```

dump4 was then imported into the new archive database
