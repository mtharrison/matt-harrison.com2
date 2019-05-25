---
title: "BSprites:  Combined web assets using Typed Arrays and Data URIs"
date: 2014-07-28T00:00:00+01:00
---
**Disclaimer:** This was a weird idea I had one day and put this together the same evening. I've not tested it cross-browser or in a production environment. I've not benchmarked this either vs actually downloading all the images. It's kind of a 'what if' project at the moment. If you think it's really dumb or cool, I'd be really interested to hear your thoughts.

---

Generally, whenever a browser loads a new image, it will make a new HTTP request to the server. This doesn't always mean establishing a new TCP connection, because browsers will leave TCP connections open for reuse if they're to the same hostname. However, these connections are managed by a 'connection pool'. Most browsers do not do [HTTP pipelining](http://en.wikipedia.org/wiki/HTTP_pipelining), which means the connection waits for a response before sending a new request. This means your requests are queued. If you need to load 300 images from the same hostname this could have serious implications for your user experience.

## CSS sprites

A strategy that people frequently use to mitigate this is to _visually_ combine multiple images into a single image, which can then be served with a single HTTP request. This requires someone to lay out the images onto a larger canvas, at known pixel coordinates. The image is then positioned using CSS in the browser to reveal the individual images.

![CSS Sprite](https://mattharrison.s3.amazonaws.com/july2014/sprite.png)

The drawbacks of this method are fairly obvious:

- Image metadata is opaque: Requires the consumer of the images to know at which coordinates to find the individual images, there's no way to serve these to some consumer you may not know, unless you also provide the metadata separately
- You need to make another image file for every combination of images you wish to serve
- If you need to add a new image, you have to recreate the sprite and update any metadata
- All images within a sprite must be of the same content type e.g. `image/png`, `image/jpg` etc
- Creating the images can be cumbersome and open to human error.

## Using data URIs

Browsers have moved on quite a lot in the time the CSS sprite hack was established. There are 2 new features that open up other possibilities.

### Data URIs

[Data URIs](http://en.wikipedia.org/wiki/Data_URI_scheme) are a way to embed inline a resource as a base64 encoded string of the actual bytes that make up that resource. A data URI has the following form:

	data:[<MIME-type>][;charset=<encoding>][;base64],<data>

Data URIs can take the place of a URL in the `src` attribute of an image. Here's an example which is a small red dot:

	<img src="data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAUA
	AAAFCAYAAACNbyblAAAAHElEQVQI12P4//8/w38GIAXDIBKE0DHxgljNBAAO
	9TXL0Y4OHwAAAABJRU5ErkJggg==" alt="Red dot" />

Data URIs can also be used in CSS:

	ul.checklist li.complete {
	    padding-left: 20px;
	    background: white url('data:image/png;base64,iVBORw0KGgoAA
	AANSUhEUgAAABAAAAAQAQMAAAAlPW0iAAAABlBMVEUAAAD///+l2Z/dAAAAM0l
	EQVR4nGP4/5/h/1+G/58ZDrAz3D/McH8yw83NDDeNGe4Ug9C9zwz3gVLMDA/A6
	P9/AFGGFyjOXZtQAAAAAElFTkSuQmCC') no-repeat scroll left top;
	}

This means if performance is critical to you, you can transfer a web page along with all images in a single HTTP request by including them inline in the HTML. Or with an extra request you can include them in the CSS.

*Examples from [Wikipedia](http://en.wikipedia.org/wiki/Data_URI_scheme)*

**Disadvantages to using data URIs**

There are however some disadvantages to the above:

- If the HTML/CSS is modified and invalidated in the browser cache, all your images need to be downloaded again even if they themselves haven't been modified
- Encoding as Base64 takes up more space than the original binary data, meaning your image data is (~33%) heavier on the wire than if you just sent the image as binary data

### ArrayBuffer

Javascript is traditionally unsuited to dealing with binary data, but modern browsers have `ArrayBuffer` which is a data type that points to a fixed size raw allocation of bytes in memory. The `ArrayBuffer` type can't be manipulated directly in Javascript, but you can create a 'view' on a `ArrayBuffer` with Typed Arrays which can then be used to read/write data to the `arraybuffer`.

Typed arrays are like normal javascript arrays, except every element in the array is a fixed capacity number and they also have a `buffer` property that points to the underlying `arraybuffer`. The array elements will overflow, so if you have a `Uint8Array` with an element set to `255` and you add `1` to that element, it (and the underlying buffer) will overflow to `0`.

In the example below, an `ArrayBuffer` is instantiated with 2 bytes of memory. All the bits are set to 0 by default.

When creating a `Uint8Array` from this buffer, we get an array, whose `buffer` property points at our original buffer and contains the elements `[0,0]`.

![Typed arrays](https://mattharrison.s3.amazonaws.com/july2014/array-buffer.png)

The `XMLHttpRequest` object in Javascript has a `responseType` property. This can be set to `arraybuffer` and will bypass any interpretation of the received data, instead just passing an `arraybuffer` object directly to the `onload` callback.

## A new approach to sprites

The disadvantages of CSS Sprites and embedding images with data URIs inside HTML and CSS can be overcome by:

1. Serving the images separately to any HTML/CSS
2. Serving binary data rather than base64 encoded data
3. Serving the combined resource dynamically based on input parameters

### Server side

- Create a webservice that receives a request for a set of images e.g. http://service.com/images?img1.jpg&img2.jpg
- Have the server sequentially read the raw bytes from each image file into a buffer
- When reading each image, maintain a metadata data structure which keeps track of the byte positions of each image inside the buffer along with mime type and filename
- Combine the metadata and all of the image data into a single binary buffer
- Respond to a request with content-type `application/octet-stream` and the entire buffer as the response body
- Also include a header `X-Metadata-Length` which tells the client how long the metadata header is in bytes

![Server side](https://mattharrison.s3.amazonaws.com/july2014/server-side.png)

### Client side

- Request the data from the given URL, and specify the `xhr.responseType` as `arraybuffer`

		var request = new XMLHttpRequest();
		request.responseType = 'arraybuffer';
		request.open('GET', 'http://service.com/images?img1.jpg&img2.jpg', true);
- According to the value of the `X-Metadata-Offset` take a `slice()` from the buffer at the corresponding offset
- Interpret the metadata chunk as a UTF-8 string and then parse as JSON.
- Use the `slice()` method on the rest of the arraybuffer to create subarrays corresponding to each image
7. Convert the array buffers to base64 strings
8. Create the `Image` DOM elements and set their Data URIs

## Example Implementations

#### Client side

[github.com/mtharrison/bsprite-client](http://github.com/mtharrison/bsprite-client) - A simple browser module for requesting bsprites from a compliant server.

#### Server side

[github.com/mtharrison/bsprite-go](http://github.com/mtharrison/bsprite-go) - A go package for creating and serving bsprites using `net/http`.

Node module coming soon too!
