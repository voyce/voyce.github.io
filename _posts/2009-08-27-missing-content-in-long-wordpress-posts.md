---
date: 2009-08-27 23:44:25+00:00
description: ''
featuredImage: build/gatsby/www.ianvoyce.com/assets/2009-08-27-missing-content-in-long-wordpress-posts_wordpress_icon.jpg
slug: /missing-content-in-long-wordpress-posts
template: blog-post
title: Missing content in long Wordpress posts
tags:
- php
- wordpress
---

[![wordpress_icon](http://72.47.193.211/wp-content/uploads/2009/09/wordpress_icon.jpg)](http://72.47.193.211/wp-content/uploads/2009/09/wordpress_icon.jpg)I've just spent several hours struggling with an annoying Wordpress problem: when I edited a post to make some additions, it suddenly stopped displaying any content. The title, header and footer were still visible, only the post body itself was missing.

After a bit of poking around, I came across [this post](http://www.undermyhat.org/blog/2009/07/sudden-empty-blank-page-for-large-posts-with-wordpress/). It points out the root cause of the problem is the PHP regular expression engine (PCRE) that Wordpress uses.

The easiest way to fix the problem is to ignore the advice in that post about modifying formatting.php, and instead make the fix to PHP.INI which is mentioned in the comments. I edited my copy to include the following line, which increases the backtracking limit from the default of 100000:

    
    
    pcre.backtrack_limit = 1000000
    



After getting IIS to pick up the changes (iisreset) my missing content appeared again.
