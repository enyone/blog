# Key material for secure things

This article contains all required information to encrypt a message (before sending to me) and to check a digital signature (signed by me).
For people who knows how things work this topic does not require further explanations.

end of TL;DR so now go and read...

If you don't understand how these things work do not try to encrypt or check anything.
You will certainly end up having only a false feeling that your message got encrypted correctly and securely
and  that way even my mother can (and will) read your "encrypted" message and starts to spread the rumour everywhere.

## OpenPGP (easier to adopt)

### On-line token (not too secure)
https://pgp.mit.edu
```
pub  2048R/70562D3F 2018-12-27 Juho Tykkala (on-line token) <juho.tykkala@phnet.fi>
```

### Off-line token (a bit more secure)
https://pgp.mit.edu
```
pub  3072R/6BF9DF93 2018-12-27 Juho Tykkala (off-line token) <enyone@rainio.org>
```

## ECC off-line token (paranoid level)
curve: brainpoolP256r1
```
-----BEGIN PUBLIC KEY-----
MFowFAYHKoZIzj0CAQYJKyQDAwIIAQEHA0IABEyh8T2iq4EoYwbVuatl3lfKW3yQ
r9bQVzTdam7Em6gzFFfpHtQCSya3SsN38HZu9F12EuWa5vmZ/86IGlLzbm8=
-----END PUBLIC KEY-----
```

## FINeID (Finland level)
https://dvv.fineid.fi
```
145927872  1005124922  3BE8FD3A  VRK Gov. CA for Citizen Certificates - G3
```
