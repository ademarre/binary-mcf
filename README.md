Binary Modular Crypt Format
===========================

Binary Modular Crypt Format (BMCF) is the binary counterpart to Modular Crypt Format (MCF). MCF is a de facto standard for encoding password hashes in an ASCII string, and it allows hashes generated with different hash algorithms to be stored alongside one another compatibly. MCF is not formally defined, but [it has been documented][mcf] by the authors of the [PassLib password hashing library][passlib] for Python.

Modular Crypt Format originated with the [crypt(3)][crypt] Unix C library function, and has since been implemented in several other libraries. Historically, MCF was used to encode the password hashes of operating system users for storage in a text file. Nowadays, MCF is also used in systems which store password hashes in database management systems. On large systems with many users, it is often desirable to maximize storage efficiency. To that end, we present Binary Modular Crypt Format as a compact binary serialization of MCF hashes. Depending on the hashing scheme, BMCF reduces size by approximately 25–30%.

[mcf]:      pythonhosted.org/passlib/modular_crypt_format.html  "Passlib's Documentation of MCF"
[passlib]:  http://pythonhosted.org/passlib/index.html          "Passlib password hashing library for Python"
[crypt]:    en.wikipedia.org/wiki/Crypt_(C)                     "crypt(3) Unix C library function"


MCF Example
-----------

A typical MCF hash looks like this:

    $2y$14$i5btSOiulHhaPHPbgNUGdObga/GC.AVG/y5HHY1ra7L0C9dpCaw8u

MCF hashes start with <code>$</code>, which is also used to delimit fields. The field following the first <code>$</code> identifies the hashing scheme that is being used. Beyond that, field definitions vary by algorithm, but generally include algorithm parameters, a salt, and the actual hash digest.

The example is a [Bcrypt][bcrypt] hash, identified by the <code>2y</code> scheme. The next field, <code>14</code>, is the Bcrypt <code>cost</code> parameter. The first 22 characters of the final field represent the 128-bit salt, <code>i5btSOiulHhaPHPbgNUGdO</code>. The remaining 31 characters represent the 184-bit Bcrypt hash digest. The salt and hash digest are base64-encoded using a special alphabet.

Note that while Bcrypt puts the salt and digest in the same field, most other schemes use separate fields, usually to accomodate variable-length salts.

[bcrypt]: http://en.wikipedia.org/wiki/Bcrypt "Bcrypt, Blowfish-based password hashing"


General Specification
---------------------

*So far, we've only fully defined BMCF for BCrypt*

Binary MCF hashes start with a header octet. This first byte employs the most significant three bits for the algorithm identifier as follows.

    0x00 [Reserved; perhaps for the pre-MCF implicit DES schemes]
    0x20 $2$    Bcrypt
    0x40 $2a$   Bcrypt
    0x60 $2x$   Bcrypt
    0x80 $2y$   Bcrypt
    0xA0 $5$    SHA-256 Crypt (not yet fully defined)
    0xC0 $6$    SHA-512 Crypt (not yet fully defined)
    0xE0 [Reserved to expand the algorithm identifier to the next 5 bits]

Usage of the remaining five bits of the header octet is defined per hash scheme. The special case <code>0xE0</code> is used to indicate that the remaining five bits (the least significant bits) are used for the algorithm identifier. If needed, usage of this space will be defined later to accomodate additional hash schemes and also overflow of the algorithm identifier to subsequent octets.

Most MCF hash schemes encode the hash digest and salt using some form of base-64 binary-to-text encoding. These components are decoded back to their compact binary forms with the reverse text-to-binary process. Typically, the encoding used does not conform to [RFC 4648 Base64][base64]. Most schemes differ only in the selection of the base-64 alphabet, but others differ in the bit ordering used during the encoding process. It is important to decode using precisely the base-64 process that is prescribed by the hash scheme, including bit ordering, alphabet selection, and alphabet letter ordering.

[base64]: http://tools.ietf.org/html/rfc4648#section-4 "RFC 4648 Base64 Specification"
