Collage: Fight Internet Censorship with User-Generated Content
==============================================================

Collage is a new platform for constructing censorship-resistant
applications. This document gives an overview of how to build
new applications using Collage.

For general information on Collage, please see http://gtnoise.net/collage.

### What Collage Provides

Collage provides a framework for constructing censorship-resistant message
channels, by storing data inside user-generated content. Some examples of
applications you could build using Collage:

* Publish daily news articles hidden inside photos on Flickr.
* Send secret emails hidden inside your tweets on Twitter.
* Distribute [Tor](http://www.torproject.org) bridge node addresses using blog posts.

### What Collage Does Not Provide

Collage only provides tools that you can use to build these message channels.
It doesn't by itself allow for communication. Each application using Collage
must provide:

* A set of user-generated content (e.g., photos on Flickr, videos on YouTube, etc.)
* A mechanism for storing data inside this user-generated content. We provide
  some example techniques, e.g., for storing data inside JPEGs.
* A set of *tasks* that upload and download content from user-generated content hosts

This may look like a lot of work. Luckily, we include examples of these
components with Collage that you can modify for your application. Collage uses
these components to construct a custom message channel for your application.

What's Included in This Source Tree
-----------------------------------

This is a brief summary of the code tree.

* `collage/`: The core of Collage.
* `collage_apps/`: A collection of sample Collage applications.
    * `proxy/`: An application for fetching news articles stored hidden in user-generated content.
        * `taskmodules/`: Pluggable modules for fetching vectors on content hosts.
    * `vectors/`: For embedding messages inside user-generated content vectors (e.g., JPEGs, tweets, etc.).
    * `providers/`: For obtaining sets of vectors, e.g., from a directory, from generous users, etc.
    * `tasks/`: For uploading and downloading vectors to and from user-generated content hosts.
* `collage_donation/`: A system for accepting donations of user-generated content from end-users.
    * `server/`: The server backend for the donation system, which stores vectors while they are encoded with censored messages.
    * `client/`: The donation clients, which accept donations (e.g., photos) from generous users.
        * `flickr_web_client/`: A Web application that accepts photo donations from Flickr users.
        * `tk_client/`: A standalone GUI application that accepts photos from Flickr users.

Building Collage Message Channels
---------------------------------

This section describes how to build a Collage message channel. You must specify
three components:

* A `Vector`, which represents user-generated content (e.g., photos, videos, etc).
* A `VectorProvider`, which obtains `Vector`s from an external source.
* A set of `Task`s that upload and download Vectors to and from user-generated content hosts (e.g., Flickr, YouTube, etc.)

These components fit together as follows:

* One user (the "sender") composes a message and stores it inside a set of `Vector`s he obtains from a `VectorProvider`.
* The sender executes the `send` method of some `Task`s to upload his `Vector`s to a user-generated content host.
* Later, another user (the "receiver") executes the `receive` method of the same `Task`s to download the sender's `Vector`s from the content host.
* The receiver recovers the sender's message from the downloaded `Vector`s.

`Vector`, `VectorProvider` and `Task` are Python classes which I now describe
in more detail. I encourage you to also look at the source code for these
classes.

### Vector

A `Vector` represents user-generated content, e.g., a single JPEG image.  Aside
from storing vector's data (e.g., an in-memory representation of the JPEG), the
`Vector` must provide an `encode` method that hides a message inside the
`Vector` (or throws an exception if that message will not fit). It must also
provide a corresponding `decode` method to extract the hidden message.

Typically these `encode` and `decode` methods are implemented using an
information hiding algorithm like steganography or watermarking. For example,
we use [OutGuess](http://www.outguess.org) to store messages inside JPEG data.

**Important**: The `encode` method must store the message inside the vector
*without altering its visual appearence*. Otherwise, it would be easy to detect
uses of Collage and make it much easier to block.

* Examples: `collage_apps/vectors`
* What you must do: Provide a new `Vector` subclass, with `encode`, `decode` and
`is_encoded` methods.
* For more information: `collage/vectorlayer.py`

### VectorProvider

A `VectorProvider` obtains `Vector`s from somewhere. This can be from a
directory on disk, a photo sharing program such as iPhoto, from an external
source such as donations from Flickr users, or pretty much anywhere else.

Typically, when developing a Collage application you use
`DirectoryVectorProvider` to fetch `Vector`s from a local repository of
vectors. However, when you deploy your application in the real world, you will
need to get your `Vector`s from somewhere else.

* Examples: `collage_apps/providers`, specifically `local.py`.
* What you must do: Provide a new `VectorProvider` subclass, with a `_find_vector` method.
* For more information: `collage/vectorlayer.py`

### Task

A `Task` uploads and downloads `Vector`s from a user-generated content host,
e.g., Flickr. Each `Task` performs a specific action on a content host that
yield some `Vector`s, for example:

* Search for "blue flowers" on Flickr and download the top 20 resulting photos.
* Download the videos from user "JohnDoe"'s on YouTube.
* Look at trending tweets on Twitter.

Each `Task` has two key methods: `send` and `receive`. `send` uploads a
`Vector` to a user-generated content host, while `receive` downloads a set of
`Vector`s from that content host, some of which might contain Collage data.

**Important**: For each `Task`, the `Vector`s uploaded by `send` must be
accessible by `receive`.  For example, if `send` uploads pictures of "pink
dinosaurs" to Flickr, then `receive` should, e.g., perform a keyword search for
"pink dinosaurs".

To construct these tasks, we recommend you use
[Selenium](http://www.seleniumhq.org), a Web automation framework bundled with
Collage. The example in `collage_apps/proxy/taskmodules/flickr_new.py`
demonstrates how to do this.

* Examples: `collage_apps/tasks`, `collage_apps/proxy/taskmodules`
* What you must do: Provide new `Task` subclasses, with `send`, `receive` and `can_embed` methods.
* For more information: `collage/messagelayer.py` (the `Task` class, near the bottom of the file.)
