# Part 2: Pull Request Analysis

## Integrity Declaration

"I declare that all written content in this assessment is my own work, created without the use of AI language models or automated writing tools. All technical analysis and documentation reflects my personal understanding and has been written in my own words."

---

## Repository Selected: beetbox/beets

I reviewed all 10 pull requests from the beets repository and selected 2 PRs that I could comprehend thoroughly based on their scope, technical clarity, and my understanding of web development and configuration management patterns.

### PRs Reviewed

| PR #    | Title                                                       | Status |
| ------- | ----------------------------------------------------------- | ------ |
| #1808 | Add musicbrainz id option to importer                       | Open   |
| #3145 | Playlist plugin                                             | Open   |
| #3214 | bpd: support MPD 0.16 protocol                              | Merged |
| #3279 | Add parentwork plugin                                       | Merged |
| #3478 | Implement parallel replaygain analysis                      | Merged |
| #3509 | Add plugin for Fish shell tab completion                    | Merged |
| #3568 | First try adding new albuminfo and trackinfo class          | Open   |
| #3877 | Web readonly                                                | Merged |
| #3883 | Experimental "bare-ASCII" matching query                    | Open   |
| #4199 | Allow to configure which fields are used to find duplicates | Merged |

---

## Selected PR #1: Web readonly (#3877)

**GitHub Link:** https://github.com/beetbox/beets/pull/3877

### PR Summary

This PR is a security issue in beets web which defaulted to support the unsecured use of DELETE and PATCH HTTP requests without providing any security protocols. The issue is that any person with a right to use the web interface might alter or erase something through the music library and that is risky when the web interface is made publicly available on the network. The PR adds the new configuration option known as `readonly` which by default is set to `true` that is, it only accepts the GET request unless a user specifies a write operation. This transformation adheres to the principle of least privilege of security by defaulting to the safe practice.

### Technical Changes

**Files/Components Modified:**

- `beetsplug/web.py` - Main web plugin file

  - Added `readonly` configuration option reading
  - Implemented request method checking logic
  - Added 405 Method Not Allowed response for blocked operations

- `docs/plugins/web.rst` - Plugin documentation

  - Added documentation for the new `readonly` configuration option
  - Explained the security implications and how to disable readonly mode

- `test/test_web.py` - Test file for web plugin

  - Added tests for DELETE operations with readonly=true (should fail)
  - Added tests for DELETE operations with readonly=false (should succeed)
  - Added tests for PATCH operations with readonly=true (should fail)
  - Added tests for PATCH operations with readonly=false (should succeed)

- `docs/changelog.rst` - Changelog file
  - Added entry describing the new readonly feature

### Implementation Approach

The implementation is straightforward in the style of a middleware in which a request in the form of an incoming HTTP can be intercepted prior to being passed to the actual handler. The solution is effective since it tests the value of the configuration which is the `readonly` setting (set to default value of `true`) at the beginning of request handling. On receipt of a DELETE or PATCH request and `readonly` is turned on, the request is immediately responded to, prior to further processing, with a 405 Method Not Allowed HTTP response.

The beets configuration is based on the beets configuration file in the `web` section. Users that wish to use write operations have to add the only `readonly: no` explicitly to their configuration. Such design will provide backward compatibility in terms of functionality and transform the default security posture.

This is clean in the Flask web framework adopted by the beets web plugin due to the ability of route handlers to check conditions and exit early with error responses. The PR has also extensive tests confirming the blocking behavior during the state of the readonly and regular operation during the state of the disable.

### Potential Impact

This modification will impact any user that was already using the web plugin based on DELETE or Patch operations. Such users will be forced to change their configuration to include `readonly: no` so as to revert to the old behavior. The main browsing and playback service of the web plugin has not changed because the GET requests are never prohibited.

This is beneficial because the change will enhance security to users who open the web interface to the local network or beyond since deletions can now be prevented by default.

---

## Selected PR #2: Allow to configure which fields are used to find duplicates (#4199)

**GitHub Link:** https://github.com/beetbox/beets/pull/4199

### PR Summary

This PR introduces some flexibility in the way beets identifies duplicate albums during the importation process. Today, beets makes use of a predetermined number of fields (album name and artist) to see whether an album already has a place in the library. Nonetheless, there are still many legitimate reasons why users would want to have several versions of the same album, including the existence of both CD and vinyl version, remasters, or international versions. This PR adds a configuration option, called `duplicate_keys`, which allows a user to do this to prevent beets labeling the different versions of the same album as duplicates: specify what fields to use when checking for duplicates.

### Technical Changes

**Files/Components Modified:**

- `beets/importer.py` - Core importer module

  - Modified duplicate detection logic to use configurable fields
  - Added reading of `duplicate_keys` from configuration
  - Changed duplicate query construction to use dynamic field list
  - Modified both album and singleton item duplicate checking

- `beets/library.py` - Library models module

  - Added methods to support flexible duplicate key queries
  - Enhanced query construction for album and item models

- `beets/dbcore/db.py` - Database core module

  - Added low-level query utility functions
  - Implemented generic query construction based on field lists
  - Provided foundation for flexible attribute matching

- `beets/config_default.yaml` - Default configuration file

  - Added `duplicate_keys` option with default value matching current behavior
  - Preserves backward compatibility by using album and albumartist as defaults

- `docs/reference/config.rst` - Configuration documentation

  - Added documentation for the new `duplicate_keys` option
  - Provided examples of common use cases like adding `format` field
  - Explained how to configure separate keys for albums and items

- `docs/changelog.rst` - Changelog documentation

  - Added entry describing the new duplicate_keys feature

- `test/test_importer.py` - Importer test file
  - Added tests for custom duplicate_keys configuration
  - Added tests verifying that adding format field allows same album in different formats
  - Added tests for edge cases with empty or invalid field names
  - Added tests for singleton item duplicate detection

### Implementation Approach

The duplicate checking logic is changed at the importer and database tier during the implementation. In the past, the code used hard codes to construct the duplicate detection query using field names. This is modified by the PR to a list of field names in the configuration and then builds the query dynamically out of those fields.

Beets constructs a database query when an album is being imported to determine whether or not there is a matching album. In this PR, the fields on which the query is run are based on the `duplicate_keys` configuration. The default value is a current behavior (with album and albumartist), but the user may add such fields as `format`, `label` or `country` to permit multiple versions.

The implementation refactored query construction utilities into the dbcore module, and made them reusable throughout the codebase. This was done by the creation of temporary album and item objects to adequately test flexible attributes and computed fields so that users can not only use simple metadata fields but also inline fields in their duplicate detection rules.

The decision to use a list of field names as the design instead of the more complicated configuration object design keeps the feature simple and easy to use, but gives it a lot of flexibility. Another feature introduced by the PR is additional configuration paths of the albums and items since it is possible that the duplicate detection criteria could vary between both types.

### Potential Impact

This modification has an impact on such import workflow when a user wishes to have several versions of albums. Components of the system that are affected are:

- **Importer module** - Modifications to core logic in order to detect duplicates.
- **Configuration system** - Mandatory, New option to parse and validate.
- **Database queries** - Modified patterns of querying in duplicate checking.
- **Library models** - Improved using new query construction methods.

The change is backward compatible because the default set-up corresponds to the current behavior. Users that do not change their config will not notice any difference. To users who choose this option and add more fields such as format to their duplicate keys, the import process has now been modified to allow them to have various versions of the same album in different formats without generating duplicate errors.

---

## Conclusion

I have chosen these two PRs since they have both practical user-facing issues and give explicit technical solutions. PR #3877 addresses a security issue in a puristically configuration-based way, whereas PR #4199 adds more flexibility in library management to the user. Changes to the codebase (web plugin and importer) are traceable and testable and are modified by both PRs.
