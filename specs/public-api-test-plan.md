# restful-api.dev Public API Test Plan

## Application Overview

Comprehensive automation plan for the documented public object API at https://api.restful-api.dev. The public endpoint inventory is GET /objects, GET /objects/{id}, POST /objects, PUT /objects/{id}, PATCH /objects/{id}, and DELETE /objects/{id}. All scenarios start from a fresh/independent state, use HTTPS and JSON where required, create their own uniquely named test records, and clean up any records they create. Read-only checks may use IDs discovered at runtime from GET /objects; do not mutate the site's predefined sample objects. Validate HTTP status, JSON parseability, response schema and persistence as specified below. The documentation says GET /objects returns a limited predefined sample set and supports repeated id query parameters; POST accepts a name and flexible JSON data; PUT replaces an object's data; PATCH modifies supplied fields; DELETE removes an object. CORS is documented as enabled. Authenticated collection routes (/collections...), /register, /login, JWT/API-key behavior, and authenticated-only response-status overrides are outside this public API plan. Treat undocumented exact error bodies/statuses as exploratory unless a scenario specifies a conventional REST expectation; record actual results and raise discrepancies rather than weakening documented success assertions.

## Test Scenarios

### 1. Public object retrieval

**Seed:** `tests/seed.spec.ts`

#### 1.1. List the predefined public objects

**File:** `tests/public-api/get-objects.spec.ts`

**Steps:**
  1. From a fresh test context, send HTTPS GET https://api.restful-api.dev/objects without query parameters.
    - expect: The request completes without authentication and returns a successful 2xx response.
    - expect: The response is valid JSON containing an array of objects.
    - expect: Each returned item has a non-empty string id, a string name, and a data property; data may be an object, array, scalar, or null as allowed by the flexible schema.
    - expect: The response contains the documented limited predefined sample set, not a promise to list objects created by other users. Do not assert globally mutable object counts or ordering.

#### 1.2. Filter the public list by one or multiple IDs

**File:** `tests/public-api/get-objects-by-id-query.spec.ts`

**Steps:**
  1. GET /objects and select two distinct IDs from the returned sample data at runtime.
    - expect: The initial response is successful and provides two usable sample IDs.
  2. Send GET /objects?id=<first-id> and then GET /objects?id=<first-id>&id=<second-id>, repeating the id query parameter as documented.
    - expect: Each request returns a successful JSON array containing only the requested existing objects.
    - expect: The single-ID response contains only the first ID; the repeated-ID response contains exactly the two selected IDs, irrespective of order.
    - expect: No unrequested object appears.
  3. Send GET /objects?id=<unknown-id> using a generated ID that is not present in the sample list.
    - expect: The request does not return objects outside the specified ID filter; for an unknown-only filter the expected result is an empty array.

#### 1.3. Retrieve a public object by ID

**File:** `tests/public-api/get-object-by-id.spec.ts`

**Steps:**
  1. GET /objects, select one item from the returned sample array, and save its complete response representation.
    - expect: A sample object with a string id is available for this read-only test.
  2. Send GET /objects/<selected-id>.
    - expect: The request succeeds and returns a JSON object, not an array.
    - expect: The response id, name, and data match the selected item from GET /objects.
    - expect: The endpoint does not require authentication.

#### 1.4. Handle missing and malformed object IDs

**File:** `tests/public-api/get-object-invalid-id.spec.ts`

**Steps:**
  1. Send GET /objects/<generated-nonexistent-id> using a valid-looking ID that was not returned by the sample list or created by this test.
    - expect: The API returns a client-error/not-found response rather than a successful object response, consistent with a conventional missing-resource result.
    - expect: The response is parseable in the documented API error format if one is returned; record the actual body because the public endpoint docs do not define an exact error schema.
  2. Send GET /objects/ with an empty path ID if the HTTP client can issue it without normalizing the URL, and send a safely URL-encoded malformed ID (for example a space or reserved character).
    - expect: The server handles invalid path input without an unhandled server error or leaking implementation details.
    - expect: Record exact status/body for behavior not explicitly specified in the docs.

### 2. Public object creation and replacement

**Seed:** `tests/seed.spec.ts`

#### 2.1. Create an object with the documented JSON payload

**File:** `tests/public-api/post-object.spec.ts`

**Steps:**
  1. Generate a unique name and POST /objects with Content-Type: application/json and a payload containing name plus a nested data object with representative string, number, boolean, and null values.
    - expect: The request is accepted without authentication and returns a successful 2xx response with valid JSON.
    - expect: The response contains a newly assigned non-empty string id, the submitted name, the submitted data values, and a createdAt timestamp as shown by the documented response example.
    - expect: The response ID is unique for this create operation.
  2. GET /objects/<created-id> and compare it with the create response.
    - expect: The created resource is retrievable and persisted with matching id, name, and data.
  3. In a finally/cleanup step, DELETE /objects/<created-id> and confirm cleanup.
    - expect: The test-created record is removed even if an earlier assertion fails; no predefined sample record is deleted.

#### 2.2. Create objects with supported flexible data shapes

**File:** `tests/public-api/post-object-flexible-data.spec.ts`

**Steps:**
  1. In independent create-and-cleanup iterations, POST valid objects whose data values are respectively an empty object, an array, and null; include a unique non-empty name and JSON content type each time.
    - expect: Each supported JSON shape is accepted as valid JSON data, and the response preserves the supplied shape and value.
    - expect: Each response contains an assigned id and can be read back using GET /objects/{id}.
    - expect: Delete every object created by this scenario in cleanup.

#### 2.3. Validate malformed and incomplete create requests

**File:** `tests/public-api/post-object-validation.spec.ts`

**Steps:**
  1. Independently submit POST /objects with malformed JSON, a non-JSON body, and a JSON body missing the documented name field; also try an empty name as a boundary case.
    - expect: Malformed JSON and unsupported content types are rejected with a non-2xx client-error response rather than creating a resource.
    - expect: For missing/empty name, verify and record whether the API rejects the input or applies documented server behavior; because the public docs do not specify validation rules, flag acceptance without asserting an undocumented exact error code.
    - expect: No rejected request creates a retrievable object.

#### 2.4. Replace an existing test object with PUT

**File:** `tests/public-api/put-object-replace.spec.ts`

**Steps:**
  1. Create a uniquely named test object using POST /objects; retain its id and original fields.
    - expect: A test-owned object is created successfully and is available for the replacement check.
  2. PUT /objects/<created-id> with Content-Type: application/json and a replacement payload containing a changed name and a different data object with a marker field omitted from the original payload.
    - expect: The request succeeds with valid JSON and returns the same id, the replacement name and replacement data, plus an updatedAt timestamp as in the documentation.
    - expect: A follow-up GET returns the replacement values and confirms that PUT replaced the prior representation rather than merely adding fields; fields omitted by the replacement are absent if the API defines full replacement.
  3. PUT /objects/<generated-nonexistent-id> with a valid replacement body; then DELETE the created test object in cleanup.
    - expect: The API does not silently create a new resource for a PUT to an unknown ID unless that behavior is explicitly documented; record actual response and ensure the test-owned object is cleaned up.
    - expect: The scenario does not modify any predefined sample object.

### 3. Public partial update and deletion

**Seed:** `tests/seed.spec.ts`

#### 3.1. Patch selected fields without losing other fields

**File:** `tests/public-api/patch-object.spec.ts`

**Steps:**
  1. Create a test-owned object with a unique name and a data object containing at least two distinguishable fields.
    - expect: The object is created successfully and the initial state is recorded for comparison.
  2. PATCH /objects/<created-id> with Content-Type: application/json and a body containing only a changed name.
    - expect: The request succeeds, returns the same id and updated name, and returns updatedAt as shown in the documented response.
    - expect: A subsequent GET shows the changed name while the original data object remains unchanged.
  3. PATCH the same object with a body containing only a changed data value (and no name), then GET it again.
    - expect: The supplied field is updated and fields omitted from the patch remain intact, consistent with partial-update semantics.
    - expect: The object remains retrievable; delete it in a finally/cleanup step.

#### 3.2. Handle invalid PATCH payloads and IDs

**File:** `tests/public-api/patch-object-validation.spec.ts`

**Steps:**
  1. Send PATCH /objects/<generated-nonexistent-id> with a valid JSON partial body, then PATCH a test-owned object with malformed JSON and an empty JSON object.
    - expect: Unknown IDs and malformed payloads do not produce a successful update to an unrelated object.
    - expect: Malformed JSON is rejected with a non-2xx client-error response.
    - expect: The empty-patch behavior is recorded as a boundary case because the docs do not specify whether it is a no-op or validation error.
    - expect: Delete any object created for this scenario in cleanup.

#### 3.3. Delete a test-owned object and verify removal

**File:** `tests/public-api/delete-object.spec.ts`

**Steps:**
  1. Create a uniquely named object with POST /objects and retain the returned ID.
    - expect: The resource is created successfully and is owned by this test.
  2. DELETE /objects/<created-id>.
    - expect: The request succeeds with a JSON response containing a deletion confirmation message, as shown in the docs.
    - expect: The confirmation identifies the deleted ID.
  3. GET /objects/<deleted-id>, then repeat DELETE /objects/<deleted-id>.
    - expect: The deleted object is no longer retrievable and is not returned as an existing object.
    - expect: The repeated delete is handled as a missing-resource/client-error result or documented idempotent response; record exact behavior because the public docs do not define the repeated-delete status/body.

### 4. Public API protocol and robustness

**Seed:** `tests/seed.spec.ts`

#### 4.1. Verify HTTPS and browser CORS access

**File:** `tests/public-api/https-and-cors.spec.ts`

**Steps:**
  1. From a browser context whose origin is https://restful-api.dev, issue a cross-origin fetch to https://api.restful-api.dev/objects and read the response body.
    - expect: The API is served over HTTPS without a downgrade to plain HTTP.
    - expect: The browser can read the successful JSON response, demonstrating the documented CORS support.
    - expect: The response includes an Access-Control-Allow-Origin value that permits the requesting origin (or wildcard where appropriate).
  2. From the same origin, issue a JSON POST preflighted request using OPTIONS behavior generated by the browser, then create and delete a uniquely named test object.
    - expect: The browser preflight succeeds and the actual POST is not blocked by CORS; required methods and Content-Type are permitted.
    - expect: The created resource is removed during cleanup.

#### 4.2. Check unsupported methods do not mutate public resources

**File:** `tests/public-api/public-endpoint-methods.spec.ts`

**Steps:**
  1. Using a generated ID that is not a sample or test-owned record, send an unsupported method such as PUT or PATCH without a valid target/body to the list endpoint /objects.
    - expect: Unsupported method handling is a controlled 4xx response (typically 405) or other documented rejection, not an unhandled server failure.
    - expect: No resource is created or modified as a side effect; record behavior if the API does not specify an exact status.

#### 4.3. Exercise public endpoint error paths safely

**File:** `tests/public-api/public-endpoint-error-status.spec.ts`

**Steps:**
  1. For each public method, send one safe invalid-input request: GET an unknown ID, POST invalid JSON, PUT an unknown ID, PATCH malformed JSON, and DELETE an unknown ID.
    - expect: Each request returns a valid HTTP response with a non-2xx status for invalid input or missing resources, rather than timing out or returning an unrelated successful resource.
    - expect: The API does not disclose stack traces, internal hostnames, credentials, or other sensitive implementation details in error bodies.
    - expect: Exact error statuses and body shapes are captured as baselines where they are not defined by public documentation.
