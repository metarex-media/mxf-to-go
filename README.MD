# mxf-to-go

**mxf-to-go** converts a snapshot of the [SMPTE RA registers][reg] to [golang
structs][golang]. It converts all the MXF [(Material Exchange Format)][MXF] and
MRX [(Metarex.media)][MRX] types to go types, so that you can easily work on
the contents of mxf files using variables and types with consistent names,
rather than writing code from scratch for every widget.

Before embarking on using this repo, please make sure you are familiar with the
technical concepts of the [Material Exchange Format (MXF)][MXFspec].

## Contents

- [Register Layout](#smpte-registers-structure)
  - [Labels](#labels)
  - [Elements](#elements)
  - [Essence](#essence)
  - [Groups](#groups)
  - [Types](#types)
- [Examples](#examples)
  - [Labels](#labels-example)
  - [Decoding Groups](#decoding-groups-example)
  - [Encoding](#encoding-groups-example)

## SMPTE Registers Structure

This repository converts the following SMPTE registers:

- [Labels](#labels)
- [Elements](#elements)
- [Essence](#essence)
- [Groups](#groups)
- [Types](#types)

Files labelled with the `-dev` suffix contain experimental data and fields that
may not be part of the official [MXF] or [MRX] specifications.

### Labels

A label is an mxf item label, the labels register contains the list of labels.
The labels go code contains an array and map of labels and their properties, as the
properties would be found in the register. Both contain identical properties for
the same UL.

The array can be accessed with the exported label iota values, these give the
position in the array of their label name.

The Universal Label (UL) of a label
`urn:smpte:ul:00000000.00000000.00000000.00000000` is the map key. The UL is
the default UL given in the register, where any `7f` bytes, which means a byte
that can have multiple values, is kept as 7f in their UL, rather than including
all the values that may be present.

### Elements

Elements have a value and a data type, the elements register contains all
of the elements.
The elements in the go code
are wrapped types from the types register, no nesting occurs as each
element is of a type from the types register and not of another element.

Elements with no declared type are stored as an any type e.g. `type EDOI any`.

### Essence

Essence contains the description for the essence (the data contents) within
an mxf file, the essence register contains all of the essence entries.
The essence go code contains a lookup table `EssenceLookUp =
map[string]essenceInformation`, that contains the register information about
the essence.

The Universal Label (UL) of a essence
`urn:smpte:ul:00000000.00000000.00000000.00000000` is the map key. The UL is
the default UL given in the register, where any `7f` bytes, which means a byte
that can have multiple values, is kept as 7f in their UL, rather than including
all the values that may be present.

### Groups

Groups are a group of elements and labels, the group register contains
all of the grouping of these elements and labels.
The groups go code contains the decode functions and encode functions for the group
register.

The `Groups` map of type `map[string]GroupID` is a look up table of all the
groups and their relative decode functions, for each field within the group.

The Universal Label (UL) of a group
`urn:smpte:ul:00000000.00000000.00000000.00000000` is the map key. The UL is
the default UL given in the register, where any `7f` bytes, which means a byte
that can have multiple values, is kept as 7f in their UL, rather than including
all the values that may be present.

The decode map functions contain the ULs for all possible records within a
group and the function to decode those content bytes into a go interface value.

The encode method only allows you to encode non optional contents of a group,
because some optional fields change the interpretation of the group, even when
they are the default value for that field. The encode function generates the
valid SMPTE 377 block of bytes for a group, including the key and BER encoded
length of the group, as well as any subsequent keys and lengths for contents.
These encoded bytes encompass the Key Length Value (KLV) of a group and can be
written straight to an mxf file.

Set length groups are generated as empty encode and decode functions, as they
have special rules which can not be encompassed in the generic SMPTE Register to go
code. These set length groups are few in number and the rules can be found in
their relevant SMPTE documentation. e.g. the primer pack

### Types

Within the types register every type denotes the data type that is used by
other registers.
The types.go file uses the go base types and maps them to the types register.
Some types wrap other register types before reaching the go type, leading to
some types wrapping round another type register value several times, before
reaching the go base. For example `TPackageStrongReference` is of type
`TStrongReference`, which is of go type []byte.

The register to go types as matched as the following:

- records - structs
- fixed - array (of fixed length)
- variable - array slice
- string - []rune
- strong and weak references - []byte
- any other type is a direct translation to the go type. e.g. float (register) to float32 (go)
because the types share the same name.

The accompanying encode functions and decode functions are also generated.

The encode functions **all** follow the layout of

```go
Encode{TypeName}(value {TypeName})([]byte, error)
```

And the decode functions have the following layout.

```go
Decode{TypeName}(value []byte)(any, error)
```

The types are returned as an any to allow the generic handling
of MXF byte streams.

Any enumerated type also has a generated String and MarshallTest function,
to allow the values to be human readable. This also happens automatically
when the values are parsed as JSON and YAML outputs.

e.g. a type of AS_11_Captions_Type_Enum has the following generated code.

```go
type TAS_11_Captions_Type_Enum uint8

func (mt TAS_11_Captions_Type_Enum) MarshalText() ([]byte, error) {
    return []byte(mt.String()), nil
}

func (s TAS_11_Captions_Type_Enum)String() string {

     switch s {
    case 0:
        return "Captions_Hard_of_Hearing"
    case 1:
        return "Captions_Translation"
    default:
        return "invalid value"
    }
}
```

## Examples

The following examples are to help you get an idea of how to utilise this
library, with some of the design patterns that we have used. To see the
examples in practice check out the [mrx-tool
repo][mrxTool], an mxf/mrx file decoder and
encoder that utilizes this library.

There are the following examples:

- [Labels](#labels-example)
- [Decoding Groups](#decoding-groups-example)
- [Encoding](#encoding-groups-example)

### Labels Example

The labels values are translated from the register values to go
and this example will
take you through extracting known and unknown labels into go values.

The `LabelsRegister` is an array containing the labels in the
order they were found in the register. A labels
position can be found using the const positional
variables such as `mxf2go.TransferCharacteristic_Gamma_2_6`, where the name of
the positional variable is the name of the symbol in the register. The label
information can also be extracted using its UL, from the labels look up map.
The array and map contain identical information.

e.g.

```go
// getting all the register information of a label, from knowing the labels symbol (as go)
labelWouldLike := mxf2go.LabelsRegister[mxf2go.TransferCharacteristic_Gamma_2_6]

// getting the label from a known universal label
sometarget := mxf2go.LabelsLookUp["urn:smpte:ul:060e2b34.04010101.01010101.01010000"]
```

### Decoding Groups Example

This example takes you through how to decode MXF bytes for
any group, by first identifying the group, then decoding
the individual fields within.

The pattern for extracting groups is to first get the group key as a string,
these are the first 16 bytes in the byte stream. Then using the `Groups` map
get the group decode map.

Then as you go through the KLVs in the group, you first find the UL of the key,
and look up the key in the decode map, which gives the decode function from the
group decode map.

Then you send just the value bytes from the KLV, to the decode function. The
key and length bytes are not sent to the decode function, as it does not know
how to handle them. An any value is returned, which is the go struct wrapped as
an any, to allow generic handling fo the fields.

A recommended way of handling the results is to put them in a
map[string]map[string]any with `map[groupname][symbol]` result as the layout,
because this can easily be converted into JSON and Yaml, but this is not the
only way of handling them.

A pseudo code example is shown below.
As the full code for an MXF decoder is too long and
complicated for a readme, checkout the [mrx-tool repo][mrxTool]
for a working example of an mxf decoder that uses this library.

```go

func byteToHex(hexb []byte) string {
    return fmt.Sprintf("%02x%02x%02x%02x.%02x%02x%02x%02x.%02x%02x%02x%02x.%02x%02x%02x%02x",
        hexb[0], hexb[1], hexb[2], hexb[3], hexb[4], hexb[5], hexb[6], hexb[7], hexb[8],
        hexb[9], hexb[10], hexb[11], hexb[12], hexb[13], hexb[14], hexb[15])
}


// get the group key in the format
// "00000000.00000000.00000000.00000000"
header := GroupBytes[:16]
stringUL := byteToHex(header)

// loop through packets
groupDecoder, ok := mxf2go.Groups["urn:smpte:ul:"+stringUL]

 if !ok {
    // then this group doesn't exist in the register (yet)
    // so best to stop decoding as no results will be generated
 }

// store the results as a
results := make(map[string]any)

// getContents would break the bytes down into san array of the
// Key Length Values (klvs) within the group.
/*
 e.g. a
type klv struct{
    Key string  //the stringified UL
    Length, Value []byte
 }
 is used for this example.*/
Packets := GetContents(GroupBytes)

// Packets is the array of values in the group
for _, packet = range Packets {

    // get the UL of the individual packet
    decode, exists := groupDecoder[packet.Key]

    // check that the decode function is there
    if exists {
        // then decode the bytes
        res, _ := groupDecoder.Decode(packet.Value)
            // add the results
            // decode.UL is the register symbol
        results[decode.UL] = res
    }
}
```

### Encoding Groups Example

This example takes you through how to generate MXF header bytes for
any group.

The recommended practice is to simply create a struct and fill in the values,
like you would any other struct, then run the `Encode` method on the object
you've created.

In the register any entry with the key `"KLVSyntax"` with a value of 53 or 06
53 requires a shorthand tag. A shorthand tag is a unique (within the mxf file)
2 byte representation of a 16 byte UL, often used for common contents to reduce
the size of the file. This results in some encode methods requiring a
`mxf2go.Primer` object as an input for generating these 2 byte tags. Some tags
are dynamically allocated so require a pointer to keep track of the dynamic
allocation, through out the encoding of all the different groups to prevent
clashes.

This map string is the shorthand form bytes as a string. These can then be
saved in the primer pack of the MXF file.

The example below is the encoding of a group with dynamic tags.

```go
// create a single primer to be used
// with every encoding method
primer := mxf2go.NewPrimer()

// an example struct
idid := mxf2go.TUUID(uuid.New())
identifier := mxf2go.GIdentificationStruct{InstanceID: idid, ApplicationSupplierName: []rune("metarex.media"), ApplicationName: []rune("An example slice of code"),
    ApplicationVersionString: []rune("0.0.1"), ApplicationProductID: productID, GenerationID: newAUID()}


// save the bytes however you please
identifierBytes, err := identifier.Encode(primer)

if err != nil {
    // handle your error
}

```

## Things to add - future work

The following is a list of things to be added to the library.

- Error handling for decode and encode functions

If you think we've missed something or have any requests then
please create an issue.

[golang]:  https://go.dev
[mxf]:     https://en.wikipedia.org/wiki/Material_Exchange_Format
[mxfspec]: https://pub.smpte.org/doc/377/
[mrx]:     https://metarex.media
[reg]:     https://registry.smpte-ra.org/apps/pages/
[mrxTool]:  https://github.com/metarex-media/mrx-tool/