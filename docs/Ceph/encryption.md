# Encryption 

## Encryption in Transit

The msgr2 protocol supports two connection modes:

* crc
    * Provides strong initial authentication when a connection is established with cephx.
    * Provides a crc32c integrity check to protect against bit flips.
    * Does not provide protection against a malicious man-in-the-middle attack.
    * Does not prevent an eavesdropper from seeing all post-authentication traffic.

* secure
    * Provides strong initial authentication when a connection is established with cephx.
    * Provides full encryption of all post-authentication traffic.
    * Provides a cryptographic integrity check.
    
> NOTE: The default mode is crc.

When installing ODF, the `In transit encryption` checkbox must be selected on the Security screen to enable this feature.  Documentation indicates it must be enabled during installation.