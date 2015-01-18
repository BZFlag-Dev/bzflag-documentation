# BZFlag Services 4.0 Design Document

## Preface
This document will attempt to describe a possible new architecture for the BZFlag services. It should be designed with privacy and security in mind throughout the process. If feasible, the new system should be distributed/distributable, allowing the system to be distributed across anywhere from a single system to dozens of systems. A modular design would allow for isolation of resources and sensitive data. It would also allow modules to be replaced without requiring massive overbearing changes to the entire system. We should also look into using open standards and existing libraries when appropriate.

Going fully modular may be overkill for the 4.0 services.  For instance, the Game Authentication module and Web Authentication module may just be a single module that provides both parts.

## Previous System Generations

This is just an informal versioning of past systems:
* 1.0 - old C/C++ based list server that was used by 1.7
* 2.0 - original PHP/MySQL list server used by 1.10
* 3.0 - PHP/MySQL list server with central authentication used by 2.0 and 2.4

---

## Game Authentication

The **Game Authentication** module will provide in-game authentication services.

### Existing System
The current (3.0) system uses a token based authentication system. A client sends a username and password to a single central server (the list server) and if valid, the list server responds with a one-time-use, time restricted and IP restricted (if the remote server provides the client IP) authentication token. This authentication token gets passed to a game server or website along with the player's username.  The remote server can then send the username, token, and optionally the client IP and global group names. If the token is valid, then a positive response is returned, along with any requested groups that the user is a member of.

### Possible Obstacles

#### IP Validation
The IP checking will be problematic once IPv6 support is added as they may connect to the authentication server over IPv6 but to a game server over IPv4, thus the address will never match. **So in this situation, what can be done to help ensure the security of the token? Should the authentication server be spoken to over both IPv4 and IPv6 during an authentication request to ensure both addresses are captured? What about IPv6 privacy extensions that may use random addresses? Is there some other way to ensure that the token cannot be used by someone else?**

#### Distributed Authentication
How will we handle ensuring that a token created on one authentication node can be successfully checked by the remote server or website? Keep in mind that direct communication with an authentication node may not be possible as there may be one or more load balancers in use. So that leaves us with some possible options:
1. Allow for some method of forcing a load balancer to speak to a specific node.
2. Sync tokens across all nodes in real-time. This would either be done by having all authentication nodes speak to the same database (either a single instance, a cluster, or a master/slave environment) or some other secure shared memory system (can redis or memcache support secure access and communication across the insecure Internet?)
3. Have a single node that handles the creation of tokens that sits behind the other authentication nodes and have all nodes communicate with that server for token creation and verification. The problem with this is that there is then a single point of failure.
4. Include a node ID as part of the token so that one authentication node could proxy the request to the proper node.
5. Possibly some combination of 2 and 4, where it first checks for the shared database/memory, but if it isn't there, it directly asks the other node.
6. Some sort of a private key or certificate based system?

### Requirements
* Must use a strong password hashing method (such as bcrypt) with sufficient salt length
* Must use strong encryption for all communication (even in the case of pre-hashed passwords)

### Discussion

For some time we've pondered using OpenLDAP (and even had someone write the start of a daemon that used OpenLDAP) to handle our authentication.  At this point, however, it looks like the best it can do for password hashing is salted SHA1, which is pretty weak.  I did find some experimental bcrypt module, but it said to not use it in a production environment and really has not seen much activity.

Kerberos is probably a bit overkill for our use case as well and may also have some hash algorithm limitations.

PHP versions 5.5 and above have a password_hash function (and associated password_verify and password_needs_rehash functions) that currently support bcrypt (which limits passwords to at most 72 characters), and will/may in the future support stronger hashing algorithms as the need arises. It is also possible to use it with PHP 5.3.7 or higher by making use of a [password_compat] PHP library.

[password_compat]: https://github.com/ircmaxell/password_compat

---

## Web Authentication

The **Web Authentication** module will provide web-based authentication services to first-party and third-party websites.

Possible open-standards:
* OpenID and/or OAuth
* Mozilla Persona / BrowserID
* [cosign]
* [OpenAM]
* [CAS]
* SAML / SAML 2.0
* [gluu]

[cosign]: http://weblogin.org/
[OpenAM]: http://openam.forgerock.org/
[CAS]: http://www.unicon.net/opensource/cas
[gluu]: http://www.gluu.org/

### Requirements
* Do our best to restrict what personal information is required to be shared with third party sites
  * Should we avoid systems that use the user's email address as their ID?
* Should ideally support multiple nodes behind one or more load balancers, though less critical than the **Game Authentication**
* Must use strong encryption for all communication

### Discussion

Some web authentication systems use the user's email address as the identifier that is passed to the remote sites. Ideally we would not expose the user's email address directly, and instead have it be an optional piece of information that can be passed along with the user's permissions if a remote site requests it.

However, the main purpose of web authentication is to limit the need to send passwords to other sites. Many users would use the same password for both the official BZFlag services and for other third-party sites, opening up either to a future attack if a database or password is exposed. So even if we use a system that uses the email address as the ID, the primary goal will still be fulfilled.

---

## Master Server List

The **Master Server List** module will provide a list of servers to game clients and other services.

### Requirements
* Proper JSON output for the server list (as this will allow future additions to the returned data without breaking parsers)

### Wants
* Additional information about servers, such as the location (country), owner, map name, etc.

### Discussion

...

---

## Main Website

The **Main Website** module will be the source of information about the game, including media (screenshots and videos), download links, and documentation.

### Discussion

Unless there is a real need to for dynamic content, the main site should just be a static files. By being a static site, it would reduce the processing and memory resources necessary to run it, and would mean that a single node could likely handle a large traffic influx. There are a number of static site generators that would ease creating and maintaining a static website.

---

## Player Portal Website

The **Player Portal Website** will be the area that players create and manage their account. Group and key management will be part of this site as well.

### Requirements
* Account management
* Group management

### Discussion

...

---

## Forums

The **Forums** will be a discussion area for players. This may end up being an existing forum solution like phpBB 3.1 or a more custom system more tailored to our architecture.

### Discussion

The forums will be run from an isolated node since they do not need access to any other data.  Authentication for the forums should be handled by the Web Authentication modules.

---

## Game Resources

The **Game Resources** module will be used to host game related resources, such as textures, sounds, videos, and maps. It will replace the existing images.bzflag.org system and should be designed around the needs of map authors. Changes to the game should also be considered to make this system work better, such as automatically fetching updated versions of files.