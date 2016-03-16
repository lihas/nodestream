# Nodestream

[![NPM Version][npm-badge]][npm-url]
[![Build Status][travis-badge]][travis-url]
[![Coverage Status][coveralls-badge]][coveralls-url]
[![Documentation Status][inch-badge]][inch-url]
![Built with GNU Make][make-badge]

> Streaming library for binary data transfers


[npm-badge]: https://badge.fury.io/js/nodestream.svg
[npm-url]: https://npmjs.org/package/nodestream
[travis-badge]: https://travis-ci.org/nodestream/nodestream.svg
[travis-url]: https://travis-ci.org/nodestream/nodestream
[coveralls-badge]: https://img.shields.io/coveralls/nodestream/nodestream.svg
[coveralls-url]: https://coveralls.io/r/nodestream/nodestream
[inch-badge]: http://inch-ci.org/github/nodestream/nodestream.svg
[inch-url]: http://inch-ci.org/github/nodestream/nodestream
[make-badge]: https://img.shields.io/badge/built%20with-GNU%20Make-brightgreen.svg

## Description

This library aims to provide an abstraction layer between your application/library and all the various remote storage services which exist on the market, either as hosted by 3rd parties or self-hosted (S3, GridFS, Azure Blob Store, etc.). Your code should not depend on these services directly - the code responsible for uploading a file should remain the same no matter which storage service you decide to use. The only thing that can change is the configuration.

## Usage

### Installation

The first step is to install nodestream into your project:

`npm install --save nodestream`

The next thing is to decide which *adapter* you want to use. An adapter is an interface for nodestream to be able to interact with a particular storage system. Let's use local filesystem for a start:

`npm install --save nodestream-filesystem`

### Configuration

Let's create and configure a nodestream instance with which your application can then interact:

```js
// Require the main Nodestream class
const Nodestream = require('nodestream')
const path = require('path')
const nodestream = new Nodestream({
  // This tells nodestream which storage system it should interact with
  adapter: require('nodestream-filesystem'),
  // This object is always specific to your adapter of choice - always check
  // the documentation for that adapter for available options
  config: {
    root: path.join(__dirname, '.storage')
  }
})

```

Great! At this point, nodestream is ready to transfer some bytes!

### Usage

#### Uploading

You can upload any kind of readable stream. Nodestream does not care where that stream comes from, whether it's an http upload or a file from your filesystem or something totally different.

For this example, we will upload a file from our filesystem.

> We will be uploading the file to our local filesystem as well as reading it from the same filesystem. Normally you would probably use a source different from the target storage, but Nodestream does not really care.

```js
const fs = require('fs')
// This is the file we will upload - create a readable stream of that file
const profilePic = fs.createReadStream('/users/me/pictures/awesome-pic.png')

nodestream.upload(profilePic, {
  // directory and name are supported by all storage adapters, but each
  // adapter might have additional options you can use
  directory: 'avatars',
  name: 'user-123.png'
})
.then(results => {
  // results can contain several properties, but the most interesting
  // and always-present is `location` - you should definitely save this
  // somewhere, you will need it to retrieve this file later!
  console.log(results.location)
})
.catch(err => {
  // U-oh, something blew up 😱
})
```

Congratulations, you just uploaded your first file!

#### Downloading

Downloading a file is quite straight-forward - all you need is the file's location as returned by the `upload()` method. You will always get back a readable stream which you can then `.pipe()` to something. Again, Nodestream does not care where you are sending the bytes, be it local filesystem, an http response or even a different Nodestream instance (ie. S3 to GridFS transfer).

```js
// We are hardcoding the location here, but you will probably want to
// retrieve the file's location from a database
const file = nodestream.download('avatars/user-123.png')

// Let's create a destination for the download
const fs = require('fs')
const destination = fs.createWriteStream('/users/me/downloads/picture.png')

// Since file is a readable stream, we can pipe it to any other writable
// stream we like
file.pipe(destination)
file.once('end', () => console.log('all bytes sent to destination!'))
```

#### Removing

Just pass the file's location to the `.remove()` method.

```js
nodestream.remove('avatars/user-123.png')
.then(location => {
  // The file at this location has just been removed!
})
.catch(err => {
  // Oh no!
})
```

### Transforms

Nodestream supports a feature called transforms. In principle, a transform is just a function that modifies the incoming or outgoing bytes in some way, transparently for each stream passing through Nodestream. Some use cases:

- Calculating checksums
- Compressing/decompressing data
- Modifying the data completely, ie. appending headers/footers and whatnot

#### Registering a transform

When you configure your Nodestream instance, you should register transforms using the `.addTransform()` function.

```js
const compress = require('nodestream-compress-transform')

// The first argument defines when the transform should be applied ('upload', 'download')
// The second argument is the actual implmentation and the third argument is
// an option configuration object which will be passed as-is to the transform stream
// when the time comes to use the transform.
nodestream.addTransform('upload', compress, { mode: 'compress' })
nodestream.addTransform('download', compress, { mode: 'decompress' })
```

Now, every time you call `.upload()` or `.download()`, the respective transform will be applied on the stream.

For uploads, a transform can optionally publish some data about the applied transformations onto the `results` object.

There is no limit to the amount of transforms which can be registered per Nodestream instance.

## License

This software is licensed under the **BSD-3-Clause License**. See the [LICENSE](LICENSE) file for more information.
