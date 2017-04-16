# Transmission Control

> Points: 250
>
> Author: Chabz
>
> We've intercepted some messages, but we've had no luck deciphering it. Can you help us?
>
> The [transmission](https://github.com/tghack/tg17hack/blob/master/crypto/transmission_control/transmission.txt)

After downloading the transmission:

    B: UmVxdWVzdGluZyBzZWN1cmUgY29ubmVjdGlvbg==
    A: SW5pdGlhdGluZyBwcm90b2NvbDogQUVTKFNIQTI1NihFQ0RIKSwgQ0JDLCBJVik=
    A: TWVzc2FnZSBmb3JtYXQ6IElWICsgQUVTKG1lc3NhZ2Up
    A: NDIwMTczNjM1MDY0LCA0NTU4NjA4NzQ4NDA1LCAxNTUxNzQ5NTIyNzE4Nw==
    A: MTM4NDczNzExOTEyOTIsIDcxNzgyMzE4NjYzNzU=
    B: MTI3NDAxMjcxMDk5OTIsIDI2NTc5OTgzNTE3NTg=
    A: MTIxMDgzMTQxMDQyNjksIDc3NTE4MzM2NzUzMjc=
    B: VYOMsCq1yqv0J+8UMe0pFaCtUi5EFlrhqKh+Y4bmR1MvoGzgSxAI6ZD8QJV0VFl9
    A: atU5UvHp5bRbUMeIiJLVxkdLaGF8tivfSKlGpOJQvLRE/74LKZ9DD42xTE6mwbmk2P7Po2Y2Ryh80aYAJY8atA==
    B: grl0/CNUkhes9mdZPsLieUC5/kWx3G0n/yEuQNbE+Ao=
    B: IvDZ++ZH9jx3LGCJzkS0TbZYz1N6siy2OD/A0xQbuLk=
    A: wXEf37wwVGHuYaoD/nULOoUN7JBA9Kq/rH6Rduj/ZW7oLbUfQcPW/0Zto5rqI0T+

we see lines ending with one or more equal signs. This is often an indicator of the data being [base64-encoded](https://en.wikipedia.org/wiki/Base64).

The transmission file can be decoded as follows:

```python
from base64 import b64decode

for line in open("transmission.txt", "r"):
    sender, message = line.split(" ")
    print("{} {}".format(sender, b64decode(message)))
``` 
 
Yielding the actual transmission data:

    B: b'Requesting secure connection'
    A: b'Initiating protocol: AES(SHA256(ECDH), CBC, IV)'
    A: b'Message format: IV + AES(message)'
    A: b'420173635064, 4558608748405, 15517495227187'
    A: b'13847371191292, 7178231866375'
    B: b'12740127109992, 2657998351758'
    A: b'12108314104269, 7751833675327'
    B: b"U\x83\x8c\xb0*\xb5\xca\xab\xf4'\xef\x141\xed)\x15\xa0\xadR.D\x16Z\xe1\xa8\xa8~c\x86\xe6GS/\xa0l\xe0K\x10\x08\xe9\x90\xfc@\x95tTY}"
    A: b'j\xd59R\xf1\xe9\xe5\xb4[P\xc7\x88\x88\x92\xd5\xc6GKha|\xb6+\xdfH\xa9F\xa4\xe2P\xbc\xb4D\xff\xbe\x0b)\x9fC\x0f\x8d\xb1LN\xa6\xc1\xb9\xa4\xd8\xfe\xcf\xa3f6G(|\xd1\xa6\x00%\x8f\x1a\xb4'
    B: b"\x82\xb9t\xfc#T\x92\x17\xac\xf6gY>\xc2\xe2y@\xb9\xfeE\xb1\xdcm'\xff!.@\xd6\xc4\xf8\n"
    B: b'"\xf0\xd9\xfb\xe6G\xf6<w,`\x89\xceD\xb4M\xb6X\xcfSz\xb2,\xb68?\xc0\xd3\x14\x1b\xb8\xb9'
    A: b'\xc1q\x1f\xdf\xbc0Ta\xeea\xaa\x03\xfeu\x0b:\x85\r\xec\x90@\xf4\xaa\xbf\xac~\x91v\xe8\xffen\xe8-\xb5\x1fA\xc3\xd6\xffFm\xa3\x9a\xea#D\xfe'

Initially, we see the initialization of the protocol and the message format before we see some strange numbers, and finally data.
In this scenario, A plays the role of the server, while B is the client.

The server initially specify the protocol to be used, namely `AES(SHA256(ECDH), CBC, IV)`. It might not be obvious, but this is the protocol specification for the AES usage in the message, which has the format `IV + AES(message)`.

Meaning each message contain it's very own IV to be used for decryption of that particular message. The protocol specify the AES key to be the SHA256 digest of the shared [ECDH key](http://andrea.corbellini.name/2015/05/30/elliptic-curve-cryptography-ecdh-and-ecdsa/)

## Breaking ECDH
After reading about [ECDH key exchange](http://andrea.corbellini.name/2015/05/30/elliptic-curve-cryptography-ecdh-and-ecdsa/), we see that the transmission is as follows:

__*a, b, p*__

__*G_x, G_y*__

__*cG_x, cG_y*__

__*dG_x, dG_y*__


Where __*a*__, __*b*__ and __*p*__ are the curve parameters, and __*p*__ must be prime, used to generate the elliptic curve by the means of the following function:

__*y^2 = x^3 + ax + b (mod p)*__

Next up, the generator base point is provided in the pair __*G_x, G_y*__.

Now, the client generate it's secret key as a random integer __*c*__ before computing and sending __*cG_x, cG_y*__.
Finally; the server does the same, only we call this key __*d*__, and the transfered data is __*dG_x, dG_y*__.

Because both client and server have their own secret key, and the communication partner's secret key multiplied with the generator __*G*__, they can calculate the __shared key__, which is calculated using the following formula:

__*shared key = c(dG) = d(cG)*__

But we do not have any knowledge of either __*c*__, nor __*d*__, meaning we can't calulate the __shared key__. Perhaps this is a poor implementation due to bad parameters?
We can easily check that the prime __*15517495227187*__ is only 44 bits, which result in the possibility of breaking the [discrete logarithm problem](https://en.wikipedia.org/wiki/Discrete_logarithm) that [Diffie-Hellman](https://wiki.openssl.org/index.php/Diffie_Hellman) relies on.

Let's try to do just that using [sagemath](http://www.sagemath.org/):

```python
# First, the three curve parameters
sage: a = 420173635064
sage: b = 4558608748405
sage: p = 15517495227187

# Create curve function
sage: E = EllipticCurve(GF(p), [a, b])

# Get the initial point
sage: G  = E(13847371191292, 7178231866375)

# Calculate the 'public key' for each
sage: cG = E(12740127109992, 2657998351758)
sage: dG = E(12108314104269, 7751833675327)

# Try to break the 'public key' to expose the secret ones.
sage: c = G.discrete_log(cG)
sage: d = G.discrete_log(dG)
sage: c
3011267900038
sage: d
545529006017

# Check that they result in the same shared key
sage: cG*d
(27356762530 : 7671609982327 : 1)
sage: dG*c
(27356762530 : 7671609982327 : 1)

# Because they are equal, this is the shared key, and we succeeded in breaking poor ECDH
sage: shared = cG*d
```

Because elliptic curves have both an X and a Y coordinates, we should try both, but X is generally the one used, so we say __*ECDH = 27356762530*__ for the rest of the writeup.

## Decrypting messages

Putting all of the above information together; we must decrypt each message by creating a new AES crypter, where the key is `SHA256(ECDH)`, the [mode of operation](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation) is CBC, and IV is the first 16 bytes of the message.

These steps are combined in the following code snippet:

```python
from base64 import b64decode
from Crypto.Cipher import AES
from Crypto.Hash import SHA256

messages = [
    "VYOMsCq1yqv0J+8UMe0pFaCtUi5EFlrhqKh+Y4bmR1MvoGzgSxAI6ZD8QJV0VFl9",
    "atU5UvHp5bRbUMeIiJLVxkdLaGF8tivfSKlGpOJQvLRE/74LKZ9DD42xTE6mwbmk2P7Po2Y2Ryh80aYAJY8atA==",
    "grl0/CNUkhes9mdZPsLieUC5/kWx3G0n/yEuQNbE+Ao=",
    "IvDZ++ZH9jx3LGCJzkS0TbZYz1N6siy2OD/A0xQbuLk=",
    "wXEf37wwVGHuYaoD/nULOoUN7JBA9Kq/rH6Rduj/ZW7oLbUfQcPW/0Zto5rqI0T+"
]

secret = "27356762530"
hasher = SHA256.new()
hasher.update(secret)
key = hasher.digest()

def decrypt(message):
    iv = message[:16]
    encrypted = message[16:]

    decrypter = AES.new(key, AES.MODE_CBC, iv)
    return decrypter.decrypt(encrypted)

for m in messages:
    print(decrypt(b64decode(m)))

```

Executing this yield the following conversation:

    Send me flag plz
    Here you go: TG17{large_primes_are_nice}
    Thanks!									
    End connection
    Connection closed
    
And the flag is __`TG17{large_primes_are_nice}`__.
