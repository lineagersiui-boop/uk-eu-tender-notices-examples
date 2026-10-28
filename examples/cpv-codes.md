# CPV codes used in the examples

CPV (Common Procurement Vocabulary) is the EU classification that UK and EU notices use to say what is being bought. Codes have 8 digits, sometimes followed by a check digit (`72000000-5`); the check digit is ignored.

**Trailing zeros mean the whole branch.** The Actor drops trailing zeros and matches by prefix, so `45210000` matches every code that starts with `4521`. A notice matches if any of its CPV codes starts with any of your prefixes. You can also type the prefix directly, such as `72` or `4521`.

## input-it-services-uk.json
| Code | Name | Matches, for example |
|---|---|---|
| `72000000` | IT services: consulting, software development, Internet and support | `72230000` Custom software development services, `72400000` Internet services |
| `48000000` | Software package and information systems | `48100000` Industry specific software package |

With `"countries": ["GB"]`, only Find a Tender and Contracts Finder are queried.

## input-construction-eu.json
| Code | Name | Matches, for example |
|---|---|---|
| `45210000` | Building construction work | `45214200` Construction work for school buildings |
| `45300000` | Building installation work | `45310000` Electrical installation work, `45312100` Fire-alarm system installation work |
| `71240000` | Architectural, engineering and planning services | `71240000` itself and its sub-codes |

For all construction work use `45000000` (or `45`) instead; it's much broader.

`"minValue": 500000` with `"includeNoValue": true` keeps notices with an estimated value of at least 500,000 in their own currency (usually EUR on TED), plus notices without a published value, which are common on TED. Set `includeNoValue` to `false` to see only notices with a published value.

The countries `DE`, `FR`, `IE` and `NL` are EU codes, so only TED is queried.

The code names above are the official English CPV names. The full list is published by the EU Publications Office.
