# MSC2448: Using BlurHash in Media Events

[BlurHash](https://blurha.sh) is a compact representation of a placeholder
for an image (or the frame of a video). Currently in Matrix, clients must
display a placeholder image in the message timeline while a piece of media is
loading. Some clients, such as Riot, simply display an empty space.

While thumbnails exist to combat this to some degree, they still need to be
downloaded from a homeserver, which is not instantaneous.

Instead, a BlurHash can be sent inside the `m.room.message` event, which upon
receipt other clients can render for a pretty preview while the actual
thumbnail downloads. They also do not contain any `"` characters, making them
simple to stick inside existing JSON blobs.

To be clear: A BlurHash does not replace a thumbnail - it will be shown
before the thumbnail is downloaded.

## Proposal

### m.room.message

A new optional field is added in `m.room.message`'s content field called
`blurhash`. It is a BlurHash of the original piece of media. Clients could
then render this using [one of the available BlurHash
implementations](https://github.com/woltapp/blurhash).

This would be optionally displayed while the thumbnail of the media is loaded
in parallel.

Example `m.room.message` content field:

```
{
  "body": "image.png",
  "info": {
    "size": 149234,
    "mimetype": "image/png",
    "thumbnail_info": {
      "w": 301,
      "h": 193,
      "mimetype": "image/png",
      "size": 72958
    },
    "w": 301,
    "h": 193,
    "thumbnail_url": "mxc://example.org/abcdefg",
    "blurhash": "JadR*.7kCMdnj"
  },
  "msgtype": "m.image",
  "url": "mxc://example.org/abcde",
}
```

Note that a BlurHash representation is really only applicable to media, and
as such should only be used in conjunction with the following
`m.room.message` msgtypes:

* `m.image`
* `m.video`

### Inline images

An optional attribute is added to `<img>` tags in messages:
`data-mx-blurhash`, where the value of the attribute is the blurhash
representation of the inline image.

This would be optionally displayed while the inline image itself is loaded in
parallel.

Example `m.room.message.formatted_body`:

```
"formatted_body": This is awesome <img alt=\"flutterjoy\" title=\"flutterjoy\" height=\"32\" src=\"mxc://matrix.example.org/abc\" data-mx-blurhash=\"LEHV6nWB2yk8pyo\" />
```

## Calculating a blurhash

BlurHashes are inserted into `m.room.message` events by the client, however
some clients may not be able to implement the BlurHash library for whatever
reason. In this case, it would be nice to allow the media repository to
calculate the BlurHash of a piece of media for the client, similar to how
thumbnails are calculated by media repositories today.

The
[`/_matrix/media/r0/upload`](https://matrix.org/docs/spec/client_server/r0.6.0#post-matrix-media-r0-upload)
endpoint response is modified to include an optional `blurhash` key,
which the client may use to insert into messages if desired:

Example response:

```
{
  "content_uri": "mxc://example.com/abcde123",
  "blurhash": "LKO2?U%2Tw=w]~RB"
}
```

A server MUST also be able to render a BlurHash image from a BlurHash string.
This is to support clients that do not have a BlurHash client implementation
or are otherwise unable to do so themselves.

This takes the form of a new endpoint, `GET
/_matrix/media/r0/blurhash/{encodedBlurHash}`. This endpoint supports the following query parameters:

* `width` - The width in pixels of the returned BlurHash image
* `height` - The height in pixels of the returned BlurHash image

Note that implementations should be careful to limit the size of accepted
BlurHash strings, as not to overload the server with processing an obscenely
long string. If this is the case, the server should return a `400
M_TOO_LARGE`.

Example request:

```
GET /_matrix/media/r0/blurhash/LG.F5%5D%3B%2BYk%5E6%23%25*%2B%2CK-%3A9%3D%3F%40%5B
```

Example response:

```
<bytes representing BlurHash image>
```

In addition, the server can return the BlurHash string for an image when
given an MXC URL. This would be through something like the Media Information
API (specified in
[MSC2380](https://github.com/matrix-org/matrix-doc/pull/2380)), or similar.

## Visualisation

Viewing an image message that is loading:

![blurhashed preview](images/2448-blurhash.png)

Once the image loads:

![the image has loaded](images/2448-loaded-image.png)

For reference, the current state of things in Riot is:

![boo, sad](images/2448-current-state.png)

## Alternatives

We could include a base64 thumbnail of the image in the event, but blurhash
produces much more efficient textual representations.

## Backwards compatibility

Older clients would ignore the new `blurhash` parameter.

Newer clients would only show it if it exists.

## Unstable prefixes

Implementations wishing to add this before this MSC is merged can do so with
the following:

The `blurhash` key in `m.room.message` should be replaced with
`xyz.amorgan.blurhash`.

`/_matrix/media/r0/upload` should return a `xyz.amorgan.blurhash` key
containing the blurhash instead of `blurhash`. This is preferred to adding
another endpoint, which is messier to later remove.

`/_matrix/media/r0/blurhash/{serverName}/{mediaId}` should be
replaced with
`/_matrix/media/unstable/xyz.amorgan/blurhash/{serverName}/{mediaId}`.

The `data-mx-blurhash` attribute in `<img>` tags should be replaced with
`data-xyz-amorgan-blurhash`.

And finally, an entry should be added to the homeserver's `GET
/_matrix/client/versions` endpoint, in `unstable_features`, with the key
`xyz.amorgan.blurhash` set to `true`.

## Security considerations

Blurhash entries in encrypted events, be it as part of the `info` property,
or `<img>` tags, should be encrypted along with the rest of the event
content.

## Links

BlurHash's algorithm description can be found
[here](https://github.com/woltapp/blurhash/blob/master/Algorithm.md), which
also includes the full output character set.