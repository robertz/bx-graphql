# Code-first schema: plan

Generate SDL and wire resolvers from annotated BoxLang classes, alongside the
hand-written `.graphqls` + `{Type}Resolver.bx` convention that exists today.
Target version: 0.2.0. Nothing that works in 0.1.1 changes behaviour.

## Shape of the feature

```javascript
graphQLService = bxGraphQL( {
	modelPaths      : [ "models/beans" ],       // classes marked @graphqlType / @graphqlInput
	servicePaths    : [ "models/services" ],    // classes with @graphqlQuery / @graphqlMutation functions
	instanceFactory : ( classPath, meta ) => wirebox.getInstance( classPath ),   // optional
	schemaPaths     : [ "graphql/schema" ],     // now optional
	resolverBasePackage : "models.resolvers"    // now optional
} )
```

The generator turns class metadata into an SDL string, which is parsed and
merged into the same `TypeDefinitionRegistry` as the hand-written files. SDL
stays the single intermediate form: it can be printed, diffed and extended by
hand (`extend type Widget { slug: String }` plus a `WidgetResolver.bx`).

### Beans

```javascript
/** A thing we sell. */
@graphqlType
@graphqlInput
class accessors="true" {
	@graphqlId
	property name="id" type="numeric";
	property name="name" type="string" required="true";
	@graphqlField( "[Tag!]!" )
	property name="tags" type="array";
	@graphqlIgnore
	property name="passwordHash";
	@graphqlReadOnly
	property name="createdAt" type="date";
}
```

| Annotation | On | Meaning |
|---|---|---|
| `@graphqlType` / `@graphqlType( "Name" )` | class | emit `type Name` (default: the class's simple name) |
| `@graphqlInput` / `@graphqlInput( "Name" )` | class | emit `input Name` (default: `{SimpleName}Input`) |
| `@graphqlId` | property | field type `ID` |
| `@graphqlField( "SDL type" )` | property | explicit type, verbatim |
| `@graphqlIgnore` | property | in neither type nor input |
| `@graphqlReadOnly` | property | in the type, not the input |

Only annotated classes are exposed; within one, every property is unless
ignored. `required="true"` adds `!`. Doc comments become SDL descriptions.

Type mapping:

| BoxLang `type` | GraphQL |
|---|---|
| `string` | `String` |
| `boolean` | `Boolean` |
| `integer` | `Int` |
| `numeric` | `Float` |
| `date`, `datetime` | `DateTime` (phase 1 scalar) |
| simple name of another annotated class | that type (its input name inside an input) |
| `array`, `struct`, `any`, anything else | needs `@graphqlField`, else `BxGraphQL.ConfigurationException` at startup naming the class and property |

### Operations

```javascript
class {
	/** One widget by id. */
	@graphqlQuery
	Widget function widget( required numeric id graphqlType="ID", struct context ) {}

	@graphqlMutation( "renameWidget" )
	@graphqlReturns( "[Widget!]!" )
	array function rename( required WidgetInput input ) {}
}
```

- `@graphqlQuery` / `@graphqlMutation` on a public function adds a field to
  `Query` / `Mutation`; an annotation value overrides the field name.
- Arguments come from the parameters: name, mapped type, `required` → `!`,
  default value → SDL default. Inline `graphqlType="…"` on a parameter
  overrides its type.
- Parameters named `context` or `env` are injected (as resolvers get them
  today) and left out of the SDL.
- Return type comes from the declared return type through the same mapping;
  `@graphqlReturns( "…" )` overrides it and is required for `array`/`any`/`struct`.
- A parameter typed as an `@graphqlInput` class receives an instance of that
  class built from the argument; one typed `struct` receives the struct.
- No CRUD is generated from a bean alone: a bean doesn't say how it is loaded
  or saved. (Possible later: `@graphqlCrud( "WidgetService" )` against a
  `get/list/save/delete` contract.)

### Wiring and precedence

- Each generated operation gets its own fetcher through
  `TypeRuntimeWiring.dataFetcher( fieldName, proxy )`, bound to one service
  instance and method. Service instances are built once, at startup, by
  `instanceFactory` (default: `new`).
- Everything else still goes through the type's `defaultDataFetcher`
  (`DataFetcherAdapter`): `{Type}Resolver.bx` method, else property lookup.
- The same field declared twice (generated and hand-written, or by two
  services) fails at startup, naming both sources.
- `Query` / `Mutation`: emitted as `type` when no hand-written file defines
  them, `extend type` when one does.

## Verified on BoxLang 1.19.0

- Annotation values need parentheses: `@graphqlField( "[Tag!]!" )` compiles,
  `@graphqlField "[Tag!]!"` doesn't.
- `getClassMetadata( "relative/path/Widget" )` reads a class's metadata
  without instantiating it, and (like `obj.$bx.meta`) gives
  `annotations`, `documentation`, `properties[].annotations`, and
  `functions[].{ access, returnType, annotations, documentation, parameters[].{ name, type, required, default, annotations } }`.
- `getMetaData()` is not usable: with bx-compat-cfml installed it flattens
  annotations into the top level and drops `annotations`.
- Inline parameter annotations (`numeric id graphqlType="ID"`) land in
  `parameters[].annotations`.
- **A class instance returned from a resolver resolves every field to null,
  silently.** `PropertyDataFetcher` can't read a BoxLang class, with
  accessors or without; structs work. Phase 2 fixes this and it is a
  prerequisite for beans.

Still to verify: whether `getClassMetadata` takes the same
relative-to-this-file path `new` needs (it took a path relative to the running
script in the probe), and the `Coercing` proxy for scalars.

## Phases

Each phase is its own commit with specs, and leaves the suite green.

### 1. Custom scalars
- `scalars` setting: `{ Name : { serialize, parseValue, parseLiteral? } }` of
  closures, wrapped as `graphql.schema.Coercing` with `createDynamicProxy` and
  registered on the `RuntimeWiring`.
- Built in, registered only when the schema declares them: `DateTime`
  (ISO-8601 instant, via `java.time`) and `Long`.
- Specs: round trip as output, as variable, as literal; bad input gives a
  GraphQL error, not an exception.

### 2. Class instances as sources
- `DataFetcherAdapter` fallback, when the source is a class instance:
  `get{Field}()` if it exists, else the public property, else null. Structs
  and Java objects keep `PropertyDataFetcher`.
- Specs: bean with accessors, bean with a hand-written getter, nested bean,
  array of beans, struct unchanged.

### 3. Shared class loading
- Move `toRelativePath()` out of `DataFetcherAdapter` into `ClassLoaderHelper`
  with `metadataOf( absolutePath )` and `instantiate( absolutePath )`.
- Directory scan: every `.bx` under a configured path, recursively.
- Settings: `modelPaths`, `servicePaths`, `instanceFactory`; `schemaPaths` and
  `resolverBasePackage` become optional, but at least one schema source is
  required.

### 4. Types and inputs from beans
- `SchemaGenerator.bx`: metadata → SDL string for `type` and `input`,
  including descriptions; `TypeMapper.bx` holds the mapping table and the
  error for unmappable types.
- `GraphQLService.getGeneratedSDL()` returns what was generated;
  `printSchema()` returns the whole executable schema through `SchemaPrinter`.
- Specs: snapshot of generated SDL for a fixture bean set; each annotation;
  unmappable type error; name collision with a hand-written type; a bean type
  extended by hand with a resolver.

### 5. Operations from services
- Generator emits `Query` / `Mutation` fields from annotated functions.
- `MethodDataFetcher.bx`: holds the instance, method name and parameter list;
  builds the argument collection, injects `context` / `env`, hydrates input
  beans.
- Specs: query with scalar args and defaults, mutation with an input bean,
  `context` injection, `instanceFactory` used, private and unannotated
  functions not exposed, duplicate field error, `type` vs `extend type`.

### 6. Docs and release
- README: "Code-first schema" section, annotation and mapping tables, the
  settings table, and the "Not yet implemented" list.
- `ModuleConfig.bx` and `box.json` to 0.2.0.

## Out of scope
- Interface / union `TypeResolver` wiring, enums from classes.
- Structured errors (`{ message, path }` instead of message strings). Worth
  doing next: the reader parses the strings back apart in `GraphQLRoutes.errorOf`.
- DataLoader batching for bean relations.
