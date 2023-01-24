---
title: "Changing Settings for all MySQL tables" # Title of the blog post.
date: 2011-01-23T09:57:07Z # Date of post creation.
draft: false # Sets whether to render this page. Draft of true will not be rendered.
toc: false # Controls if a table of contents should be generated for first-level links automatically.
usePageBundles: false # Set to true to group assets like images in the same folder as this post.
codeMaxLines: 10 # Override global value for how many lines within a code block before auto-collapsing.
codeLineNumbers: false # Override global value for showing of line numbers within code block.
figurePositionShow: true # Override global value for showing the figure label.
# comment: false # Disable comment if false.
---

As part of my migration to MariaDB I wanted to change all the wordpress tables from MyISAM to INNODB. Of course you can do this one table at a time with
alter table fred engine=innodb;
Personally I prefer to automate where ever possible so  I created the following linux shell script to convert all the tables in one database in one go

```bash
DBNAME=”blog”
DBUSER=”xxxx”
DBPWD=”xxxxxxxx”
for t in $(mysql -u$DBUSER -p$DBPWD –batch –column-names=false -e “show tables” $DBNAME);
do
echo “Converting table $t”
mysql -u$DBUSER -p$DBPWD -e “alter table $t engine=innodb” $DBNAME;
done
```

This script could of course be used for many things by just changing the SQL in line 7
