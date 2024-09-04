# MSC0000: Template for new MSCs

*Note: Text written in italics represents notes about the section or proposal process. This document
serves as an example of what a proposal could look like (in this case, a proposal to have a
template) and should be used where possible.*

## Introduction

*In this first section, cover the problem and provide a broad overview of the solution. Mention the
expected impact if applicable. For example, this document states that we're missing a template,
which causes confusion, and proposes a template as the solution. If your proposal is more invasive
(e.g., changing how servers discover each other), list the expected impact here.*

*If you're having trouble coming up with a description, ask yourself, "How does this proposal
improve Matrix?" The answer could reveal a small impact, which is okay.*

There can never be enough templates in the world, and MSCs shouldn't be any different. The level
of detail expected of proposals can be unclear - this is what this example proposal (which doubles
as a template itself) aims to resolve.

## Proposal

*Reinforce your position from the introduction in more detail, and cover the technical points of
your proposal. Include rationale for your proposed solution and detail why parts are important.
Not including enough detail can result in people guessing, leading to confusing arguments in the
comments section. This example covers why templates are important again, giving a stronger argument
as to why we should have a template. Afterwards, it covers the specifics of what the template could
look like.*

### Client-Server API Changes

*Detail the changes to the client-server API here. Include new endpoints, modifications to existing
endpoints, and any other relevant changes.*

1. **GET `/_matrix/client/v3/example/{param}`**: This endpoint will return an example response:

    ```json
    {
      "example_key": "example_value"
    }
    ```

2. **PUT `/_matrix/client/v3/example/{param}`**: This endpoint will set an example value:

    ```json
    {
      "example_key": "new_value"
    }
    ```

3. **DELETE `/_matrix/client/v3/example/{param}`**: This endpoint will delete an example key:

    ```json
    {
      "example_key": "value_to_delete"
    }
    ```

### Server-Server API Changes

*Detail the changes to the server-server API here. Include new endpoints, modifications to existing
endpoints, and any other relevant changes.*

**GET `/_matrix/federation/v1/query/example`** will mirror the client-server API changes to ensure
example information is consistently available across the federated network.

### Capabilities

*Detail any new capabilities introduced by this proposal.*

A new capability `m.example_capability` will be introduced to control the ability to use the new
example endpoints.

Example capability object:

```json
{
  "capabilities": {
    "m.example_capability": {
      "enabled": true
    }
  }
}
```

### Error Handling

*Detail the error handling mechanisms and error codes introduced by this proposal.*

To ensure clear communication of issues, the following error codes and messages will be used:

- **400 Bad Request**: When the request is malformed or exceeds specified limits.
  - **Error Code for Malformed Request**: `M_BAD_JSON`

    ```json
    {
        "errcode": "M_BAD_JSON",
        "error": "The provided JSON is malformed."
    }
    ```

  - **Error Code for Exceeding Size Limit**: `M_TOO_LARGE`

    ```json
    {
        "errcode": "M_TOO_LARGE",
        "error": "The data exceeds the maximum allowed size."
    }
    ```

- **403 Forbidden**: When the user does not have permission to perform the action.
  - **Error Code**: `M_FORBIDDEN`

    ```json
    {
        "errcode": "M_FORBIDDEN",
        "error": "You do not have permission to perform this action."
    }
    ```

- **404 Not Found**: When the requested resource does not exist.
  - **Error Code**: `M_NOT_FOUND`

    ```json
    {
        "errcode": "M_NOT_FOUND",
        "error": "The requested resource does not exist."
    }
    ```

### Propagation of Changes

*Detail how changes will be propagated, if applicable.*

Changes to example fields will generate state events in all rooms the user is a member of.

### Key/Namespace Requirements

*Detail any key/namespace requirements for new fields introduced by this proposal.*

The namespace for field names is defined as follows:

- The namespace `m.*` is reserved for fields defined in the Matrix specification.
- The namespace `u.*` is reserved for user-defined fields.
- Client-specific or unstable fields MUST use the Java package naming convention: `tld.name.*`.

### Size Limit

*Detail any size limits for new fields or data structures introduced by this proposal.*

The key *must* be a string of *at least* one character, and *must* not exceed 255 bytes. The entire
JSON object must not exceed 64KiB.

### Implementation Details

*Detail any implementation details, including optional but recommended features, compliance with
regulations, and other relevant information.*

## Potential Issues

*Not all proposals are perfect. Sometimes there's a known disadvantage to implementing the proposal,
and they should be documented here. There should be some explanation for why the disadvantage is
acceptable, however - just like in this example.*

## Alternatives

*This is where alternative solutions could be listed. There's almost always another way to do things
and this section gives you the opportunity to highlight why those ways are not as desirable.*

## Security Considerations

*Some proposals may have some security aspect to them that was addressed in the proposed solution.
This section is a great place to outline some of the security-sensitive components of your
proposal, such as why a particular approach was (or wasn't) taken.*

## Unstable Prefixes

*Detail any unstable prefixes or features introduced by this proposal.*

### Unstable Endpoints

`/_matrix/client/unstable/uk.tcpip.msc0000/example/{param}` would be necessary for the new methods
allowed when unstable capability is advertised by the server.

### Unstable Client Capability

The client capability `m.example_capability` should use this prefix until stable:

```json
{
    "capabilities": {
        "tld.name.msc0000.example_capability": {
            "enabled": true
        }
    }
}
```

### Unstable Client Features

The client feature `uk.tcpip.msc0000` should be advertised on the `/_matrix/client/versions`
endpoint when the new methods are accepted on the unstable endpoint.

```json
{
    "unstable_features": {
        "tld.name.msc0000": true
    },
    "versions": [
        "v1.11"
    ]
}
```

Once merged, homeservers can communicate supporting this stable feature until it is written into
the spec using the stable feature `tld.name.msc0000.stable`.

## Dependencies

*List any dependencies on other MSCs or external factors.*

This MSC builds on MSCxxxx, MSCyyyy and MSCzzzz (which at the time of writing have not yet been
accepted into the spec).
