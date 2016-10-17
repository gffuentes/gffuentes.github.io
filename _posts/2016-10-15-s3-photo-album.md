---
layout: post
title:  "S3 Photo Album"
date:   2016-10-15 21:39:00 -0500
categories: static
image: photo-album.png
---
#Background

After many vacations of borrowing my partner's aunt's DSLR, we had put together some pretty nice albums of fancy photos. I did not have a place to publicly store these photos, so I did some research into online options like SmugMug and Flickr. These options seemed like overkill or did not meet my main goals. 

But what were my main goals?

- Cheap
- Not have to worry about tiered pricing given GB of photos
- Easily accesible by family and friends
- Organized by album for easy navigation

After finding a JS based photo gallery library (more to come on that later), I created a Sinatra based application served on an EC2 instance. After learning about static web pages hosted on S3, I thought I could do better (and even cheaper!). Below is a quick explanation of how I was able to create a photo album in S3.

#Application

**Ingredients:**

1. AWS S3 for Storage and Static Website Hosting
2. Twitter Bootstrap for style
3. [PhotoSwipe JS Library](http://photoswipe.com/)  for gallery
4. ImageMagick for thumbnail creation

**Setting up S3 Bucket**

For more detailed instructions, see [Setting Up a Static S3 Site](http://docs.aws.amazon.com/AmazonS3/latest/dev/website-hosting-custom-domain-walkthrough.html)

1. Create a bucket named after the URL of where your site will live. In my case, it was `on-vacation.gabefuentes.com`. This will be your website.

2. Create an additional bucket for storing the photos. Whatever name is fine.

3. Bucket structure I used:

        on-vacation.gabefuentes.com
        ├── gallery.html
        ├── home.html
        └── public
            ├── javascripts
            │   ├── photoswipe-ui-default.min.js
            │   └── photoswipe.min.js
            └── stylesheets
                ├── default-skin
                │   ├── default-skin.css
                │   ├── default-skin.png
                │   ├── default-skin.svg
                │   └── preloader.gif
                └── photoswipe.css

        image-storage
        └── images
            └── North Pole Christmas Trip
                ├── image_1 (4608x3456).jpg
                ├── image_2 (2592x1728).jpg
                └── small
                    ├── image_1.jpg
                    └── image_2.jpg
