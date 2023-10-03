---
title: Moving from jekyll to hugo
subtitle: Perfect is the enemy of good
date: 2023-10-02
tags: [ meta, hugo, bash ]
---

I'm dusting off the old blog. I was inspired to do this while building
my [website](https://felix-frank.net), which I did with
[hugo](https://gohugo.io) and enjoyed quite a bit.
So on a whim, I test-migrated the blog content to hugo as a proof of concept,
with a "rapid prototyping" state of mind: Just throw things at the wall,
see if anything sticks. If it's pretty, maybe even keep it.

# Top-down blog design

Rather than committing a new base to the [original blog
repository](https://github.com/ffrank/ffrank.github.io), I started
a fresh new repo and did the basic steps to make a site using the
[Beautiful Hugo](https://themes.gohugo.io/themes/beautifulhugo/) theme.
I had selected this theme through a rigorous process (it caught my
eye in the list of thumbnails and then I also liked the demo page).
When installing the theme, I skipped the step of copying the
contents of the "example site" folder into my blog directory.
I prefer to be minimalistic about the files that pile up here.

The old blog was very simplistic as well. It had a silly little
cover photo, a cheesy "about" page, and a very enthusiastic announcement
for the publishing of the book I wrote almost a decade ago.
This announcement received two additions through the years, specifically,
there was yet another announcement for the second edition, and the review
that was kindly guest-written by John Arundel.

I ended up dropping the cover photo. The blog may or may not receive
some art decoration at some point, but for now, I like it minimalistic.
The About page received a slight overhaul, and the book related content
I basically kept. I did add another "Update" to the original
announcement, to put it into perspective.

The four static pages I just copied from the root folder of the old
jekyll repository to the **content/page** folder for hugo.
With some simplification of the header fields, they were basically
good to go.

Next was the fun part: Moving all the posts over.

# Bashing things into shape

The structure for blog content in jekyll and hugo is mostly similar:
Markdown files live in **_posts/** or **content/post/** respectively,
and their file names look like `2023-10-02-moving-jekyll-to-hugo.md`.
Some files in the old blog used .markdown as their extension, but that
did not matter so much.

The meta data in each post file looked like this in the old blog:

```
---
layout: post
category: features
tags: [ puppet, module, resource, shell ]
summary: Introducing a new mode of running Puppet and syncing resources
---
```

In the hugo world, I wanted that to be just this:

```
---
title: Puppet Scripting Host
date: 2020-09-02
---
```

Both the **title** and **date** fields are derived from the respective
filename. So what needed to happen was:

1. discard the existing header
2. get the respective values from the filename
3. add a new header with these values
4. write this to a file in the hugo site

Any scripting language will make rather short work of this. I like doing
this in bash one-liners because it is so easy to iterate on the approach
before even writing a single output file. And once it looks fine, writing
the output is straight forward as well.

For example, this was an initial one-liner to get the basic metadata fields:

```
ls ../ffrank.github.io/_posts/ | tr -d \" | tr - \ | cut -d. -f1 | while read year month day title ; do head -n7 "../ffrank.github.io/_posts/$year-$month-$day-$( echo $title | tr \  -)".* ; done
```

A little more readably:

```
ls ../ffrank.github.io/_posts/ |
	tr -d \" |
	tr - \ |
	cut -d. -f1 |
	while read year month day title ; do
		head -n7 "../ffrank.github.io/_posts/$year-$month-$day-$( echo $title | tr \  -)".* ;
	done
```

So with each filename, the little invocations of `tr` and `cut` will

1. remove all quotes
2. replace all dashes with spaces
3. split the name at dots, and use only the first field (effectively removing the extension)

Then I use this little `while read` loop to assign tokens from each line of input to local variables
for **year**, **month**, **day**, and **title** (the latter using all tokens
that are left on the input line, which is what I want here).
Finally, something weird: In order to print the header from each file, I reconstruct
the original filename by converting spaces back to dashes.

So first of all, why even print the header? I wanted to make sure all of them looked alike,
and that there were no weird outliers in some of my posts. In order to check
for this, I ran the above line again, but piped its output to the following:

```
... | cut -d: -f1 | sort | uniq -c
```

I `cut` the output down to field names, sorted all lines, and for each
distinct occurrence, counted how many of them there were. The output looks
like this:

```
     30
     60 ---
     30 category
     30 layout
     30 summary
     30 tags
```

So it looks like I had a total of 30 posts, each with the exact same header
fields. (Copy/Paste still saving the day, years after the fact.)

So that's good, but it felt weird to rebuild filenames of input files
from the values I had painstakingly extracted. So I ended up replacing
the clever `ls | cut/tr | while read` loop with the following:

```
for post in $(ls ../ffrank.github.io/_posts/) ; do date=$( echo $post | cut -c1-10 ) ; name=$(echo $post | cut -d. -f1 | cut -c12-) ; title=$(echo $name | tr - \ | sed 's/\b\w/\U&/g') ; echo --- ; echo title: $title ; echo date: $date ; echo ---  ; done
```

Breaking it up for readability:

```
for post in $(ls ../ffrank.github.io/_posts/) ; do
	date=$( echo $post | cut -c1-10 ) ;
	name=$(echo $post | cut -d. -f1 | cut -c12-) ;
	title=$(echo $name | tr - \ | sed 's/\b\w/\U&/g') ;
	echo --- ;
	echo title: $title ;
	echo date: $date ;
	echo ---  ;
done
```

The `cut` invocations here take advantage of the fixed length of 10 characters
for the respective date. The **title** is again derived by replacing dashes with
spaces, but then there is also a regular expression via `sed` that capitalizes
each word (more or less) in those phrases.

# Why no python? 😭

Again, I like doing these shell one-liners because...well one because I'm a shell
nerd. But also, writing them is so seemless in my normal CLI workflow.
For one-shot work like this, I don't like python code files sitting in
arbitrary locations of my filesystem. Sure, they could go to /tmp, but then
I can just lose them to a thoughtless reboot, with my shell just remembering
the call, or even just opening them in vim.

As for the shell one-liners, bear in mind that just like a python script,
I don't write these in one go...they grow through a lot of iterations.
And it's easy to add the final file generation steps once the generated
headers from the previous step look correct. Essentially I just wrap
all those `echo` calls in braces, and write the output to the right
destination:

```
( echo ... ; tail -n +7 ../ffrank.github.io/_posts/$post ) > post/$date-$name.md
```

The `tail -n +7` is added in order to also write the original content minus
the metadata header, after my newly generated header. There, all good (almost).

# Final touches

The posts were mostly rendered fine by hugo, but of course, the
jekyll-specific shortcodes needed to be stripped still. The way
I like to do this is

1. check all the content into git as is
2. use `sed -i` and friends to try and regex-fix lots of stuff in one go
3. use `git diff` to check that the regex really did the right thing in (almost) all cases

It's fine if some exceptions get garbled. In that case, doing final repairs
manually is acceptable. Perfect is the enemy of good. But if it turns out
I missed something foundational, and `sed` happily destroyed most of my posts,
I can just use `git checkout` to undo all the silliness.

A commonly needed replacement was from the way that jekyll creates links
between posts:

```
[link text]({% post_url 2020-04-01-will-we-ever-go-outside-again %})
```

With the hugo theme I'm using, these should all just look like this:

```
[link text](/post/2020-04-01-will-we-ever-go-outside-again)
```

Regex through the whole set of posts

```
sed -i 's,{% post_url \(.*\) %},/post/\1/,g' content/post/*
```

In some places I had been naughty and not used that shortcode. I had
just linked to the jekyll-internal path, like hugo does it. And since
jekyll creates URLs depending on the type of posts, I had some links
with destinations starting with **/hacks/** or **/features/** etc.

```
sed -i s,articles/20,post/20, content/post/*
sed -i s,features/20,post/20, content/post/*
sed -i s,hacks/20,post/20, content/post/*
```

Another difference in jekyll URLs is the date representation. The date
does not just become part of the single URL path component, but rather
three distinct components, like

```
/hacks/2017/07/22/how-to-put-your-pants-on-sideways
```

Since everything was already translated to **/post/** paths, this
can now be fixed with

```
sed -i 's,/post/\(20..\)/\(..\)/\(..\)/,/post/\1-\2-\3-,' content/post/*
```

This took care of most stuff. At this point, looking through the remaining shortcodes
was quite relaxing (`grep '{%' -r content/post/`). All those that remained
looked like this:

```
{% highlight bash %}
{% endhighlight %}
{% highlight ruby %}
{% endhighlight %}
```

These are code block delimiters in jekyll, that enforce a certain syntax
hilighting. I think hugo can do something like that, but frankly, I cannot
be arsed at this point.

```
sed -i '/^{%/s/.*/```/' content/post/*
```

No more shortcodes.

# Preparing the move

At this point, I can essentially upload the new hugo generated blog and
link to it from the old github hosted place. But I also felt to go the
extra mile and replace the favicon. The old blog still used an old
Puppet community smiley face. This feels hardly fitting anymore.

So after some searching I found a cool open source icon that I liked.
You can see it credited on the homepage of the blog. I took the liberty
of adding some `style` to make it not horrible for Dark Mode enjoyers.

So here we are now. It was fun, it probably broke some edge cases,
but I don't mind so much. I hope you won't either. Hope I will manage
to be around here more again. And maybe you? 😊
