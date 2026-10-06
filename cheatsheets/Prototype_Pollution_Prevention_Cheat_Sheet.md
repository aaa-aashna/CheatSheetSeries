# Prototype Pollution Prevention Cheat Sheet

## Introduction

Prototype pollution occurs when attacker-controlled property names or paths cause an application to add or modify properties on an object's prototype. This can change the behavior of objects that inherit from that prototype and may lead to security-impacting application behavior.

The two phases are:

- **Pollution:** attacker-controlled input reaches a prototype or a property assignment path that can modify one.
- **Exploitation:** application code later reads the polluted property and behaves unexpectedly.

The most important defenses are to avoid dynamic writes from untrusted keys, validate input against an allowlist or schema, and use `Map`, `Set`, or null-prototype objects where appropriate. See [MDN's prototype pollution guidance](https://developer.mozilla.org/en-US/docs/Web/Security/Attacks/Prototype_pollution) for the underlying JavaScript behavior.

## Suggested protection mechanisms

### Avoid dynamic property writes with untrusted keys

Do not pass attacker-controlled property names into recursive merge functions, path setters, or code that performs dynamic assignments such as `obj[key1][key2] = value`.

Explicitly reject the property segments `__proto__`, `constructor`, and `prototype` when they are not part of a narrowly defined allowlist. Schema validation should reject unexpected properties rather than attempting to sanitize arbitrary object structures.

### Use Map or Set for dynamic collections

When data is being used as a collection of keys or key-value pairs, prefer `Map` or `Set` instead of a plain object:

```javascript
const options = new Map();
options.set("spaces", 1);

const spaces = options.get("spaces");
```

Map entries are separate from object properties, so a polluted `Object.prototype` does not change the value returned by `Map.prototype.get()` for an entry that was explicitly stored in the map.

### Use null-prototype objects when an object is required

When an object must be used for dynamically keyed data, create it without a prototype:

```javascript
const obj = Object.create(null);
```

Object initializer syntax can also create a null-prototype object:

```javascript
const obj = { __proto__: null };
```

The `__proto__: null` syntax above sets the prototype during object creation. It is different from assigning to the deprecated `Object.prototype.__proto__` accessor.

Freezing built-in prototypes with [`Object.freeze()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/freeze) prevents adding or removing their properties and makes existing data properties non-writable. Freezing is shallow: objects referenced by those properties remain mutable unless separately frozen. Test compatibility first, because libraries that modify built-in prototypes can break.

[`Object.seal()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/seal) only prevents adding or removing properties and changing their configuration; existing writable property values can still change. Do not rely on sealing to prevent those modifications.
Null-prototype objects do not inherit properties from `Object.prototype`, including the `__proto__` accessor.

### Validate object properties before use

Treat parsed JSON and other objects containing untrusted data as untrusted even when they look like ordinary configuration objects.

- Validate input with a schema and reject unexpected properties.
- Define default values on the object itself rather than relying on inherited properties.
- When checking a property that may come from untrusted input, prefer an own-property check such as `Object.hasOwn()` where appropriate.
- Prefer `Object.keys()` or `for...of` over `for...in` when iterating over untrusted object data.

A JSON object containing a `__proto__` key is not itself prototype pollution. The risk appears when later operations such as `Object.assign()` or other implicit property writes use that value in a way that invokes the prototype setter.

### Consider locking down built-in objects

For high-integrity environments, freezing built-in objects can reduce the ability of code to modify shared prototypes. This is a defense-in-depth measure and may be incompatible with libraries that intentionally modify built-in objects.

### Node.js configuration flag

In Node.js, the `--disable-proto` option can remove or disable the `Object.prototype.__proto__` accessor:

```text
node --disable-proto=delete app.js
node --disable-proto=throw app.js
```

This is defense in depth. It does not prevent every form of prototype pollution because the `constructor.prototype` path remains available.

## References

Credit to [Gareth Hayes](https://garethheyes.co.uk/) for providing the original protection guidance [in this comment](https://github.com/OWASP/ASVS/issues/1563#issuecomment-1470027723).

## References

- [MDN: JavaScript prototype pollution](https://developer.mozilla.org/en-US/docs/Web/Security/Attacks/Prototype_pollution)
- [Node.js: Command-line API](https://nodejs.org/download/release/v26.5.1/docs/api/cli.html)
- [MDN: JavaScript prototype pollution](https://developer.mozilla.org/en-US/docs/Web/Security/Attacks/Prototype_pollution)
- [MDN: Object.create()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/create)
- [Node.js documentation: --disable-proto](https://nodejs.org/dist/latest/docs/api/all.html#--disable-protomode)
