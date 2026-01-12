# Part 3: Prompt Preparation Document

## Integrity Declaration

"I declare that all written content in this assessment is my own work, created without the use of AI language models or automated writing tools. All technical analysis and documentation reflects my personal understanding and has been written in my own words."

---

## Selected PR: Web readonly (#3877)

**GitHub Link:** https://github.com/beetbox/beets/pull/3877  
**Issue Reference:** https://github.com/beetbox/beets/issues/3870

---


## 3.1.1 Repository Context

Beets is a Python based command-line music library management program targeting music collectors who are concerned with how they arrange their digital music in their collections. The essence of beets is quite straightforward but mighty: once you have imported your music, beets should clean up all the sloppy metadata and then your collection would be structured eternally.

It operates by scanning music files, comparing them with online databases such as MusicBrainz to obtain correct metadata, using standardized patterns of file names, and creating all of it into a local SQLite database. Beets are used by its users by issuing commandline instructions such as `beet import`, `beet ls` and `beet modify`.

The outstanding feature of beets is its plugin architecture. The core application is the one that manages simple library tasks, yet there are dozens of extensions that are used to add functionality on different directions. It has a plug-in to get lyrics, one to analyze audio files, one to create playlists and many others. The web plugin in particular makes beets a music server, and opens up your library to a REST API that can be queried by other programs.

The target audience is people who love music and have large amounts of MP3s, FLACs, or any other audio files and desire to have consistent and accurate metadata of their entire library. They are usually technical users who are familiar with command-line applications and configuration files. There are users that have beets operate as a server component in home media systems, where the web plugin is required.

The issue area that beets is dealing with is the organization of digital music, which is the mess that ensues due to downloading music of all kinds, each of them having different or wrong tags. Beets provides sanity to such disorder by automated identification and tagging.
---

## 3.1.2 Pull Request Description

Pull request #3877 provides a security improvement to the beets web plug-in by the addition of a read-only feature that limits the type of operations that clients can do with it. This is a direct reaction to issue #3870, which pointed out that the web interface would by default support destructive operations, such as DELETE and PATCH, which is potentially dangerous when the interface is published to networking.

Prior to this PR, the web plugin had no restriction on the type of HTTP methods it accepted. Anyone who had started the web server using `beet web` would be unaware that he or she was exposing an interface, which any client on the network could use to delete items on their library or update metadata. No inbuilt mechanism existed to discourage this without installing external access controls such as a reverse proxy with authentication.

The modifications this PR makes revolve around a new option of configuration that is the so-called `readonly`. When this is `true` (the default is now to make this true), then the plugin will decline every DELETE or PATCH request with the response 405 Method Not Allowed. Browsing and streaming music are also normal using get requests. Users wishing to permit modification need to explicitly configure beets as read-only by putting `readonly: no` in their beets configuration file.

The former policy was permissive to all HTTP methods. The new behavior prevents the possibility of destructive ways unless the user chooses to do so. This is in accordance with security best practices in which the default is the safest default and risky features have to be enabled by a user.

This is technically a breaking change to any user who had been using DELETE or PATCH operations, although the debate in issue #3870 concluded that security is more important. The solution to affected users is simple, one line should be added to the configuration file.

---

## 3.1.3 Acceptance Criteria

✓ A client that sends a GET request to any endpoint of any web plugs must receive the request and process it as usual,  irrespective of the `readonly` setting.

✓ In the case of a client issuing a DELETE or PATCH request and with `readonly` having the value of `true` (the default), the server must issue an HTTP 405 Method Not Allowed response.

✓ On a request of a client with a DELETE or PATCH  request and the setting explicit like the `readonly` to `false`, the deletion should be carried out in the normal manner by the server.

✓ The implementation is expected to read the `readonly` configuration setting in the beets config file under the web section.

✓ When not configured, the default value of the feature is `readonly` as which the default value is also `true`.

✓ The documentation must specify clearly the option of `readonly`, its default and the way of disabling it.

✓ All the tests that were already done on the web plugin must pass upon being adjusted to the new default behavior.

✓ The behavior of the blocking of the DELETE and PATCH should be checked with new tests when the active mode is the readonly.

---

## 3.1.4 Edge Cases

### Edge Case 1: Missing Configuration Section

In case the configuration file of the user does not contain any `web` section at all, the default of the readonly should still be used by the plugin (`readonly:true`). The implementation should be able to cope with the situation when the config section is not present without creating an error. This necessitates default configuring.

### Edge Case 2: Invalid Configuration Value

In the case when a user enters a non-boolean value such as `readonly : maybe`, the plugin must gracefully deal with it. There are the possibilities of a fall to the safe default (true), an explicit error of configuration, or any truthy/falsy value being interpreted based on YAML conventions.

### Edge Case 3: Other HTTP Methods

The PR is geared towards DELETE and Patch, and what of PUT or POST? The application must be clear on which methods are blocked. In the case that the web plugin has such new write endpoints in future, which are POST, the behavior of the readonly flag should be documented either to allow all non-GET operations, or explicitly to list the operations that are being blocked by the readonly flag.

### Edge Case 4: HEAD and OPTIONS Requests

HEAD requests (not a GET but with no body) and OPTIONS requests (CORS preflight requests) should likely be permitted operating in the readonly mode as they do not alter data. These read-only methods should be taken into consideration during the implementation and not blocked.


---

## 3.1.5 Initial Prompt

This is a security feature you are implementing on the beets music library web plugin. The web-based plugin presents a REST API, which enables customers to access, stream, edit, and remove music in a music library. You are to modify it to include a read-only mode that does not allow any destructive operation.

**Context:**
Beets is a Python library music manager that has a plug-in architecture. Fiasco The web plugin (found in `beetsplug/web.py`) uses Flask to offer endpoints through the HTTP protocol. Nowadays, all the HTTP methods are permitted on any endpoint, this is, anyone having access to the web server is allowed to delete or alter library items.

**Requirements:**

Add a new web configuration option known as the `readonly` in the `web` section. This option should:
- default to `true` in case it is not mentioned.
- It can be read with the configured system of beets with the help of the following method:
- `config['web']['readonly'].get(bool)`
- Delete and patch requests are blocked on by default.
- BLOCKED Requests are not allowed to return HTTP 405 Method Not Allowed.
- Permit GET requests, HEAD requests and OPTIONS requests in any setting.

**Implementation Details:**

1. At the correct stage of request handling, add the check of being read only. It is possible to use the handlers `before_requests` are received by the handlers, which could be achieved with Flask using the before-request decorator.

2. In case of suspecting a blocked method, make a correct HTTP response:
- Status code: 405
- Incorporate the right headers, such as `Allow: GET, HEAD, OPTIONS`.
- (Optional) add a message body stating the reason as to why the request was blocked.

3. Beets configuration system Read configuration. Deal with the observation that the default of the `web` section or the `read only` option is not present by default.




**Acceptance Criteria to Verify:**

- GET requests work normally with readonly enabled.
- DELETE and PATCH requests return 405 when readonly is true.
- DELETE and PATCH requests succeed when readonly is false.
- Default behavior (no config specified) blocks DELETE and PATCH.

**Edge Cases to Handle:**

- Configuration section is not present at all.
- Bogus configuration of falsehood.
- HEAD, OPTIONS methods should never be prohibited.
- Behaviors of documents should be documented in a manner that the user understands how to access the write operations.

**Testing Requirements:**

Add tests in test/test_web.py which test:
- Preventing behavior reading only (explicit and default)
- Behavior that is also read-only allowable
- Behavior may be read-only as well.
- Proper HTTP status codes and headers.

**Documentation Requirements:**

Update `docs/plugins/web.rst` to document the new `readonly` option, explaining:
- What it does
- The default value and why
- How to disable it for users who need write access
- Security implications of disabling it

Add a changelog entry in `docs/changelog.rst` describing this security improvement.

The implementation should prioritize clarity and security. The code should make it obvious that this is a security feature protecting users who might not realize their library is exposed.
