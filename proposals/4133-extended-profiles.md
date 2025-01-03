# MSC4133: Extending User Profile API with Custom Key:Value Pairs

This proposal aims to enhance user profiles in the Matrix ecosystem by introducing customisable
key:value pairs to global profiles. By allowing users to add arbitrary public information
(such as preferred languages, organisational roles, or other relevant details) we can enrich
user interactions without impacting existing functionality.

## Proposal Overview

Currently, the Matrix protocol supports limited user profile fields: `avatar_url` and `displayname`.
This proposal is modelled on the [current API endpoints](https://spec.matrix.org/v1.12/client-server-api/#profiles),
extending the profile API to include extra fields, enabling users and servers to publish key:value
pairs to their global profiles. This extension provides a flexible framework for users to share
additional public information.

The proposal is designed to be straightforward and compatible with existing clients and servers,
requiring minimal changes to facilitate quick adoption. It complements, rather than replaces,
[MSC1769](https://github.com/matrix-org/matrix-spec-proposals/pull/1769) (Extensible Profiles as
Rooms) by focusing on global profile data without the complexity of per-room profile management.

## Client-Server API Changes

### Get a Profile Field

- **Endpoint**: `GET /_matrix/client/v3/profile/{userId}/{key_name}`
- **Description**: Retrieve the value of a specified `key_name` from a user's profile.
- **Response**:

```json
{
  "key_name": "field_value"
}
```

*Example*: Requesting `GET /_matrix/client/v3/profile/@alice:matrix.org/displayname` returns:

```json
{
  "displayname": "Alice"
}
```

### Set a Profile Field

- **Endpoint**: `PUT /_matrix/client/v3/profile/{userId}/{key_name}`
- **Description**: Set or update the value of a specified `key_name` in the user's profile,
  if permitted by the homeserver.
- **Request Body**:

```json
{
  "key_name": "new_value"
}
```

*Example*: Setting `PUT /_matrix/client/v3/profile/@alice:matrix.org/displayname` with:

```json
{
  "displayname": "Alice Wonderland"
}
```

**Note**: As a `DELETE` endpoint exists to remove a key, setting a `null` value with the `PUT`
method SHOULD NOT delete the key but rather retain it with a `null` value. Servers MAY forbid a
`null` request if their policy does not allow keys to exist with a `null` value.

### Delete a Profile Field

- **Endpoint**: `DELETE /_matrix/client/v3/profile/{userId}/{key_name}`
- **Description**: Remove a specified `key_name` (and its value) from the user's profile,
  if permitted by the homeserver.

### Get All Profile Fields

- **Endpoint**: `GET /_matrix/client/v3/profile/{userId}`
- **Description**: Retrieve all profile fields for a user, identical to the
  [current API](https://spec.matrix.org/latest/client-server-api/#get_matrixclientv3profileuserid).
- **Response**:

```json
{
  "avatar_url": "mxc://matrix.org/MyC00lAvatar",
  "displayname": "John Doe",
  "m.example_field": "value1",
  "org.example.job_title": "Software Engineer"
}
```

## Server-Server API Changes

The federation endpoint `GET /_matrix/federation/v1/query/profile` will mirror the client-server
API changes to ensure profile information is consistent between local and federated users.

As per the current stable endpoint, it accepts an optional `field` query string parameter to
request a single field. At the time of writing, the Matrix specification says:

> If no `field` was specified, the response should include the fields of the user's profile that
> can be made public, such as the display name and avatar.

Given this wording, homeservers currently have the flexibility to decide whether some fields are
published over federation, and this proposal continues to allow this behaviour.

## Capabilities

A new capability `m.profile_fields` controls the ability to *set* profile fields and is advertised
on the `GET /_matrix/client/v3/capabilities` endpoint.

This capability deprecates the use of `m.set_displayname` and `m.set_avatar_url`, which are not
required when this capability is present.

Clients SHOULD check for this capability before attempting to create or modify a profile field.

### Capability Structure

```json
{
  "capabilities": {
    "m.profile_fields": {
      "enabled": true,
      "allowed": ["m.example_field", "org.example.job_title"],
      "disallowed": ["org.example.secret_field"]
    }
  }
}
```

### Behaviour

- **When capability is missing**: Clients SHOULD assume extended profiles are supported and that
  they can be created or modified. If a server intends to deny some (or all) changes, it SHOULD use
  the capability to advertise this, improving the client experience.

- **When `enabled` is `false`**: Clients SHOULD expect to display profiles but NOT create or update
  fields. Any attempt to do so SHOULD result in a `403 Forbidden` error. This does not affect
  `avatar_url` and `displayname` fields, which are allowed for compatibility purposes.

- **When `enabled` is `true`**: Clients SHOULD allow users to create or update fields, except those
  keys listed in the `disallowed` array. Servers MAY specify an `allowed` array to allowlist fields
  that users can set. If both `allowed` and `disallowed` keys are provided, the `disallowed` one
  should be ignored. Clients SHOULD receive `400 Bad Request` or `403 Forbidden` responses if
  server-side policies prevent them.

## Key and Namespace Requirements

Profiles MUST be at most 64 KiB (65,536 bytes) in size, as measured in Canonical JSON, including
the `avatar_url` and `displayname` fields.

Homeservers SHOULD NOT enforce namespaces, as future expansions may be unknown to the server, but
clients SHOULD use the correct namespace when creating or updating fields.

Keys MUST follow the [Common Namespaced Identifier Grammar](https://spec.matrix.org/unstable/appendices/#common-namespaced-identifier-grammar),
with the following considerations:

- **Namespace `m.*`**: Reserved for fields explicitly defined in the Matrix specification. Clients
  that do not recognise a field in this namespace MAY attempt to display it but SHOULD NOT attempt
  to update the content unless they understand its formatting and validation requirements.

- **Namespace `tld.name.*`**: For client-specific or unstable fields, using Java package naming
  convention (e.g., `com.example.custom_field`).

Following this change, clients could use `m.example_field` if that field is defined by the Matrix
specification, or `org.example.job_title` for organisation, client-specific fields, or MSC-backed
unstable features. Proposal [MSC4175](https://github.com/matrix-org/matrix-spec-proposals/pull/4175)
demonstrates the process of defining new fields in the `m.*` namespace.

## Error Handling

### 400 Bad Request: Request Exceeds Limits or Is Malformed

- **`M_BAD_JSON`**: Malformed request.

```json
{
  "errcode": "M_BAD_JSON",
  "error": "The provided JSON is malformed."
}
```

- **`M_PROFILE_TOO_LARGE`**: Exceeds total profile size limits.

```json
{
  "errcode": "M_PROFILE_TOO_LARGE",
  "error": "The profile data exceeds the maximum allowed size of 64 KiB."
}
```

- **`M_KEY_TOO_LARGE`**: Exceeds individual key length limits.

```json
{
  "errcode": "M_KEY_TOO_LARGE",
  "error": "The key name exceeds the maximum allowed length of 255 bytes."
}
```

### 403 Forbidden: User Lacks Permission

Unchanged from the [current spec](https://spec.matrix.org/v1.13/client-server-api/#server-behaviour).

### 404 Not Found: Target Cannot Be Found

- **`M_NOT_FOUND`**: Profile key does not exist (this is unchanged, just expanded to
  apply to arbitrary keys).

```json
{
  "errcode": "M_NOT_FOUND",
  "error": "The requested profile key does not exist."
}
```

## Propagation of Profile Fields

The existing fields, `avatar_url` and `displayname`, will continue to trigger state events in each
room. These fields are replicated per-room via member events.

All other fields (unless a future proposal specifies otherwise) WILL NOT trigger state events in
rooms and will exist solely at the global level for storing metadata about the user.

Clients SHOULD consider the increased traffic implications when displaying values (e.g. timezones)
outside of the profile. Servers MAY wish to optimise and relax rate limits on these endpoints in
consideration of this.

## Implementation Details

- Custom fields MUST NOT trigger state events in rooms; their data MUST NOT be replicated to
  `m.room.member` events unless a future proposal creates exceptions for specific fields.

- Servers SHOULD cache remote profiles for at least 5 minutes after retrieval. Until a method for
  servers to notify each other of updates is established, it is recommended that profiles be cached
  no longer than an hour to avoid displaying stale data. Servers SHOULD NOT cache profiles for
  longer than 24 hours to avoid unwanted data persistence.

- Clients MAY provide a UI for users to view and enter custom fields, respecting the appropriate
  namespaces.

- Clients SHOULD only display profiles of users in the current room whose membership status is
  `join`, `invite`, or `knock`. If a client offers an option for any free-text fields to always be
  available in the UI, an option SHOULD be provided to hide or minimise them automatically.

- Servers MAY add, remove, or modify fields in their own users' global profile data, whether for
  moderation purposes or for other policy reasons (e.g., to automatically populate a job title
  based on the user's organisation).

## Potential Issues

There is no method to verify the history of global profile fields over federation. This proposal
updates the global profile only, while other more complex proposals, such as
[MSC1769](https://github.com/matrix-org/matrix-spec-proposals/pull/1769) (Extensible Profiles as
Rooms), offer additional mechanisms for users to track changes to their profile data over time.

Ensuring uniform support across different servers and clients during the rollout phase is crucial.
We do not want users to expect that others will check their profile (e.g. languages) before
communicating with them unless most clients and servers support this feature.

As such, this MSC is designed to be as simple as possible to get initial functionality and data
structures implemented widely, so further extensions can be debated, implemented, and tested with
due care over time.

As this data is stored only at the global level, it won't allow users to modify fields per-room or
track historical changes in profile fields. This proposal recommends that future MSCs only add
certain fields to per-room member events when there is explicit value in doing so, and the current
functionality added by this proposal is not anticipated to have this value.

This proposal also does not offer a method to "broadcast" to other users or homeservers that
changes have occurred. This is intentional to keep the scope of this change narrow and maximise
compatibility with existing servers. A future proposal may wish to use an EDU (such as Presence)
to notify users and homeservers that these custom fields have been updated.

This proposal does not directly address reporting of user profiles over federation, but
[MSC4202](https://github.com/matrix-org/matrix-spec-proposals/pull/4202) offers a facility for
users to report offensive content to the homeserver that account belongs to. This proposal is not
dependent on MSC4202 but encourages the use of moderation options to allow users to report
offensive content.

## Security Considerations

Since profile fields are public, the server is not directly responsible for the privacy of the data;
however, clients SHOULD make users aware that any information published in their profile will be
visible to others on the federated network.

Likewise, if a server automatically publishes data in user profile fields (e.g. setting a job title
based on an organisation's internal user database), then they SHOULD have consent to do so, and
users SHOULD be made aware that data is published on their behalf.

To minimise the impact of abuse, clients SHOULD offer suitable defaults for displaying user-entered
content. A user MAY choose to display fields from all users globally, but *by default* profiles
SHOULD only be shown when the users share the current room and the other user is in the `join`,
`invite`, or `knock` membership states.

If future clients (or spec proposals) implement the ability to set custom user-entered text in
profiles, servers MAY require additional moderation and safety tooling on the server side to
provide better visibility of problematic content, e.g., during reports.

Proposal [MSC4202](https://github.com/matrix-org/matrix-spec-proposals/pull/4202) adds reporting of
user profiles over federation, which offers a facility for users to report offensive content to the
homeserver that account is registered on.

Homeservers and clients SHOULD comply with relevant privacy regulations, particularly regarding
data deletion and retention. Profile data SHOULD be cleared when a user is deactivated, and while
homeservers SHOULD cache remote profiles, they SHOULD avoid caching beyond 24 hours to minimise the
risk of unintended data persistence.

## Alternatives

An alternative approach could involve introducing a completely new API for extended profile
information. However, this may lead to increased complexity for client and server implementations.

At the time of writing, Extensible Profiles as Rooms
([MSC1769](https://github.com/matrix-org/matrix-spec-proposals/pull/1769) and variant
[MSC4201](https://github.com/matrix-org/matrix-spec-proposals/pull/4201)) is under development for
richer and more granular content and privacy controls, which this proposal does not intend to
replace. This proposal focuses on basic global profile data without the complexity of per-room
profile management.

## Unstable Prefixes

### Unstable Profile Fields

Until this proposal is stable, fields SHOULD use an unstable prefix:

```json
{
  "avatar_url": "mxc://matrix.org/MyC00lAvatar",
  "displayname": "John Doe",
  "uk.tcpip.msc4133.m.example_field": "field_value"
}
```

### Unstable Endpoints

Use unstable endpoints when the capability is not yet stable:

- **Get/Set/Delete Profile Fields**:
  - `/_matrix/client/unstable/uk.tcpip.msc4133/profile/{userId}/{key_name}`

### Unstable Capability

Advertise the capability with an unstable prefix:

```json
{
  "capabilities": {
    "uk.tcpip.msc4133.profile_fields": {
      "enabled": true,
      "disallowed": ["org.example.secret_field"]
    }
  }
}
```