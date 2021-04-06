Binary Modular Crypt Format
===========================

Binary Modular Crypt Format (BMCF) is the binary counterpart to Modular Crypt Format (MCF). MCF is a de facto standard for encoding password hashes in an ASCII string, and it allows hashes generated with different hash algorithms to be stored alongside one another compatibly. MCF is not formally defined, but [it has been documented][mcf] by the authors of the [Passlib password hashing library][passlib] for Python.

Modular Crypt Format originated with the [crypt(3)][crypt] Unix C library function, and has since been implemented in several other libraries. Historically, MCF was used to encode the password hashes of operating system users for storage in a text file. Nowadays, MCF is also used in systems which store password hashes in database management systems. On large systems with many users, it is often desirable to maximize storage efficiency. To that end, I present Binary Modular Crypt Format as a compact binary serialization of MCF hashes. Depending on the hash scheme, BMCF reduces size by approximately 25–33%.

[mcf]:      https://passlib.readthedocs.io/en/stable/modular_crypt_format.html  "Passlib's Documentation of MCF"
[passlib]:  https://passlib.readthedocs.io/en/stable/                           "Passlib password hashing library for Python"
[crypt]:    https://en.wikipedia.org/wiki/Crypt_(C)                             "crypt(3) Unix C library function"



### Terminology

- **MCF:** Modular Crypt Format. A textual notation for representing password hashes.
- **BMCF:** Binary Modular Crypt Format, Binary MCF. A compact binary representation of password hashes.
- **Scheme:** Password hash scheme. A precise procedure by which a cryptographic hash function is employed to hash a password and represent it in MCF.
- **Hash:** A complete MCF or BMCF hash.
- **Field:** Fields are sections of an MCF hash delimited by <code>$</code>. Fields may contain more than one component if the leading components have a fixed width.
- **Identifier:** Hash scheme identifier. The first field of all MCF hashes, or its binary counterpart.
- **Digest:** The output of a hash function. A vital component of a complete hash.
- **BMCF Definition:** A specification for translating MCF hashes to BMCF hashes for a particular hash scheme.
- **Decode:** The process of converting an MCF hash to BMCF.
- **Encode:** The process of converting a BMCF hash to MCF.



MCF Example
-----------

A typical MCF hash has several components and looks like this:

    $2y$14$i5btSOiulHhaPHPbgNUGdObga/GC.AVG/y5HHY1ra7L0C9dpCaw8u

MCF hashes start with <code>$</code>, which is also used to delimit fields. The field following the first <code>$</code> identifies the hash scheme that was used. Beyond that, field definitions vary by hash scheme, but generally include algorithm parameters, a salt, and the actual digest.

The example above is a [Bcrypt][bcrypt] hash, identified by the <code>2y</code> scheme *identifier*. The next field, <code>14</code>, is the Bcrypt *cost* parameter. The first 22 characters of the final field represent the 128-bit *salt*, <code>i5btSOiulHhaPHPbgNUGdO</code>. The remaining 31 characters represent the 184-bit Bcrypt hash *digest*, <code>bga/GC.AVG/y5HHY1ra7L0C9dpCaw8u</code>. The salt and digest are base-64 encoded using a non-standard alphabet.

Note that while Bcrypt puts the salt and digest in the same field, other schemes may use separate fields, usually to accommodate variable-length salts.

Hash sizes vary by algorithm and hash scheme. A Bcrypt hash, as in the example above, is 60 characters in MCF, which of course requires 60 bytes of storage in any single-byte character encoding. A Bcrypt hash in BMCF requires 40 bytes: <code>(128+184)/8 + 1</code>.

[bcrypt]: https://en.wikipedia.org/wiki/Bcrypt "Bcrypt, Blowfish-based password hashing"



General Specification
---------------------

*So far, BMCF is only fully defined for Bcrypt. BMCF definitions for additional algorithms can be added as needed.*

Binary MCF hashes start with a header octet. This first byte employs its three most significant bits for the scheme identifier as follows:

    0x00 [Reserved; perhaps for the pre-MCF implicit DES schemes]
    0x20 $2$    Bcrypt
    0x40 $2a$   Bcrypt
    0x60 $2x$   Bcrypt
    0x80 $2y$   Bcrypt
    0xA0 $5$    SHA-256 Crypt (not yet fully defined)
    0xC0 $6$    SHA-512 Crypt (not yet fully defined)
    0xE0 [Reserved to expand the identifier to the next 5 bits]

Usage of the remaining five bits of the header octet (and all subsequent octets) is specified by the *BMCF definition* for the hash scheme. The only exception is the special case <code>0xE0</code>, which is used to indicate that the remaining five bits (the least significant bits) are used for the hash scheme identifier. If needed, usage of this space will be defined later to accommodate additional hash schemes and indicate overflow of the identifier to subsequent octets.

Most MCF hash schemes encode the password digest and salt using some form of base-64 binary-to-text encoding. For conversion to BMCF, such fields are decoded back to their binary forms with the reverse text-to-binary process. Typically, the encoding used by crypt(3) hash schemes does not conform to [RFC 4648 Base64][base64]. Most schemes differ only in the selection of the base-64 alphabet, but others differ in the bit ordering used during the encoding process. For use with BMCF, it is important to decode using precisely the base-64 process that is prescribed by the hash scheme, including bit ordering, alphabet selection, and alphabet letter ordering.

With regard to the hash as a whole, to *decode* is to convert a hash from its textual notation (MCF) to its binary serialization (BMCF).

    BMCF = decode(MCF)

To *encode* is to convert a hash from binary form to textual MCF notation.

    MCF = encode(BMCF)

BMCF definitions for specific hash schemes must ensure data integrity, meaning they perform lossless decode/encode roundtrips for *valid* hashes. Given an invalid hash, behavior may vary by hash scheme or code implementation. An error may be raised; an erroneous hash component may be corrected, breaking data integrity; or the invalid hash component may be encoded and decoded as-is, maintaining data integrity of the invalid hash.

[base64]: https://tools.ietf.org/html/rfc4648#section-4 "RFC 4648 Base64 Specification"



Bcrypt BMCF Definition
----------------------

A [Brcypt][bcrypt] MCF hash looks like this:

    $2y$14$i5btSOiulHhaPHPbgNUGdObga/GC.AVG/y5HHY1ra7L0C9dpCaw8u

Decoded to BMCF, the hash requires 40 bytes, numbered 0–39.

Bcrypt hashes have four components in three fields. The salt and digest share the same field; they are not separated by the <code>$</code> delimiter.

    $<scheme>$<cost>$<salt><digest>

There are four Bcrypt scheme identifiers.

    0x20 $2$
    0x40 $2a$
    0x60 $2x$
    0x80 $2y$

To decode, we start with the scheme identifier in the header octet, byte 0. For the <code>2y</code> scheme in our example, this is <code>0x80</code>. The algorithm identifier uses only the three most significant bits. In the lower five bits we store the cost parameter, which Bcrypt requires to be between 04 and 31 (with the leading zero in MCF). With five bits we can store any value between 00 and 31. Adding <code>14</code> to the example header octet, we now have <code>0x8E</code>.

The 128-bit salt and 184-bit digest in the MCF representation are base-64 encoded. Bcrypt base 64 can be encoded and decoded as described in [RFC 4648][base64], but using the following alphabet, and omitting the <code>=</code> padding:

    ./ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789

The Bcrypt salt is the first 22 characters from the salt/digest field, and the digest is the final 31 characters. The salt and digest must be base-64 decoded separately. Decoding the salt yields 16 bytes, appended to the header octet in positions 1–16. Decoding the digest yields 23 bytes to occupy positions 17–39, completing the decoded hash.



Code Implementations
--------------------

There is a [PHP implementation of BMCF][bmcfphp].

[bmcfphp]: https://github.com/ademarre/mcf-hash-encoder-php "McfHash - BMCF Implementation in PHP"
