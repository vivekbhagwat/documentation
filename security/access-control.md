# Access Policies

[Aidbox](https://www.health-samurai.io/aidbox) provides a flexible model to customize request authorization rules. User is allowed to declare a set of checks for all incoming requests. If incoming request satisfies those checks, it's considered authorized and being processed further. Otherwise the request is denied and client gets `403 Unauthorized`. Such checks are declared with AccessPolicy resource.

AccessPolicy resource has following structure:

```yaml
resourceType: AccessPolicy
description: policy description text

# type of evaluation engine
engine: allow | sql | json-schema | complex

# JSON Schema for `json-schema` engine
schema: {}

# SQL string for `sql` engine
sql:
  query: "SELECT true FROM ..."

# References to either Client, User or Operation resources
link:
- { resourceType: Operation, id: op-id }
- { resourceType: User, id: user-1 }
- { resourceType: Client, id: client-1 }
```

It supports three evaluation modes \(engines\): SQL, JSON Schema and Allow. Evaluation engine specifies how checks are expressed: with a SQL statement, with a [JSON Schema](https://json-schema.org/), or as a list of allowed endpoints.

Aidbox evaluates AccessPolicy against a **request object**, a data structure representing incoming HTTP request. It has following structure:

```yaml
# Request body (for PUT or POST)
body: null

# Parsed query-string params merged with URL params extracted by routing engine
params: {__debug: policy, type: Patient}

# Request method (get/post/put/delete)
request-method: get

# Request URI (no query string)
uri: /fhir/Patient

# Object containing current User resource (if any)
user:
  data:
    patient_id: 42
  email: foo@foo.com
  id: b66bc7c5-1a56-422f-8cf8-e64469135ce2
  meta:
    lastUpdated: '2018-10-31T13:43:18.566Z'
    tag:
    - code: updated
      system: https://aidbox.io
    versionId: '6'
  phone: 123-123-123
  resourceType: User

# Parsed JWT claims (if any)
jwt: {sub: xxxxxxx, jti: xxxxxxx, iss: aidbox,
  iat: 15409xxxxx, exp: 15409xxxxx}

# Query string
query-string: __debug=policy

# Client's IP address
remote-addr: 10.128.0.6

# Client resource (if any)
client:
  id: b4930671-410c-462b-8b12-23cdef91af0c
  meta:
    lastUpdated: '2018-10-31T13:38:08.982Z'
    tag:
    - code: created
      system: https://aidbox.io
    versionId: '4'
  resourceType: Client

# Request scheme (http or https)
scheme: http

# Request headers
headers: {x-original-uri: '/fhir/Patient?__debug=policy', origin: 'https://ui.aidbox.app',
  x-forwarded-host: xxxx.aidbox.app, host: xxxx.aidbox.app, user-agent: 'Mozilla/5.0
    (Macintosh; Intel Mac OS X 10_12_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/69.0.3497.100
    Safari/537.36', content-type: 'text/yaml, application/json', x-forwarded-port: '443',
  referer: 'https://ui.aidbox.app/', connection: close, accept: text/yaml, accept-language: 'en-US,en;q=0.9,ru;q=0.8',
  authorization: Bearer xxxxxx,
  x-forwarded-for: 10.128.0.6, accept-encoding: 'gzip, deflate, br', x-forwarded-proto: https,
  x-scheme: https, x-real-ip: 10.128.0.6}
```

### Request Authorization Logic

AccessPolicy instance can be linked to User, Client or Operation resources with `AccessPolicy.link` resource. If AccessPolicy has no links, it's considered as global policy. To authorize a request, Aidbox uses AccessPolicies linked to current request's User, Client and Operation plus all global policies. 

It iterates through them, evaluating each AccessPolicy against current request object. If some policy validates the request \(evaluation result is `true`\), the request considered authorized and Aidbox stops further policies evaluation. If all policies denied the request \(all of them evaluated to `false`\), then Aidbox denies such request and responds with `403 Unauthorized`.

{% hint style="info" %}
Performance consideration: Link you policy to User, Client or Operation to reduce number of checks per request! 
{% endhint %}

If no AccessPolicy instances exist for a box, all requests will be denied.

### Debugging

#### Using \_\_debug parameter

There is a special query-string parameter `__debug=policy` you can pass to every Aidbox request. It will toggle debug mode for Request Authorization Layer, and in this mode instead of actual response client will get a object containing:

* full request object;
* an array of existing AccessPolicies;
* evaluation result for every AccessPolicy \(under `AccessPolicy.evalResult`\).

#### Using /auth/test-policy Operation

You can use special operation  `POST /auth/test-policy` to design policy without creating AccessPolicy resource and for different users and clients. Do post on `/auth/test-policy` with simulated **request** attribute \(you can provide existing `user-id` and `client-id`  - aidbox will find and populate request\) and  temporal policy in **policy** attribute. Response contains result of evaluated  policy.

```yaml
POST /auth/test-policy

-- simulate request document
request:
  uri: '/Patient'
  request-method: get
  user:
    data: {role: 'admin'}
  -- or 
  -- user-id: 'user-1' -- aidbox will find user in database
  client:
    id: basic
    grant_types: ['basic']
  -- or 
  -- client-id: 'client-1' -- aidbox will find client in database
    
policy:
  engine: sql
  sql:
    query: 'SELECT {{user.role}} FROM {{!params.resource/type}}'

-- response

request:
  uri: /Patient
  request-method: get
  user: {role: admin}
  params: {resource/type: Patient}
operation:
  request:
  - get
  - {name: resource/type}
  action: proto.operations/search
  module: proto
  id: Search
-- original policy
policy:
  engine: sql
  sql: {query: 'SELECT {{user.role}} FROM {{!params.resource/type}}'}
-- result of policy evaluation
result:
  eval-result: false
  query: ['SELECT ? FROM "patient"', admin]
```

### JSON Schema Engine

JSON Schema engine allows to put JSON Schema under `AccessPolicy.schema` element and this schema will be used to validate request object. Currently supported JSON Schema version is **draft-07**.

#### Example

The following policy requires presence of `request.user` attribute \(only authenticated requests are allowed\):

```yaml
resourceType: AccessPolicy
description: Allow
engine: json-schema
schema:
  type: object
  required: ["user"]
```

### SQL Engine

SQL Engine executes SQL statement and uses its result as an evaluation result. The SQL statement should return single row with just one column, i.e.:

```sql
SELECT true FROM patient WHERE id = {{jwt.patient_id}} LIMIT 1;
```

SQL statement can include interpolations in double curly braces, like in example above. String inside curly braces will be used as a path to get value from the request object.

#### Example

Assuming that User has a reference to Practitioner resource through `User.data.practitioner_id` element, the following policy allow requests only to `/fhir/Patient/<patient_id>` URLs, and only for those patients, who has `Patient.generalPractitioner` element referencing same practitioner as a current User. In other words, User as a Practitioner is only allowed to see patients who referencing him with `Patient.generalPractitioner`.

```yaml
sql:
  query: |
    SELECT
      {{user}} IS NOT NULL
      AND {{user.data.practitioner_id}} IS NOT NULL
      AND {{uri}} LIKE '/fhir/Patient/%'
      AND resource->'generalPractitioner' @>
    jsonb_build_array(jsonb_build_object('resourceType',
        'Practitioner', 'id', {{user.data.practitioner_id}}::text))
      FROM patient WHERE id = {{params.resource/id}};
engine: sql
id: practitioner-only-allowed-to-see-his-patients
resourceType: AccessPolicy
```

#### Interpolation rules

In your SQL query you can parametrise with attributes from request object using  `{{path}}` syntax. For example to get role from user `{data: {role: 'admin'}}` you can  write `{{user.data.role}}`. Parameter expressions are escaped by default to protect from SQL injection. If you want to make dynamical queries \(parametrise table name for example\) you have to use `{{!path}}` syntax. For example expression `SELECT true from {{!params.resource/type}} limit 1` with params = `{resource/type: "Patient"}` will be transformed into `SELECT true from "patient".` Such identifier names are double quoted with and lowercased by default.

### Allow Engine

Allow Engine constantly evaluates to `true` regardless the content of the request object.

### Complex Engine

Complex engine provides ability to include several checks into a single policy and apply  "AND" / "OR" operator on results. It's allowed to use any policy engine to define a check, you can even use "complex" engine to get sub-expression:

```yaml
and:
  - { engine: "sql", sql: { query: "select true" } }       # Check 1
  - engine: complex
    or:
      - { engine: "sql", sql: { query: "select false" } }  # Check 2
      - { engine: "sql", sql: { query: "select false" } }  # Check 3

engine: complex
id: complex-example-1
resourceType: AccessPolicy
```

Policy in the example above represents following logical expression: `check 1 AND (check 2 OR check 3)`. It's forbidden to have both `and` and `or` keys on the same level.

#### Example

Let's split SQL Policy example into two separate checks and combine them with AND operator:

```yaml
and:
  # Check: request.user && request.user.data.practitioner_id are required
  - engine: json-schema
    schema:
      type: object
      required: ["user"]
      properties:
        user:
          type: object
          required: ["data"]
          properties:
            data:
              type: object
              required: ["practitioner_id"]

  # Check: Current practitioner is patient's generalPractitioner
  - engine: sql
    sql:
      query: |
        SELECT
          {{uri}} LIKE '/fhir/Patient/%'
          AND resource->'generalPractitioner' @>
        jsonb_build_array(jsonb_build_object('resourceType',
            'Practitioner', 'id', {{user.data.practitioner_id}}::text))
          FROM patient WHERE id = {{params.resource/id}};
engine: complex
id: complex-example-2
resourceType: AccessPolicy
```



