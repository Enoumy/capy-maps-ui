# Capy Maps

Hosted on [capymaps.com](https://capymaps.com) via github pages.

The site is roughtly equivalent to:

```sh
curl https://panynj.gov/bin/portauthority/ridepath.json | jq '.results | map({(.consideredStation|tostring) : .})|add|map_values(.destinations | map({(.label|tostring) : .}) | add | map_values(.messages | map({target : .target, eta: .arrivalTimeMessage}) | group_by(.target)[] | {(.[0].target|tostring) : [.[] | .eta]}))'```

for all path stations. Here is the jq query with nicer, non-bash formatting:

```jq
.results
| map({( .consideredStation | tostring ) : . })
| add
| map_values(
  .destinations
  | map({( .label | tostring ) : . })
  | add
  | map_values(
    .messages
    | map({ target : .target, eta: .arrivalTimeMessage })
    | group_by(.target)[]
    | {( .[0].target | tostring ) : [ .[] | .eta ]}))
```
