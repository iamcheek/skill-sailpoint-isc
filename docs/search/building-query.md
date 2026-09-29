# Building a Search Query

Source: https://documentation.sailpoint.com/saas/help/search/building-query.html

See [searchable-fields.md](searchable-fields.md) for which fields exist per index, and [README.md](README.md) for how field "levels" (first/second/nested) map to query syntax.

## Basic field:term queries

Format: `field:term`. **Do not put spaces in the field portion of your query or after the colon following the field** — `field: term` and `field :term` are both invalid.

```
displayName:john.smith
created:2018-06-01
status:ACTIVE
```

## Second-level fields

Dot notation: `secondLevelField.firstLevelField:term`

```
attributes.location:london
manager.name:amanda.ross
identityProfile.name:ID_Profile_1
requester.name:bob.dobbs
```

## Nested object queries

Fields that can occur multiple times per record (an identity's accounts, an identity's access items, an account activity's account requests, etc.) require the `@<nestedObject>(<field:term>)` syntax:

```
@accounts(source.name:Acme)
@access(privileged:true)
@apps(source.name:"Microsoft Entra ID")
@accountRequests(op:create)
@accountRequests(result.status:failed)
```

## Boolean operators

All-caps keywords `AND`, `OR`, `NOT`, or symbolic equivalents `&&`, `||`, `!`.

```
"sao paulo" OR london
attributes.location:"sao paulo" OR attributes.location:london

london AND amanda.ross
attributes.location:london AND manager.name:amanda.ross

amanda.ross NOT london
!manager.name:amanda.ross
```

## Wildcards

`*` matches any number of characters; `?` matches exactly one character. Wildcards only work with text fields.

```
name:j*n     # matches jon, jordan, jasmin, etc.
name:j?n     # matches jon, jin — single character only
*            # matches all data
```

## Range queries

`[value1 TO value2]` is inclusive; `{value1 TO value2}` is exclusive.

```
accountCount:[1 TO 4]
accountCount:{1 TO 4}
@accounts(created:[2018-03-01 TO 2018-03-31])
```

Relative date math is supported:

```
created:[now-1M TO now]
attributes.endDate:[now+1y TO now]
```

## Comparison operators

`>`, `<`, `>=`, `<=` work against dates and numbers.

```
appCount:>12
created:<=2015-02-04
```

## Exact matching (`.exact`)

Append `.exact` to a field name for case-sensitive matching and to enable special-character searches.

```
manager.name.exact:"amanda ross"
lastName.exact:Ross
attributes.location.exact:Aus*
@accounts(source.name.exact:"ad")
name.exact:"jon-smith"
name.exact:"TaxProject#2019"
```

## Existence checks (`_exists_`)

Finds records where a field is present/non-null. Only reliably matches fields whose values conform to the documented type/format for that field (see [searchable-fields.md](searchable-fields.md)).

```
_exists_:attributes.endDate
NOT _exists_:owner.name
NOT _exists_:manager.name
```

## Special characters and regex

Symbols like `#` or `+` cannot be searched directly — `?` (single-char wildcard) is commonly used as a substitute. For genuine pattern matching on special characters, wrap the pattern in forward slashes as a regular expression and combine with `.exact`:

```
attributes.lastname.exact:/.*-.*/
name.exact:/.*OU=AUSTIN_._GI.*OU=File Access*/
```

Inside a regex, use `.*` where you'd otherwise use `*` (any characters) and `.` where you'd use `?` (single character) — plain wildcard syntax doesn't apply inside `/.../`.

## Grouping with parentheses

Parentheses control evaluation order, same as any boolean expression:

```
attributes.location:london AND manager.name:amanda.ross AND (lastName:trent OR lastName:primrose)

attributes.location:london AND manager.name:amanda.ross AND lastName:(trent OR primrose)
```

## Notes / things NOT documented on this page

The building-query source page does **not** mention fuzzy matching (`~`), term boosting (`^`), or proximity search — these are standard Lucene/OpenSearch query-string features but are not called out as supported/tested in SailPoint's own docs. Don't assume they work; if you need them, prefer the DSL query mode (`queryType: "DSL"` in a `/search/v1` POST body — see [audit-events-searching.md](audit-events-searching.md)) where you have direct access to OpenSearch Query DSL instead of the simplified query-string syntax described here.

## Gotcha: manager name searches

Searching `manager.name:amanda.ross` returns **both** the identity named Amanda Ross **and** every identity that lists her as their manager — the field matches on either role. Narrow with additional filters if you only want one side of that relationship.
