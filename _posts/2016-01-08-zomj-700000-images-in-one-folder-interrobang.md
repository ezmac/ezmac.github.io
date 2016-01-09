---
layout: post
title:  "700,000 images in one folder?!"
date:   2016-01-08 11:38:37
categories: work_related bash
---

Or how to crash Coldfusion without changing a thing

One of the services we run at <strike>redacted</strike> scrapes feeds for news stories and makes them available for inclusion in our feeds.  It does this up to 100 entries at a time, every 5 minutes.  Due to an uncaught bug, we had over 700,000 images in one directory. This brought to our attention that having 700,000 files in one directory crashes coldfusion. It also brought to our attention that we've accumulated over 32,000 _real_ images since the site launched. It turns out linux doesn't like having a lot of files in a single directory, so it was time to do some cleanup.  Our images are stored in folders according to some pre-set sizes.

Our saving grace was that for each image we needed to keep, there was a database row with the image name in it.  All we really needed to do was pull a list of those files and pay someone on fiverr to manually sort through.. Wait a sec.  [What's our mantra?](http://www.geekpeak.de/images/produkte/i22/22-go-away-or-i-will-replace-you-de.jpg).  So, step 1 was to pull a \n separated list of files to keep from the database.  I let a co-worker do this and it got done as coldfusion, but it got done and *Props!* I could curl it as frequently as I wanted!

`curl http://dev.redacted.tld/images.cfm | uniq|sort >goodimages.txt`

It would have been so much easier if he had just given me a list of files to delete instead of ones to keep.  Well, forunately, Linux has a tool for everything and it's called Bash.  All I need to do to get a list of files to keep is get a list of files and remove the files that should be kept.  That sounds like a good use of `comm`.  Comm shows three columns: unique to file1, unique to file 2, lines in both files.  You can check out the manual, but what we want is `comm -2 -3 all.txt good.txt`.  This form leaves column 1 only.  One note with comm is it wants to work on sorted files, so we'll handle that too.  Now, we need just a list of files to work on.  I started out using `ls |...` but switched to `find` for the list and `sed` to remove the relative prefix.  Parsing `ls` isn't usually a very portable solution.


{% highlight bash %}
  #generate list of all images in a directory
  find ../360x360 -type f -maxdepth 1|sed 's|../360x360/||g'|uniq|sort> 360x360.txt

  # sanity check.  returns the number of images to be deleted.`
  comm -2 -3 360x360.txt goodimages.txt|wc -l 
{% endhighlight %}

Since this was a production system, I decided that I didn't want to just delete the files.  This can probably be done easier, but I used a bash script to move the file to another folder.  

{% highlight bash %}
  #!/bin/sh
  #safedelete.sh
  dir=$1
  if [[ ! -d ../errant/$dir/ ]]; then
    mkdir -p ../errant/$dir/
  fi
  if [[ -f ../$dir/$2 ]]; then
    mv ../$dir/$2 ../errant/$dir/$2
  fi
{% endhighlight %}

If the numbers look reasonable, actually delete them.

`comm -2 -3 360x360.txt goodimages.txt|xargs -n 1 -P 15 ./safedelete.sh 360x360`

You may notice all the relative directories.  Photos contained folders for sizes, an errant folder, and a cleanup folder.  The above commands were run from the cleanup folder.

## The Walrus

After a little research and napkin math, we decided 2000 or fewer images per directory would be acceptable since we didn't _really_ have a problem until we hit 700,000.  I'll save you the boring math, but I stole an idea from git and decided to use buckets.  Linux can handle large heirarchies of files, but get too many files in any single directory and performance is slowed.  A work around for this is to create subfolders to store groups of files. Our image names are hex and there were 32,000 of them.  Assuming an even distribution, 2^16=256; 32000/256 ~= 125.  Seems a bit like overkill, but the next option was one character and 16 buckets.  At least this way, we won't have more than 10,000 images in a folder for at least 20 years (napkin math is not boring).

With that part out of the way, I wrote a suite of bash scripts I like to call [Walrus](https://github.com/CU-WebTech/walrus). Once everything was cleaned up, we were able to clone walrus into out images directory and run it for each size we had.  To run walrus, execute `./createBuckets.sh folder` followed by `./stackBuckets 15 directory` where 15 is number of processes to use moving images.  If you're using this in production, you may also want to put in the rewrite rule before moving a lot of files that are still being served up.


