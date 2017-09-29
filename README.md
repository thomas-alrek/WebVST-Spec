# WebVST Specification

Current version: *v0.0.1* 

This document defines the requirements that a plugin must meet in order to be compliant and interoperable with other WebVSTs.

This spec is a work in progres. Please submit issues and pull requests to provide feedback!

The keywords **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **RECOMMENDED**, **MAY** and **OPTIONAL** in this document are to be interpreted as described in [RFC2119](http://www.ietf.org/rfc/rfc2119.txt)

## Goals

The goals of the WebVST specification is to promote interoperability and reusability of code. There are a lot of web audio demos with amazing effects and signal processing only able to be used within the demo, are bound to the global `window` object, and have differing ways of creating composite audio nodes. 

Adherence to this specification allows the following:

* WebVSTs able to connect to and be connected by other WebVSTs and native AudioNodes
* Creation of composite audio nodes, comprised of several low-level AudioNodes, or other existing WebVSTs
* Allowing external libraries to manipulate WebVSTs
* Allow a more succinct way of creating WebVSTs with a library, rather than defining boilerplate for every WebVSTs created.
* Sharing and reusing code -- rather than rewriting audio components or signal processing algorithms from scratch, just use an existing component found on the NPM [registry](http://npmjs.com)

## Background

### package.json

First and foremost, WebVSTs are a subset of NPM packages. WebVSTs construction and manifests follow the NPM package spec, with additional properties and requirements; if you haven't already done so, it is recommended that you familiarize yourself with the [package.json spec](https://docs.npmjs.com/files/package.json).

### WebVST Types

There are three different types of WebVSTs currently: effect, generator, and analyzer, explained in more detail below.

#### Effect

Effect are WebVSTs that have an input and an output, and pass a signal from input to output and **SHOULD** alter the signal in some way.

#### Analyzer

Like effect WebVSTs, analyzer WebVSTs also have an input and an output, and pass a signal from input to output. They differ from effect WebVSTs as they **SHOULD NOT** alter the signal, but used to analyse the signal in some way.

#### Generator

Unlike both analyzer and effect WebVSTs, source WebVSTs *generate* a signal. They have a source node, and an output.
They accept a form of note input (either programatically or via MIDI), and provide a voice to the notes that can be chained into other WebVSTs. They act as the basic for software synthesizers.

## Manifest

In addition to all the properties required by NPM, WebVSTs **MUST** include a `web-vst` property in their `package.json` file -- this property indicates that this is a WebVST. The `web-vst` property **MUST** be an object.

### Properties

The following properties are the required properties of the `web-vst` property. Additional properties may be added, as long as they don't interfer with any of the following properties.

#### .type

The `web-vst` property **MUST** contain a `type` property. The `type` property can be any of the following values:

  * `analyzer`
  * `effect`
  * `generator`

A WebVST with a invalid or undefined type property may not be able to interface with other WebVSTs correctly. Types are defined in the background section above.

#### .version

The spec version the component adheres to **MUST** be listed. This should be in [semver](http://semver.org/) form as a string.

### Example

```json
{
  "name": "example-plugin",
  "description": "An example plugin for the WebWST API.",
  "version": "1.0.0",
  "keywords": [
    "web-vst",
    "effect",
  ],
  "dependencies": {},
  "main": "example-plugin.js",
  "license": "MIT",
  "web-audio": {
    "type": "effect",
    "version": "0.0.1"
  }
}
```

## Implementation

### Constructor

A WebVST **MUST** export only a single constructor whose instances support all of the property requirements below. Constructors exposed via `module.exports` or `export default` **MUST** have the following signature

```javascript
ExamplePlugin (host, properties)
```

- `host` A WebVST constructor **MUST** take an [AudioContext](http://www.w3.org/TR/webaudio/#AudioContext-section) as the first argument. 
- `propertes` The constructor **MAY** take an **OPTIONAL** second argument, which, if present, **MUST** be an object containing any necessary configuration information. Properties defined in the `properties` object **SHOULD** also be public properties on the module instance itself. Exceptions to this may be a WebVST that cannot be changed after instantiation.

### Example

```javascript
import ExamplePlugin from 'example-plugin'

const host = new AudioContext()
const example = new ExamplePlugin(host, {
    mix: 0.5,
    speed: 100
})
```

## Instance Properties

Every WebVST instance **MUST** have the following properties:

- `input` **MUST** be an [AudioNode](http://www.w3.org/TR/webaudio/#AudioNode-section). This node will accept incoming connections from other AudioNodes for tool and effect components. For source components, this is the generator node.
- `output` **MUST** be an [AudioNode](http://www.w3.org/TR/webaudio/#AudioNode-section). This node will make all outgoing connections.
- `meta` **MUST** be an object containing all necessary metadata related to your module. The `meta` object **SHOULD** contain the following properties:

  - `name` **SHOULD** represent the display name of your component.
  - `spec` **SHOULD** represent the spec version the WebVST adheres to. This should be in [semver](http://semver.org/) form as a string.
  - `type` **SHOULD** represent the type of your WebVSTs. **MUST** be one of `effect`, `generator`, or `analyzer`.
  - `properties` if exists, **MUST** be an object listing each of the public, configurable properties of the WebVST. Each property of `properties` **MUST** have a `default` value, and `type`, where `type` is one of `float`, `int`, `bool`, or `enum`. 
  `enum` **MUST** have a `values` array of acceptable values, `float` and `int` types types **MUST** have a `min` and `max` value.

Adherence to the metadata allows other WebVSTs and WebVST Hosts to interact with the WebVST in a consistent way. 

A WebVST **SHOULD** constrain the setters of the property to be within the defined min and max range, or an acceptable fallback value, and also constrain it to the type.

### Demonstration of required metadata

```javascript
ExamplePlugin.prototype.meta = {
  name: "Example WebVST",
  properties: {
    mix: {
      type: 'float',
      min: 0,
      max: 1,
      default: 0.5,
    },
    speed: {
      type: "enum"
      values: ['Half', 'Normal', 'Double'],
      default: 'Half',
    },
  }
};
```

**Note,** `input` and `output` may be the same node.

## Instance Methods

Every WebVST instance **MUST** contain `connect` and `disconnect` methods. These methods **SHOULD** delegate to the `output` node's native `connect` and `disconnect` methods, but should be aware that the `connect` method may need to connect to another WebVST, and may require some internal rerouting upon connect or disconnect.

### Example

```javascript
ExamplePlugin.prototype.connect = function (node) {
  this.output.connect(node.input);
};

ExamplePlugin.prototype.disconnect = function () {
  this.output.disconnect();
};
```

WebVST Generators **MUST** additionally contain `onNoteDown` and `onNoteUp` methods. These methods represents a key being pressed down and up.
These methods **must** adhere to the following signature

```javascript
ExampleGenerator.prototype.onNoteDown = function (frequency, timestamp) {
  // Respond to noteDown event
};

ExamplePlugin.prototype.onNoteUp = function (frequency, timestamp) {
  // Respond to noteUp event
};
```

Where `frequency` represents the pressed keys frequency in hertz, and timestamp represents the current time at the time of the down or up event, represented in milliseconds since 1st of January 1970 00:00:00 UTC.

## Contributing

If you have any comments, questions, or suggestions regarding the WebVST spec,
[please open an issue](https://github.com/thomas-alrek/webvst-spec/issues) for discussion with the community.

## Acknowledgement
This spec is partially based on the previous work of the now inactive [Web Audio Component Spec](https://github.com/web-audio-components/web-audio-components-spec).