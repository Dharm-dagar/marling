# Part 3: Prompt Preparation Document

## Integrity Declaration

"I declare that all written content in this assessment is my own work, created without the use of AI language models or automated writing tools. All technical analysis and documentation reflects my personal understanding and has been written in my own words."

---

## Selected PR: Web readonly (#3877)

**GitHub Link:** https://github.com/beetbox/beets/pull/3877  
**Issue Reference:** https://github.com/beetbox/beets/issues/3870

---

## 3.1.1 Repository Context

Beets is a command-line music library management application written in Python, designed specifically for music collectors who care about organizing their digital music properly. The core idea behind beets is simple but powerful: import your music once, let beets fix all the messy metadata, and keep your collection organized forever after that.

The application works by scanning music files, matching them against online databases like MusicBrainz to fetch accurate metadata, applying consistent file naming patterns, and storing everything in a local SQLite database. Users interact with beets primarily through command-line commands like `beet import`, `beet ls`, and `beet modify`.

What makes beets stand out is its plugin architecture. The core application handles basic library management, but dozens of plugins extend functionality in various directions. There is a plugin for fetching lyrics, one for analyzing audio content, one for generating playlists, and many more. The web plugin specifically turns beets into a music server, exposing your library through a REST API that other applications can query.

The intended users are music enthusiasts who have large collections of MP3s, FLACs, or other audio formats and want to maintain accurate, consistent metadata across their entire library. These are typically technical users comfortable with command-line applications and configuration files. Some users run beets as a server component in home media setups, which is where the web plugin becomes essential.

The problem domain beets addresses is digital music organization, specifically the chaos that results from downloading music from various sources, each with inconsistent or incorrect tags. Beets brings order to this chaos through automated identification and tagging.

---

## 3.1.2 Pull Request Description

Pull request #3877 introduces a security enhancement to the beets web plugin by adding a read-only mode that restricts what HTTP operations clients can perform. This change directly responds to issue #3870, which highlighted that the web interface allows destructive operations like DELETE and PATCH by default, creating a potential risk when the interface is exposed on a network.

Before this PR, the web plugin accepted all HTTP methods without restriction. A user who started the web server with `beet web` would unknowingly expose an interface where any client on the network could delete items from their library or modify metadata. There was no built-in way to prevent this without setting up external access controls like a reverse proxy with authentication.

The changes this PR introduces center around a new configuration option called `readonly`. When this option is `true` (which is now the default), the plugin will reject any DELETE or PATCH request with a 405 Method Not Allowed response. GET requests for browsing and streaming music continue to work normally. Users who want to allow modifications must explicitly set `readonly: no` in their beets configuration file.

The previous behavior allowed all HTTP methods unconditionally. The new behavior blocks potentially destructive methods unless the user opts in. This follows security best practices where the safe option is the default and users must consciously enable risky features.

This is technically a breaking change for anyone who was using DELETE or PATCH operations, but the discussion in issue #3870 concluded that security should take priority. The fix for affected users is straightforward: add one line to the configuration file.

---

## 3.1.3 Acceptance Criteria

✓ When a client sends a GET request to any web plugin endpoint, the request should be processed normally regardless of the `readonly` setting.

✓ When a client sends a DELETE request and `readonly` is `true` (default), the server should respond with HTTP 405 Method Not Allowed.

✓ When a client sends a PATCH request and `readonly` is `true` (default), the server should respond with HTTP 405 Method Not Allowed.

✓ When a client sends a DELETE request and `readonly` is explicitly set to `false`, the server should process the deletion normally.

✓ When a client sends a PATCH request and `readonly` is explicitly set to `false`, the server should process the modification normally.

✓ The implementation should read the `readonly` configuration option from the `web` section of the beets config file.

✓ The default value for `readonly` should be `true` when not specified in configuration.

✓ The documentation should clearly explain the `readonly` option, its default value, and how to disable it.

✓ All existing tests for the web plugin should continue to pass after adjusting for the new default behavior.

✓ New tests should verify the blocking behavior for DELETE and PATCH when readonly is enabled.

---

## 3.1.4 Edge Cases

### Edge Case 1: Missing Configuration Section

When the user's configuration file has no `web` section at all, the plugin should still apply the `readonly: true` default. The implementation must handle the scenario where the config section doesn't exist without raising an error. This requires proper config initialization with default values.

### Edge Case 2: Invalid Configuration Value

If a user sets `readonly: maybe` or some other non-boolean value, the plugin needs to handle this gracefully. Options include falling back to the safe default (true), raising a clear configuration error, or interpreting any truthy/falsy value according to YAML conventions.

### Edge Case 3: Other HTTP Methods

The PR focuses on DELETE and PATCH, but what about PUT or POST? The implementation should be clear about which methods are blocked. If the web plugin adds new write endpoints in the future that use POST, the readonly flag's behavior should be documented to either cover all non-GET methods or explicitly list which methods it blocks.

### Edge Case 4: HEAD and OPTIONS Requests

HEAD requests (which are like GET but without body) and OPTIONS requests (for CORS preflight) should probably be allowed even in readonly mode since they don't modify data. The implementation should consider these read-only methods and not block them.

### Edge Case 5: Concurrent Configuration Changes

If someone modifies the beets config file while the web server is running, the behavior regarding readonly could be confusing. The implementation should document whether config changes require a server restart or are picked up dynamically.

---

## 3.1.5 Initial Prompt

You are implementing a security feature for the beets music library web plugin. The web plugin exposes a REST API that allows clients to browse, stream, modify, and delete items from a music library. Your task is to add a read-only mode that prevents destructive operations by default.

**Context:**
Beets is a Python music library manager with a plugin architecture. The web plugin (located in `beetsplug/web.py`) uses Flask to provide HTTP endpoints. Currently, all HTTP methods are allowed on all endpoints, meaning anyone who can reach the web server can delete or modify library items.

**Requirements:**

Implement a new configuration option called `readonly` under the `web` plugin section. This option should:
- Default to `true` when not specified
- Be read from the beets configuration system using `config['web']['readonly'].get(bool)`
- Block DELETE and PATCH requests when enabled
- Return HTTP 405 Method Not Allowed for blocked requests
- Allow GET, HEAD, and OPTIONS requests regardless of the setting

**Implementation Details:**

1. Add the readonly check at the appropriate point in request handling. Consider using Flask's `before_request` decorator to intercept requests before they reach handlers.

2. When a blocked method is detected, return a proper HTTP response:
   - Status code: 405
   - Include appropriate headers like `Allow: GET, HEAD, OPTIONS`
   - Optionally include a message body explaining why the request was blocked

3. Read configuration using beets' config system. Handle the case where the `web` section or `readonly` option doesn't exist by defaulting to `true`.

**Acceptance Criteria to Verify:**

- GET requests work normally with readonly enabled
- DELETE requests return 405 when readonly is true
- DELETE requests succeed when readonly is false  
- PATCH requests return 405 when readonly is true
- PATCH requests succeed when readonly is false
- Default behavior (no config specified) blocks DELETE and PATCH

**Edge Cases to Handle:**

- Configuration section missing entirely
- Invalid boolean value in configuration
- HEAD and OPTIONS methods should always be allowed
- Document behavior clearly so users know how to enable write operations

**Testing Requirements:**

Add tests in `test/test_web.py` that verify:
- Blocking behavior with readonly=true (explicit and default)
- Allowing behavior with readonly=false
- Correct HTTP status codes and headers

**Documentation Requirements:**

Update `docs/plugins/web.rst` to document the new `readonly` option, explaining:
- What it does
- The default value and why
- How to disable it for users who need write access
- Security implications of disabling it

Add a changelog entry in `docs/changelog.rst` describing this security improvement.

The implementation should prioritize clarity and security. The code should make it obvious that this is a security feature protecting users who might not realize their library is exposed.
