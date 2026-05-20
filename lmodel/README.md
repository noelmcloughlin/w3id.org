# /lmodel/

Perma-id service for https://github.com/lmodel

## Maintainer

Noel McLoughlin (noel.mcloughlin AT gmail.com)
https://github.com/noelmcloughlin

## Smoke Test
```
#!/usr/bin/env bash
# Smoke test: lmodel/.htaccess routes for lmodel/oscal repo.
# Tests two things per case:
#   1. Local Apache returns expected status (and Location for redirects)
#   2. The Location target (raw GH URL) returns 200
set -u
BASE="http://localhost:18080"
REPO="oscal"
STEM="oscal"
RAW="https://raw.githubusercontent.com/lmodel/$REPO/main/project"
SRC="https://raw.githubusercontent.com/lmodel/$REPO/main/src/$STEM"

PASS=0
FAIL=0
FAILS=()

check() {
  local desc="$1" url="$2" accept="$3" want_status="$4" want_loc="$5"
  local out got_status got_loc
  out=$(curl -s -o /dev/null -D - -H "Accept: $accept" "$url")
  got_status=$(printf '%s' "$out" | awk 'NR==1{print $2}')
  got_loc=$(printf '%s' "$out" | awk 'tolower($1)=="location:"{print $2}' | tr -d '\r')

  local raw_ok="-"
  if [[ -n "$got_loc" && "$got_loc" == http* ]]; then
    raw_ok=$(curl -sI -o /dev/null -w "%{http_code}" -L "$got_loc")
  fi

  local ok=1
  [[ "$got_status" == "$want_status" ]] || ok=0
  if [[ -n "$want_loc" && "$got_loc" != "$want_loc" ]]; then ok=0; fi
  if [[ "$want_status" == 30* && -n "$got_loc" && "$raw_ok" != "200" ]]; then ok=0; fi

  if (( ok )); then
    PASS=$((PASS+1))
    printf "PASS  %-60s  %s  loc=%s  raw=%s\n" "$desc" "$got_status" "${got_loc:--}" "$raw_ok"
  else
    FAIL=$((FAIL+1))
    FAILS+=("$desc")
    printf "FAIL  %-60s  got=%s  want=%s  loc=%s  want_loc=%s  raw=%s\n" "$desc" "$got_status" "$want_status" "${got_loc:--}" "${want_loc:--}" "$raw_ok"
  fi
}

echo "## Conneg (bare repo) ##"
check "oscal turtle"         "$BASE/$REPO" "text/turtle"            303 "$RAW/rdf/$STEM.ttl"
check "oscal rdf+xml"        "$BASE/$REPO" "application/rdf+xml"    303 "$RAW/rdf/$STEM.rdf"
check "oscal owl+xml"        "$BASE/$REPO" "application/owl+xml"    303 "$RAW/rdf/$STEM.rdf"
check "oscal n-triples"      "$BASE/$REPO" "application/n-triples"  303 "$RAW/rdf/$STEM.nt"
check "oscal ld+json"        "$BASE/$REPO" "application/ld+json"    303 "$RAW/jsonld/$STEM.context.jsonld"
check "oscal schema+json"    "$BASE/$REPO" "application/schema+json" 303 "$RAW/jsonschema/$STEM.schema.json"
check "oscal shacl+turtle"   "$BASE/$REPO" "application/shacl+turtle" 303 "$RAW/shacl/$STEM.shacl.ttl"
check "oscal shex"           "$BASE/$REPO" "text/shex"              303 "$RAW/shex/$STEM.shex"
check "oscal yaml"           "$BASE/$REPO" "application/yaml"       303 "$SRC/schema/$STEM.yaml"
check "oscal text/yaml"      "$BASE/$REPO" "text/yaml"              303 "$SRC/schema/$STEM.yaml"
check "oscal html"           "$BASE/$REPO" "text/html"              303 "https://lmodel.github.io/$REPO"
check "oscal */*"            "$BASE/$REPO" "*/*"                    302 "https://lmodel.github.io/$REPO"
check "oscal unsupported"    "$BASE/$REPO" "application/zip"        406 ""

echo
echo "## Conneg (trailing slash) ##"
check "oscal/ turtle"        "$BASE/$REPO/" "text/turtle"           303 "$RAW/rdf/$STEM.ttl"
check "oscal/ html"          "$BASE/$REPO/" "text/html"             303 "https://lmodel.github.io/$REPO/"

echo 
echo "## Conneg (deep extensionless - class IRI) ##"
check "oscal/Catalog turtle""$BASE/$REPO/Catalog" "text/turtle"    303 "$RAW/rdf/$STEM.ttl"
check "oscal/Catalog html"   "$BASE/$REPO/Catalog" "text/html"      303 "https://lmodel.github.io/$REPO/Catalog"

echo
echo "## Schema routes ##"
check "schema yaml"          "$BASE/$REPO/schema/$STEM.yaml" "*/*"  303 "$SRC/schema/$STEM.yaml"
check "schema bare path"     "$BASE/$REPO/schema/anything"    "*/*" 303 "$SRC/schema/$STEMNyaml"

echo echo "## Core artifacts (extension) ##"
check ".ttl"        "$BASE/$REPO/$STEM.ttl"               "*/*" 303 "$RAW/rdf/$STEM.ttl"
check ".rdf"        "$BASE/$REPO/$STEM.rdf"               "*/*" 303 "$RAW/rdf/$STEM.rdf"
# .nt: no oscal.nt present in repo? Actually let me check - was listed in rdf? Only oscal.ttl. So expect 404 on raw.
check ".owl"         "$BASE/$REPO/$STEM.owl"               "*/*" 303 "$RAW/owl/$STEM.owl.ttl"
check ".jsonld"      "$BASE/$REPO/$STEM.jsonld"            "*/*" 303 "$RAW/jsonld/$STEM.jsonld"
check ".context.jsonld" "$BASE/$REPO/$STEM.context.jsonld" "*/*" 303 "$RAW/jsonld/$STEM.context.jsonld"
check "context.jsonld bare" "$BASE/$REPO/context.jsonld"  "*/*" 303 "$RAW/jsonld/$STEM.context.jsonld"
check "context.json bare"   "$BASE/$REPO/context.json"    "*/*" 303 "$RAW/jsonld/$STEM.context.jsonld"
check ".schema.json"        "$BASE/$REPO/$STEM.schema.json" "*/*" 303 "$RAW/jsonschema/$STEM.schema.json"
check ".graphql"            "$BASE/$REPO/$STEM.graphql"   "*/*" 303 "$RAW/graphql/$STEM.graphql"
check ".proto"              "$BASE/$REPO/$STEM.proto"     "*/*" 303 "$RAW/protobuf/$STEM.proto"
check ".shacl.ttl"          "$BASE/$REPO/$STEM.shacl.ttl" "*/*" 303 "$RAW/shacl/$STEM.shacl.ttl"
check ".shex"               "$BASE/$REPO/$STEM.shex"      "*/*" 303 "$RAW/shex/$STEM.shex"
check "linkml yaml"         "$BASE/$REPO/$STEM.merged.linkml.yaml" "*/*" 303 "$RAW/linkml/$STEM.merged.linkml.yaml"
check "prefixmap yaml"      "$BASE/$REPO/prefixmap/$STEM.yaml" "*/*" 303 "$RAW/prefixmap/$STEM.yaml"
check "sqlschema sql"       "$BASE/$REPO/$STEM.sql"       "*/*" 303 "$RAW/sqlschema/$STEM.sql"

echo
echo "## Non-core artifacts (302) ##"
check ".xlsx"        "$BASE/$REPO/$STEM.xlsx"          "*/*" 302 "$RAW/excel/$STEM.xlsx"
check ".java"        "$BASE/$REPO/Catalog.java"        "*/*" 302 "$RAW/java/Catalog.java"
check ".ts"          "$BASE/$REPO/$STEM.ts"            "*/*" 302 "$RAW/typescript/$STEM.ts"
check ".h"           "$BASE/$REPO/$STEM.h"             "*/*" 302 "$RAW/cpp/$STEM.h"
check ".csv"         "$BASE/$REPO/$STEM.csv"           "*/*" 302 "$RAW/csv/$STEM.csv"
check ".dbml"        "$BASE/$REPO/$STEM.dbml"          "*/*" 302 "$RAW/dbml/$STEM.dbml"
check ".er.md"       "$BASE/$REPO/$STEM.er.md"         "*/*" 302 "$RAW/erdiagram/$STEM.er.md"
check ".go"          "$BASE/$REPO/$STEM.go"            "*/*" 302 "$RAW/golang/$STEM.go"
check "golr yaml"    "$BASE/$REPO/golr/Catalog_config.yaml" "*/*" 302 "$RAW/golr/Catalog_config.yaml"
check ".dot.dot"     "$BASE/$REPO/$STEM.dot.dot"       "*/*" 302 "$RAW/graphviz/$STEM.dot.dot"
check "markdown-datadict" "$BASE/$REPO/markdown-datadict/$STEM.md" "*/*" 302 "$RAW/markdown-datadict/$STEM.md"
check "mermaid md"   "$BASE/$REPO/mermaid/Catalog.md"  "*/*" 302 "$RAW/mermaid/Catalof.md"
check "namespaces py" "$BASE/$REPO/namespaces/$STEM.namespaces.py" "*/*" 302 "$RAW/namespaces/$STEM.namespaces.py"
check "pandera py"   "$BASE/$REPO/pandera/${STEM}_pandera.py" "*/*" 302 "$RAW/pandera/${STEM}_pandera.py"
check ".plantuml"    "$BASE/$REPO/$STEM.plantuml"      "*/*" 302 "$RAW/plantuml/$STEM.plantuml"
check "sqla py"      "$BASE/$REPO/sqla/${STEM}_sqlalchemy.py" "*/*" 302 "$RAW/sqla/${STEM}_sqlalchemy.py"
check "sqlvalidation" "$BASE/$REPO/sqlvalidation/$STEM.sql" "*/*" 302 "$RAW/sqlvalidation/$STEM.sql"
check ".sssom.tsv"   "$BASE/$REPO/$STEM.sssom.tsv"     "*/*" 302 "$RAW/sssom/$STEM.sssom.tsv"
check "summary tsv"  "$BASE/$REPO/summary/$STEM.summary.tsv" "*/*" 302 "$RAW/summary/$STEM.summary.tsv"
check "terminusdb json" "$BASE/$REPO/terminusdb/$STEM.json" "*/*" 302 "$RAW/terminusdb/$STEM.json"
check ".tql"         "$BASE/$REPO/$STEM.tql"           "*/*" 302 "$RAW/typedb/$STEM.tql"
check "yaml dir"     "$BASE/$REPO/yaml/$STEM.yaml"     "*/*" 302 "$RAW/yaml/$STEM.yaml"
check "rust Cargo"   "$BASE/$REPO/rust/Cargo.toml"     "*/*" 302 "$RAW/rust/Cargo.toml"

echo
echo "## Deeper-pass fixes ##"
# .owl.ttl direct request now resolves to the same gen-project file
check ".owl.ttl direct" "$BASE/$REPO/$STEM.owl.ttl" "*/*" 303 "$RAW/owl/$STEM.owl.ttl"
# Bare <repo>/<stem>.yaml -> schema YAML
check "bare stem yaml"  "$BASE/$REPO/$STEM.yaml"    "*/*" 303 "$SRC/schema/$STEM.yaml"
# application/json conneg fallback -> JSON-LD context
check "json conneg"     "$BASE/$REPO" "application/json" 303 "$RAW/jsonld/$STEM.context.jsonld"
# Schema bare-path narrowed: no-extension still works, .json now falls through
check "schema bare narrow" "$BASE/$REPO/schema/anyname" "*/*" 303 "$SRC/schema/$STEM.yaml"

echo
echo "## Hyphenated repo: uco-core (single-hyphen STEM family) ##"
UCO_RAW="https://raw.githubusercontent.com/lmodel/uco-core/main/project"
UCO_SRC="https://raw.githubusercontent.com/lmodel/uco-core/main/src/uco_core"
check "uco-core jsonld"  "$BASE/uco-core/uco_core.jsonld" "*/*" 303 "$UCO_RAW/jsonld/uco_core.jsonld"
check "uco-core schema"  "$BASE/uco-core/schema/uco_core.yaml" "*/*" 303 "$UCO_SRC/schema/uco_core.yaml"
check "uco-core conneg ttl" "$BASE/uco-core" "text/turtle" 303 "$UCO_RAW/rdf/uco_core.ttl"
# Slash form -> [N] internal rewrite -> uco-core
check "uco/core slash"   "$BASE/uco/core" "text/html" 303 "https://lmodel.github.io/uco-core"

echo
echo "## Hyphenated repo: nist-ai-rmf (multi-hyphen STEM family) ##"
NAR_RAW="https://raw.githubusercontent.com/lmodel/nist-ai-rmf/main/project"
NAR_SRC="https://raw.githubusercontent.com/lmodel/nist-ai-rmf/main/src/nist_ai_rmf"
check "nist-ai-rmf jsonld" "$BASE/nist-ai-rmf/nist_ai_rmf.jsonld" "*/*" 303 "$NAR_RAW/jsonld/nist_ai_rmf.jsonld"
check "nist-ai-rmf conneg" "$BASE/nist-ai-rmf" "application/yaml" 303 "$NAR_SRC/schema/nist_ai_rmf.yaml"
# Alias: nist-ai-600-1 -> nist-ai-rmf via [N]
check "nist-ai-600-1 alias jsonld" "$BASE/nist-ai-600-1/nist_ai_rmf.jsonld" "*/*" 303 "$NAR_RAW/jsonld/nist_ai_rmf.jsonld"
check "nist-ai-600-1 alias html"   "$BASE/nist-ai-600-1" "text/html" 303 "https://lmodel.github.io/nist-ai-rmf"

echo 
echo "================================================"
echo "PASS=$PASS FAIL=$FAIL"
if (( FAIL > 0 )); then
  echo "Failures:"
  printf '  - %s\n' "${FAILS[@]}"
fi
```
