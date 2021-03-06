---
layout: post
title: "Encrypted Blog Posts"
date: 2019-02-14
categories: blog 
tags: coding crypto
tagline: "Using good ol' PGP to make password-protected posts on gh-pages."
---
SHHHHH SECRETS! Github pages is one of my favorite services. Fast, free, open web hosting?! Dream come true. Brings back nostalgic echo-feels of [GeoCities](https://en.wikipedia.org/wiki/Yahoo!_GeoCities) back in the day. But there is one thing missing. Private posts. I'm not a public person; I can also be a tad disorganized. Through the years I've tried a million ways of keeping track of notes journal entries and idle thoughts -- google docs, google notes, evernote, emails to myself, physical notebooks and journals -- things which I don't want to share with the world, and noise I'm sure the world doesn't want. What if I could just store everything on my blog, yet still have it be only available to me? The answer? PGP encryption.

[Click here for a demo]({{ site.baseurl }}{% post_url 2019-02-12-encrypt-test %}). The password is "test".

Rusty as I am with encryption, [PGP](https://users.ece.cmu.edu/~adrian/630-f04/PGP-intro.html), [node.js async syntax](https://thecodebarbarian.com/80-20-guide-to-async-await-in-node.js.html), and jekyll/liquid, pulling it all together took me a couple nights. But now I can encrypt posts on my PC using PGP symmetric key encryption, post them on the blog, and decrypt them right in the browser! Public yet reasonably secure, assuming the password is decent. Plus it gives flexibility!

Say I want to share a post with a few friends, but don't want anyone else to read it (I could email them, sure, but I'd prefer to have all my thoughts in one place, my website/blog). Well, I could encrypt that post using a unique one-time passphrase, which I could then give to my friends. Now they can decrypt the one post and read it! 

**Disclaimer:** I am using a javascript library, [OpenPGP-JS](https://github.com/openpgpjs/openpgpjs) to decrypt the PGP message in-browser. There are [good reasons this is potentially bad](https://www.nccgroup.trust/us/about-us/newsroom-and-events/blog/2011/august/javascript-cryptography-considered-harmful/). 
Of course, posting something publicly, even encrypted, is potentially an awful idea.
If someone cracks the key they can easily read the post, PGP uses a random IV (initialization vector) to prevent easy cracking of a reused symmetric key, but even so it scares me a bit.
Anway, I hope this is helpful to someone, but use at your own risk. 
I was careful to follow best practices as near as I can tell, but if I really screwed something up, please let me know!

---

## Cookbook

Now for the how-to bit.

### Encryption
PGP, which stands for Pretty Good Privacy, is widely used for encrypting emails. But it's a proven model which can be used to encrypt pretty much any document. Since I'm on Linux, I'm using [GPG](https://www.gnupg.org/gph/en/manual/x110.html), the GNU PGP implementation. 
Since I want to be able to share the password with friends and decrypt the post in-browser without sharing my private key, I will use [symmetric encryption](https://www.tutonics.com/2012/11/gpg-encryption-guide-part-4-symmetric.html). The recommendation seems to be to use AES256 as the cipher, which can be done with the below command to encrypt `file.txt`:

```terminal
gpg --symmetric --armor --cipher-algo AES256 file.txt
```
Or, if you're in a hurry:
```terminal
echo "hello world" | gpg -ca --cipher-algo AES256
```

The `--armor` option tells it to produce [base64](https://en.wikipedia.org/wiki/Base64)-encoded text instead of binary, so it can be placed into a text-based blog post. It will ask you to enter a password and confirm, then print the PGP message onto the console. 

That's it for encryption! We'll come back to the process of getting it into a blog later, but basically you can just copy-paste. Decrypting it in-browser is the only hard part.

### Decryption

Decrypting with `gpg` is easy:
```terminal
gpg -d cyphertext.txt
```

Slightly harder is decrypting in-browser.
Looking around, there were a few JS implementations of PGP. 
I chose to use [OpenPGPjs](https://github.com/openpgpjs/openpgpjs), which is developed by [ProtonMail](https://protonmail.com/blog/openpgpjs-email-encryption/). 
I picked this one because it had symmetric key support, was audited, and ProtonMail is reasonably well-known.
The docs for symmetric key decryption were... lacking , but the code is quite clean:

```javascript
// use openpgpjs to decrypt a pgp symmetric key-encrypted blogpost
var openpgp = window.openpgp;
openpgp.initWorker({ path:'/js/openpgp.worker.min.js' });

// decrypted successfully! handle in webpage, jquery etc
function handleDecryptSuccess(plaintext) {
  $('#plainText p').html(plaintext.data);
}

// decryption failed. handle error.
function handleDecryptError(error) {
  console.log(error);
  $('#errorMsg').removeClass('d-none');
}

// OpenPGPjs call to decrypt PGP msg
async function symDecrypt(encrypted, pw) {
  var options = {
    message: await openpgp.message.readArmored(encrypted),
    passwords: [pw],
  };

  openpgp.decrypt(options)
    .then(handleDecryptSuccess, handleDecryptError);
}

// ---------------------------------------
// grab the PGP message from the blog post
var encrypted = $('#encryptedText pre').html();
// grab password, e.g., from form input
var pw = $('#pwInput').val();

// decrypt msg using password!
symDecrypt(encrypted, pw);
```

### Jekyll
My, admittedly manual, method of putting this into a blog post with jekyll is then to copy the cipher into a blog template, which I've prepared to look like this: 

````
---
layout: encrypted
title: "Test Post"
date: 2019-02-14
categories: secret
---
```
-----BEGIN PGP MESSAGE-----
Version: GnuPG v1

jA0EBwMCPGpXuDiLLVhg0kEBrhOpWgup+Vxf2Oxq0X4UORRbpgIMEKSlklLiSovO
K6kz0nXrAJe1HbOBjAIXRb2xcJT5QSlPCJyaFIq70nVubQ==
=8k8s
-----END PGP MESSAGE-----
```
````
This way, you can have a special layout for `encrypted` posts, and just include just the ciphertext in the <code>&#123;&#123; content &#125;&#125;</code> tag as normal.

