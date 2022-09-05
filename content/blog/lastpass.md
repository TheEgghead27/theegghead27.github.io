---
title: 'Migrating TOTP keys from LastPass Authenticator to FreeOTP'
description: "LastPass Authenticator added a QR code export feature, but it's restricted to its own app. Let's break it!"

---

# Migrating TOTP keys from LastPass Authenticator to FreeOTP

For the longest time, LastPass Authenticator never had an option for exporting keys, so I was delighted to see a QR code exporter tool added in the latest release.

![The latest patch notes for LastPass Authenticator. It mentions "a QR scan feature to allow users to move TOTP codes from one device to another more easily"](/assets/blog/lastpass/patch-notes.jpg)

However, I soon realized that there wasn't a standard token-by-token export function that I could scan into a new TOTP app, since there was only one huge QR code that would only work with... another phone running LastPass Authenticator :/

Since it was so big, I wondered if it was simply encoding the tokens in the QR code. After discovering [zbarimg](https://sourceforge.net/projects/zbar/) from an [AskUbuntu post](https://askubuntu.com/questions/22871/software-to-read-a-qr-code) and scanning the QR code, my suspicions were confirmed: there was a proprietary URI with what looked to be Base64 encoded data.

`lpaauth-migration://offline?data=H4sIAAAAAAAAAK2...`

I then realized that some of the characters were [percent encoded](https://en.wikipedia.org/wiki/Percent-encoding), so `base64 -d` wasn't having it.

A little Google-fu brought me to a convenient [StackExchange answer](https://unix.stackexchange.com/a/159254) with a python one-liner I could use to decode the URL, but when I ran the stripped data through `base64 -d`, all I got was garbage!

![Unrecognized symbols are displayed on a terminal](/assets/blog/lastpass/garbage.jpg)
This doesn't help!

Looking further into it, I had a vague feeling that I had seen that starting sequence of `H4s` and 9 `A`s before, so I looked that up, and found this:
> This may be a long shot, but if anyone ever searches for this string, I know exactly what you are looking at – It’s a base64 encoded zipped string.

["H4sIAAA What’s so important about this string?"](https://blog.dotnetframework.org/2016/12/07/h4siaaa-whats-so-important-about-this-string/) - [Infinite Loop Development Ltd](https://blog.dotnetframework.org/author/dananos/)

Bingo! The data was gzipped, and all I had to do was `gunzip` it, giving me a JSON with my precious 2FA keys.

After running it through `python -m json.tool` to unminify it, I was left with this unit of a shell one-liner:
```sh
zbarimg lastpass-qr.png | python3 -c "import sys, urllib.parse as ul; print (ul.unquote_plus(sys.stdin.read()))" | tail -c +42 | base64 -d | gunzip | python -m json.tool > 2fa.json
```

The JSON keys were minified too, but with a little guesswork based on the key abbreviations and content, I figured out enough of the format to make a [script](https://gist.github.com/TheEgghead27/f00cc83b0f8a28c43c15b9727fcf6383) that generates `otpauth://` URIs that I could import into FreeOTP, with everything working just as expected!

~~moral of the story: lastpass please give us an official way to export our 2fa secrets~~
