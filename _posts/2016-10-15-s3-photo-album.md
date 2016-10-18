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

**Setting up S3 Buckets:**

For more detailed instructions, especially on how to setup route53 to host your custom domain, see [Setting Up a Static S3 Site](http://docs.aws.amazon.com/AmazonS3/latest/dev/website-hosting-custom-domain-walkthrough.html)

1. Create a bucket named after the URL of where your site will live. In my case, it was `on-vacation.gabefuentes.com`. This will be your website. This is assuming you have your own registered domain. You can also set up an S3 static website using their domain (see [Setting Up a Static S3 Site](http://docs.aws.amazon.com/AmazonS3/latest/dev/website-hosting-custom-domain-walkthrough.html)) 

2. Create an additional bucket for storing the photos. Whatever name is fine.

3. Bucket structure I used (will discuss specifics later):

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

4. Configure website bucket (in my case the `on-vacation.gabefuentes.com` bucket) to be publicly accessible by adding the following bucket policy to `Properties > Permissions > Edit bucket policy`:
    
    *from [Setting Up a Static S3 Site](http://docs.aws.amazon.com/AmazonS3/latest/dev/website-hosting-custom-domain-walkthrough.html)
    NOTE: The `"Resource":` key should be your bucket name
     
        {
	        "Version": "2012-10-17",
	        "Statement": [
		        {
		    	    "Sid": "AddPerm",
		    	    "Effect": "Allow",
		    	    "Principal": "*",
		    	    "Action": "s3:GetObject",
		    	    "Resource": "arn:aws:s3:::on-vacation.gabefuentes.com/*"
		        }
	        ]
        }

5. Configure website bucket to enable website hosting in `Properties > Static Website Hosting > Enable website hosting`. Type your index document in the open field - in my case, `home.html`.

6. Your S3 static website should now be live! In S3, under `Properties > Static Website Hosting` you will be able to see the `amazonaws.com` URL. Confirm that it is up, and you can reroute your own hostname in Route53 if you so choose.

**Creating the Photo Album Views:**

For more detailed instructions on how to implement PhotoSwipe, please view their website [here](http://photoswipe.com/).

For more details on the AWS Javascript API, view their docs [here](http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html).

1. I wanted a `home.html` index page to dynamically list each photo album (given by the name of each folder in the `image-storage` bucket) and link to related gallery. I will post the whole code at the bottom, but below is the "beef" of the work.

        var trips = { Bucket: "gabe-storage", Prefix: "images/", Delimiter: '/', StartAfter: 'images/' }

        s3.listObjectsV2(trips, function(err, data) {
          if(err) {
            console.log(err, err.stack)
          } else {
            $.each( data.CommonPrefixes, function (index, value) {
              var galleryName = value.Prefix.replace(galleryRegex,"")
                $(".container").append("<h3><a href='/gallery.html?name=" + galleryName  + "'>" + capitalize(galleryName) + "</a></h3>");
            });
          }
        });

    The above goes through each S3 Object (in this case the different vacation trip folders) and appends it to the HTML of the page as links to the gallery section.

    The params you give `listObjectsV2` are tricky and can be a new post in and of itself. Play around with the parameters to limit the specific objects as much as possible. You can then use regex to limit further.

2. Much of the `gallery.html` view will be directly from PhotoSwipe. The main function that makes this dynamic is below.

        var galleryName = getUrlParameter("name");
        
        var getValues1 = function(params) {
          s3.listObjectsV2(params, function(err, data) {
            if (err) {
              console.log(err, err.stack) // an error occurred
            } else {
              var thumbRegex = /(\+%).*(%29)/g;
              var imgSizeRegex = /(%28)(.*)(?=%29)/g;
              var data = JSON.parse(JSON.stringify(data))
                
              $.each( data.Contents, function (index, value) {
                var thumb = parseFilename(value.Key.replace(thumbRegex, ""))
                var imgSize = value.Key.match(imgSizeRegex, "")[0].replace(/(%28)/g,"")
        
                $('.row').append("<div class= 'col-xs-6 col-md-3'>" +
                  "<a href='https://s3.amazonaws.com/gabe-storage/" + value.Key + "' itemprop='contentUrl' data-size='" + imgSize + "' class='thumbnail'>" +
                  "<img class='thumb' src='https://s3.amazonaws.com/gabe-storage/images/" + galleryName + "/small/" + thumb + "' itemprop='thumbnail' alt='Image desc' />" +
                  "</a></div>");
              });
            };
          });
        };

     
