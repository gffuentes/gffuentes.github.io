---
layout: post
title:  "S3 Photo Album"
date:   2016-10-15 21:39:00 -0500
categories: static
image: photo-album.png
---
## Background

After many vacations of borrowing my partner's aunt's DSLR, we had put together some pretty nice albums of fancy photos. I did not have a place to publicly store these photos, so I did some research into online options like SmugMug and Flickr. These options seemed like overkill or did not meet my main goals. 

But what were my main goals?

- Cheap
- No tiered pricing (per GB pricing models)
- Easily accesible by family and friends
- Organized by album for easy navigation
- Backup images in their full resolution

After finding a JS based photo gallery library (more to come on that later), I created a Sinatra based application served on an EC2 instance. After learning about static web pages hosted on S3, I thought I could do better (and even cheaper!). Below I share how I was able to create a photo album in S3.

## Application

**Ingredients:**

1. AWS S3 for storage and static website hosting
2. Twitter Bootstrap for style
3. [PhotoSwipe JS Library](http://photoswipe.com/)  for gallery
4. [ImageMagick](http://www.imagemagick.org/script/index.php) for thumbnail creation via [rmagick](https://github.com/rmagick/rmagick)

**Setting up S3 Buckets:**

For more detailed instructions, especially on how to setup Route 53 to host your custom domain, see [Setting Up a Static S3 Site](http://docs.aws.amazon.com/AmazonS3/latest/dev/website-hosting-custom-domain-walkthrough.html)

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
        
        // From Setting Up a Static S3 Site
        
        {
	        "Version": "2012-10-17",
	        "Statement": [
		        {
		    	    "Sid": "AddPerm",
		    	    "Effect": "Allow",
		    	    "Principal": "*",
		    	    "Action": "s3:GetObject",
		    	    "Resource": "arn:aws:s3:::on-vacation.gabefuentes.com/*" // Should be replaced with your bucket name
		        }
	        ]
        }

5. Configure website bucket to enable website hosting in `Properties > Static Website Hosting > Enable website hosting`. Type your index document in the open field - in my case, `home.html`.

6. Your S3 static website should now be live! In S3, under `Properties > Static Website Hosting` you will be able to see the `amazonaws.com` URL. Confirm that it is up, and you can reroute your own hostname in Route 53 if you so choose.

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
        var largePath = 'images/' + galleryName + '/';
        var thumbPath = 'images/small/' + galleryName + '/';
        var params = { Bucket: 'gabe-storage', Prefix: largePath, Delimiter: '/', EncodingType: 'url', StartAfter: largePath };
        
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

    The previous `home.html` gallery links lead to `/gallery.html?name=gallery+name`, with whatever vacation gallery you are visiting replacing `gallery+name`. On the gallery side, you then use this url parameter (with a nifty `getUrlParameter` function I found at StackOverflow linked [here](http://stackoverflow.com/questions/19491336/get-url-parameter-jquery-or-how-to-get-query-string-values-in-js)) to search S3 with the same `s3.listObjectsV2` function used in the home screen.

    Originally, I did not have thumbnails - I just had the gallery load up the full resolution images in a smaller image size. This meant the gallery was extremely slow to load. To solve this, I created thumbnails as well (the `/small` folder nested within the storage bucket - more on resizing later). As part of the process of converting the full resolution images into thumbnails, I also added the full image size to the file name. I can then use those image sizes to display the zoomed in image as part of PhotoSwipe.

    So, to wrap things up, I use the URL parameter to search through S3 for the right vacation. Once there, I display the thumbnail versions of each image and also give photoswipe the full resolution version, as well as the image size. To make like easier, the thumbnail file names and full resolution file names are almost identical except the image size addition for the full rez.

**Image Resizing**

I created the image resizing aspect as part of the original Sinatra application, so I kept the ruby script for image manipulation and manually uploaded the images to S3 in their corresponding folders. However, this could easily be updated to hit the AWS API and upload them automatically.

If you have not used ImageMagick before, it is a really great command line tool for manipulating images in all sorts of ways - my three uses of it barely scratch the surface. Learn more  [here](http://www.imagemagick.org/script/index.php)!

I started off with the photo album saved locally. The script iterates through each image and performs the following using the ruby interface for ImageMagick called `rmagick`.

1. To resize the images, I scaled them to 10% their original size.

        img.scale(0.1)

2. I had a lot of trouble understanding why PhotoSwipe was displaying images in the wrong rotation only sometimes. What I figured out was that when a digital camera takes a picture, it stores the orientation of the camera as part of the metadata called EXIF profile. By `auto_orient`ing images to 'TopLeftOrientation', they will all display upright. More info [here](http://www.imagemagick.org/script/command-line-options.php#auto-orient). ImageMagick makes reading the current orientation and converting if necessary easy!

        if img.orientation.to_s != 'TopLeftOrientation'
          img.auto_orient!
          img.write(i)
        end

3. Getting the image size and renaming the file is straightforward.

        width,height = FastImage.size(i) unless i.include?("(")
        File.rename(i, i.downcase.sub('.jpg', " (#{width}x#{height}).jpg")) unless i.include?("(")

**Conclusion & After Thoughts**

Overall, this was a good "beach project" to dive into. I had some great take aways on PhotoSwipe and ImageMagick. These are great tools to add to my repertoire. You can view the full source code [here](https://github.com/gffuentes/static-s3-photoalbum) for more details.

As always, there are things that can be improved on! See below:

- There is likely a better way to get the URL parameters using JS or jQuery - the solution I took seems quick and dirty, but works.

- The image size that is added to the full resolution images could be in a better format for easier parsing

- Image resizing can be automated using Amazon's Lambda! Funny enough, they have an [example](http://docs.aws.amazon.com/lambda/latest/dg/with-s3-example.html) on how to do this. I did not see this until after my implementation.
