# AMWA BCP-007-02: NMOS With IPMX/USB
{:.no_toc}  
Copyright 2024, Matrox Graphics Inc.

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the “Software”), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED “AS IS”, WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
  
---
  
{:toc}

## Introduction

The VSF/IPMX [TR-10-14][] technical recommendation defines the transport of a USB stream over TCP/IP. It enables the multiplexed transport of keyboard, mouse, data, audio, and video sub-streams over TCP/IP. Senders and Receivers using the USB transport have their `format` attribute set to `urn:x-nmos:format:data` and their `transport` attribute set to  `urn:x-nmos:transport:usb`. The `media_type` attribute of a USB Receiver is `application/usb`, and the `media_type` of a data Flow connected to a USB Sender is also `application/usb`.

The content of a USB stream is not exposed at the NMOS level. It is treated as an opaque data stream. A USB Receiver connected to a USB Sender gains access to the USB devices available at the Sender. Each such device is represented by a corresponding multiplexed data sub-stream. Devices may be plugged or unplugged dynamically without affecting the connection between the Receiver and Sender, or the associated Source, Flow, and Sender resources.

The roles of USB Senders and Receivers may initially appear counterintuitive due to their physical placement within devices. Consider a KVM system in which a User interacts with a local KVM endpoint that connects to a remote computer over the network. The remote computer outputs video and audio; therefore, the audio and video Senders reside on the remote side, and the corresponding Receivers reside on the local KVM endpoint. Conversely, the local KVM endpoint provides keyboard, mouse, and storage devices; hence, the USB Sender resides at the local KVM endpoint, while the USB Receiver is located on the remote computer side.

KVM systems are often sensitive to privacy and security. The [TR-10-14][] technical recommendation includes built-in support for privacy encryption. Given that USB streams are unlikely to be used without privacy encryption, the [TR-10-14][] technical recommendation retains the privacy encryption components in the protocol messages, even when encryption is not enabled. For more details on privacy-encrypted USB streams, refer to the [BCP-005-03][] "NMOS With Privacy Encryption" document.

## Use of Normative Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119][RFC-2119].

## Definitions

The NMOS terms 'Controller', 'Node', 'Source', 'Flow', 'Sender', 'Receiver' are used as defined in the [NMOS Glossary](https://specs.amwa.tv/nmos/main/docs/Glossary.html).

## USB IS-04 Sources, Flows and Senders

Nodes that are capable of transmitting USB data streams MUST expose Source, Flow and Sender resources in the IS-04 Node API.

Nodes compliant with this specification MUST implement IS-04 v1.3 or higher, and IS-05 v1.1 or higher.

### Sources

A USB Source resource MUST set the `format` attribute to `urn:x-nmos:format:data` and MUST be associated with a Flow of the same `format` via the `source_id` attribute of the Flow. 

A `usb_device` object is defined as:

```
{
    "ipmx_bus_id": [64]integer, // IPMX USB UTF-8 BUSID 64 byte value (integers in the range 0 to 255)
    "class": []integer,         // class of a device or classes if a composite device (integers in the range 0 to 255)
    "vendor": integer,          // vendor id (integer in the range 0 to 65535)
    "product": integer,         // product id (integer in the range 0 to 65535)
    "serial": string,           // serial number (empty string "" if not available)
}
```
A USB Source MAY include a `usb_devices` attribute, which is an array of `usb_device` objects. This attribute describes the USB devices accessible to a Receiver via the USB data stream. Inclusion of this information is optional.

Examples of Source resources are provided in [Examples](./examples).

### Flows

A USB Flow resource MUST set the `media_type` attribute to `application/usb` and the `format` attribute to `urn:x-nmos:format:data`. A USB Flow MUST include a `source_id` attribute referencing a Source of the same `format`.

Examples of Flow resources are provided in [Examples](./examples).

### Senders

A USB Sender resource MUST set the `transport` attribute to `urn:x-nmos:transport:usb`.

A Sender associated with a USB Flow via the `flow_id` attribute SHOULD provide Capabilities describing the characteristics of the data Flow.

The Sender MUST express its limitations or preferences regarding the USB streams it supports by indicating constraints in accordance with the [BCP-004-02][] specification. The Sender SHOULD declare its constraints as precisely as possible to allow a Controller to determine, with high confidence, the Sender's stream capabilities. It is not always practical for the constraints to enumerate every type of stream a Sender can or cannot produce; however, they SHOULD describe as many commonly used operating points as practical, along with any preferences.

The `constraint_sets` parameter within the `caps` object MUST be used to describe combinations of parameters the Sender can support, using the parameter constraints defined in the [Capabilities Register](https://specs.amwa.tv/nmos-parameter-registers/branches/main/capabilities/) of the NMOS Parameter Registers.

A Sender SHOULD provide the `urn:x-nmos:cap:transport:usb_class` capability to indicate the USB classes (integers in the range 0 to 255) supported by the Sender. See [USB](https://www.usb.org) for class code definitions.

A USB Sender operates as a TCP/IP server and accepts connections from USB Receivers. The underlying transport protocol for `urn:x-nmos:transport:usb` is TCP, optionally using MPTCP (MultiPath TCP) for redundancy.

An example Sender resource is provided in [Examples](./examples).

#### SDP format-specific parameters

The `manifest_href` attribute of the Sender MUST provide the URL to an SDP transport file that complies with the requirements of [TR-10-14][] and the following additional rules:

- The media description line `m=<media> <port> <proto> <fmt> ...` MUST have `<media>` set to `application`, `<proto>` set to `TCP` and `<fmt>` set to `usb`. This indicates that the `media_type` is `application/usb` and that the transport protocol is TCP, as used by `urn:x-nmos:transport:usb`.

- The connection information lines `c=<nettype> <addrtype> <connection-address>` MUST have `<connection-address>` set to the IP address of the Sender's TCP server.

- The attribute `a=setup:passive` MUST be specified.

- If redundancy is used, at most two paths (legs) MUST be specified using two separate media descriptors. The `<connection-address>` of each descriptor MUST specify a different path to the Sender's TCP server. A `a=group:DUP` session attribute MUST reference both media descriptors using their `a=mid:` attributes. The first identifier in the `a=group:DUP` session attribute MUST correspond to the first leg (path), and the second identifier to the second leg. The first leg corresponds to entry 0 of the IS-05 transport parameters array; the second leg corresponds to entry 1.

## USB IS-04 Receivers

Nodes that are capable of receiving USB data streams MUST have Receiver resources in the IS-04 Node API.

Nodes compliant with this specification MUST implement IS-04 v1.3 or higher and IS-05 v1.1 or higher.

A USB Receiver resource MUST set the `transport` attribute to `urn:x-nmos:transport:usb`.

A USB Receiver MUST set the `format` attribute to `urn:x-nmos:format:data`, MUST include `application/usb` in the `media_types` array within the caps object, and SHOULD declare the Receiver's Capabilities for the data stream.

The Receiver MUST express its limitations or preferences regarding the USB streams that it supports by declaring constraints in accordance with the [BCP-004-01][] specifications. The Receiver SHOULD express its constraints as precisely as possible, to enable a Controller to determine, with high confidence, the Receiver's compatibility with available streams. It is not always practical for the constraints to enumerate every type of stream a Receiver can or cannot consume; however, they SHOULD describe as many commonly used operating points as practical, along with any preferences.

The `constraint_sets` parameter within the `caps` object MUST be used to describe combinations of parameters that the Receiver can support, using the parameter constraints defined in the [Capabilities Register](https://specs.amwa.tv/nmos-parameter-registers/branches/main/capabilities/) of the NMOS Parameter Registers.

A USB Receiver SHOULD provide the `urn:x-nmos:cap:transport:usb_class` capability to indicate the USB classes (integers in the range 0 to 255) supported by the Receiver. See [USB](https://www.usb.org) for class code definitions.

A USB Receiver operates as a TCP/IP client. A USB Sender accepts connections from Receivers. The underlying transport protocol for `urn:x-nmos:transport:usb` is TCP, optionally using MPTCP (MultiPath TCP) for redundancy.

An example Receiver resource is provided in [Examples](./examples).

### Grouping of Receivers

In some scenarios, a group of USB Receivers controls the USB sub-system of a Device. 

Receivers MUST declare a device-scope tag "urn:x-nmos:tag:grouphint/v1.0" in their `tags` attribute.

The "urn:x-nmos:tag:grouphint/v1.0" tag array MUST contain a single string formatted as follows: "<group-name>:<role-in-group> <role-index>".

All USB Receivers in the group MUST share the same `<group-name>`, and each MUST declare a unique `<role-index>` within the `DATA` role.

Using this grouping convention, a Controller can determine how many USB Senders simultaneously control the USB sub-system of a Device. 

## USB IS-05 Senders and Receivers

Connection Management using IS-05 proceeds in the same manner as for any other transport, using USB-specific transport parameters defined in [USB Sender transport parameters](./API/schemas/sender_transport_params_usb.json) and [USB Receiver transport parameters](./API/schemas/receiver_transport_params_usb.json). The `source_ip` and `source_port` transport parameters MUST be present in the IS-05 `active`, `staged`, and `constraints` endpoints of a USB Sender. The `source_ip`, `source_port` and `interface_ip` transport parameters MUST be present in the IS-05 `active`, `staged`, and `constraints` endpoints of a USB Receiver.

Redundancy MUST be implemented using MPTCP. At most two sets of transport parameters MUST be specified for Senders and Receivers supporting redundancy with the `urn:x-nmos:transport:usb` transport. The parameters for the first leg MUST appear as entry 0 in the transport parameters array; those for the second leg MUST appear as entry 1.

For security reasons, USB streams are typically encrypted using the IPMX [TR-10-5][] Privacy Encryption Protocol. Additional `ext_privacy_*` extended transport parameters, as normatively defined in [TR-10-13][], are present in the IS-05 `active`, `staged`, and `constraints` endpoints of USB Senders and Receivers. 
Refer to the [BCP-005-03][] "NMOS With Privacy Encryption" document for more details.

### Receivers

A `PATCH` request on the **/staged** endpoint of an IS-05 Receiver MAY include an SDP transport file in the `transport_file` attribute. The SDP transport file for a USB stream is expected to comply with the IPMX [TR-10-14][] specification. It need not comply with the additional requirements specified for SDP transport files at Senders.

If the USB Receiver is not capable of consuming the stream described by the `PATCH` request, it SHOULD reject the request. If it is unable to assess stream compatibility because some parameters are missing from the `PATCH` request, it MAY accept the request and defer stream compatibility assessment.

A Controller MAY connect a USB Receiver that does not support redundancy to either leg of a Sender supporting redundancy. When connecting to a Sender that does not support redundancy, a Controller MUST set the `source_ip` and `source_port` attributes of the unused leg of a Receiver supporting redundancy to `null`.

#### Transport Parameters

| Name          | Description |
|---------------|-------------|
| `interface_ip`| MUST be set to the IP address of the Receiver’s network interface. The Receiver lists available interface addresses in the Constraints endpoint; the special value `auto` lets the Receiver choose an interface automatically. |
| `source_ip`   | MUST be set to the IP address of the TCP server (Sender) that delivers the USB packets. A `null` value indicates the address has not yet been configured. |
| `source_port` | MUST tbe set to the port of the TCP server (Sender) that delivers the USB packets. Accepts either an integer within the range 0–65535, the string "auto", or `null`. If set to "auto", the default is 5004. A `null` value indicates the port has not yet been configured. |
| `ext_*`       | Vendor‑specific, future AMWA extension parameters, or `ext_privacy_*`  transport parameters specified in [TR-10-14][] |


### Senders

If the IS-04 Sender's `manifest_href` attribute is not `null`, the SDP transport file returned by the **/transportfile** endpoint MUST comply with the same requirements as the SDP transport file referenced by the Sender's `manifest_href` attribute.

Unless constrained by [IS-11][], a Sender MAY produce any USB stream that is compliant with the associated [TR-10-14][] Flow.

#### Transport Parameters

| Name          | Description |
|---------------|-------------|
| `source_ip`   | MUST be set to the IP address of the TCP server that delivers the USB packets. The Sender should enumerate available interfaces in the `constraints` endpoint; the special value `auto` lets the Sender select an interface automatically, based on its routing rules. |
| `source_port` | MUST be set to the port of the TCP server that delivers the USB packets. Accepts either an integer within the range 0–65535 or the string "auto"  (default 5004). |
| `ext_*`       |  Vendor‑specific, future AMWA extension parameters, or `ext_privacy_*`  transport parameters specified in [TR-10-14][] |

## USB IS-11 Senders and Receivers


## Controllers

A Controller SHOULD use a Sender's `urn:x-nmos:cap:transport:usb_class` capability to verify  Receivers' compatibility with the Sender and, if necessary, constrain the Sender to ensure compliance with the Receivers. A Sender indicates its support for being constrained on this capability by enumerating `urn:x-nmos:cap:transport:usb_class` in its [IS-11][] `constraints/supported` endpoint.

> Note: There is no `usb_class` Sender attribute, as might typically be expected, because a USB stream is composed of multiple sub-streams, each of which can be associated with multiple USB classes. The set of classes present in a given USB stream often changes dynamically.

[RFC-2119]: https://tools.ietf.org/html/rfc2119 "Key words for use in RFCs"
[IS-04]: https://specs.amwa.tv/is-04/ "AMWA IS-04 NMOS Discovery and Registration Specification"
[IS-05]: https://specs.amwa.tv/is-05/ "AMWA IS-05 NMOS Device Connection Management Specification"
[IS-11]: https://specs.amwa.tv/is-11/ "AMWA IS-11 NMOS Stream Compatibility Management Specification"
[NMOS Parameter Registers]: https://specs.amwa.tv/nmos-parameter-registers/ "Common parameter values for AMWA NMOS Specifications"
[VSF]: https://vsf.tv/ "Video Services Forum"
[SMPTE]: https://www.smpte.org/ "Society of Media Professionals, Technologists and Engineers"
[BCP-004-01]: https://specs.amwa.tv/bcp-004-01/ "AMWA BCP-004-01 NMOS Receiver Capabilities"
[BCP-004-02]: https://specs.amwa.tv/bcp-004-02/ "AMWA BCP-004-02 NMOS Sender Capabilities"
[TR-10-5]: https://vsf.tv/download/technical_recommendations/VSF_TR-10-5_2024-02-23.pdf "HDCP Key Exchange Protocol - HKEP"
[TR-10-8]: https://vsf.tv/download/technical_recommendations/VSF_TR-10-8_2023-04-14.pdf "NMOS Requirements"
[TR-10-14]: https://vsf.tv/download/technical_recommendations/VSF_TR-10-14_2024-09-24.pdf "IPMX	USB"
[BCP-005-03]: https://specs.amwa.tv/bcp-005-03/ "AMWA BCP-005-03 NMOS With Privacy Encryption"
