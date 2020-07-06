---
title: "Implementing EDHOC and OSCORE"
abbrev: "EDHOC + OSCORE"
docname: draft-palombini-core-oscore-edhoc-latest
cat: std

ipr: trust200902
area: Internet
workgroup: CoRE Working Group
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: F. Palombini
    name: Francesca Palombini
    organization: Ericsson
    email: francesca.palombini@ericsson.com

normative:
  RFC2119:
  RFC7252:
  RFC8174:
  RFC8613:
  I-D.selander-lake-edhoc:
  I-D.ietf-cbor-7049bis:
  I-D.ietf-lake-reqs:

informative:



--- abstract

TODO Abstract

--- middle

# Introduction

This document presents possible optimization options to combine the EDHOC protocol {{I-D.selander-lake-edhoc}}, run over CoAP {{RFC7252}}, with the first subsequent OSCORE {{RFC8613}} transaction. 
This allows for a minimum number of round trips necessary to setup the OSCORE security context and complete an OSCORE transaction, for example when an IoT device gets configured in a network for the first time.

The number of round trips for a protocol implies a minimum for the number of flights, which can have a substantial impact on performance with certain radio technologies as discussed in Section 2.11 of {{I-D.ietf-lake-reqs}}. 
Without this optimization it is not possible even in theory to obtain the minimum number of flights. 
With this optimization it is possible also in practice since the last message of the EDHOC protocol can be made relatively small (see Section 1 of {{I-D.selander-lake-edhoc}}) and allows additional OSCORE protected CoAP data within target MTU sizes {{I-D.ietf-lake-reqs}}.

The goal of this draft is to gather opinions on each option, and develop only one of these options.

## Terminology

{::boilerplate bcp14}

The reader is expected to be familiar with terms and concepts of {{RFC7252}}, {{RFC8613}} and {{I-D.selander-lake-edhoc}}.

# Background

EDHOC is a 3 message key exchange protocol.
Section 7.1 of {{I-D.selander-lake-edhoc}} specifes how to transport EDHOC over CoAP: the EDHOC data (referred to as "EDHOC messages") are transported in the payload of CoAP requests and responses.
More specifically, the Initiator, acting as CoAP Client, sends a POST request to a reserved resource at the Responder, acting as CoAP Server.
This triggers the EDHOC exchange on the CoAP Server, which replies with 2.04 (Changed) Response containing EDHOC message 2.
The EDHOC message 3 is also sent by the CoAP Client in a CoAP POST request to the same resource used for EDHOC message 1.
The Content-Format of these CoAP messages is set to "application/edhoc".

After this exchange takes place, and after successful verifications specified in the EDHOC protocol, Client and Server derive the OSCORE Security Context, as specified in Section 7.1.1 of {{I-D.selander-lake-edhoc}}.
They are then ready to use OSCORE.

This sequential way of running EDHOC and then OSCORE is specified in {{fig-non-combined}}.
As shown, this mechanism is executed in 3 roundtrips.

~~~~~~~~~~~~~~~~~
   CoAP Client                                  CoAP Server
        | ------------- EDHOC message_1 ------------> |
        |                                             |
        | <------------ EDHOC message_2 ------------- |
        |                                             |
EDHOC verification                                    |      
        |                                             |
        | ------------- EDHOC message_3 ------------> | 
        |                                             |
        |                                    EDHOC verification
        |                                             |
OSCORE Sec Ctx                                OSCORE Sec Ctx
  Derivation                                    Derivation 
        |                                             |
        | -------------- OSCORE Request ------------> |
        |                                             |
        | <------------ OSCORE Response ------------- |
        |                                             |
~~~~~~~~~~~~~~~~~
{: #fig-non-combined title="EDHOC and OSCORE run sequentially" artwork-align="center"}

The number of roundtrips can be minimized: after receiving EDHOC message 2, the CoAP Client has all the information needed to derive the OSCORE Sec Ctx before sending EDHOC message 3. 
That means that it can potentially send at the same time both EDHOC message 3 and the subsequent OSCORE Request.
On a semantyc level, this approach requires in practice to send two separate 
REST requests at the same time.
Defining the specific details of how to transport the data and order of processing is the goal of this specification.

The high level message flow of running EDHOC and OSCORE combined is shown in {{fig-combined}}.

~~~~~~~~~~~~~~~~~
   CoAP Client                                  CoAP Server
        | ------------- EDHOC message_1 ------------> |
        |                                             |
        | <------------ EDHOC message_2 ------------- |
        |                                             |
EDHOC verification +                                  |
  OSCORE Sec Ctx                                      |
    Derivation                                        |
        |                                             |
        | ------------- EDHOC message_3 ------------> |
        |              + OSCORE Request               | 
        |                                             |
        |                                  EDHOC verification +
        |                                     OSCORE Sec Ctx
        |                                        Derivation 
        |                                             |
        | <------------ OSCORE Response ------------- |
        |                                             |
~~~~~~~~~~~~~~~~~
{: #fig-combined title="EDHOC and OSCORE combined" artwork-align="center"}

# EDHOC in OSCORE Protected CoAP {#edhoc-in-oscore}

The first possibility is to send the EDHOC message 3 inside an OSCORE protected CoAP message.
The request is in practice the OSCORE CoAP Request from {{fig-non-combined}}, sent to the protected resource andpoint and with correct CoAP method and options, with the addition that it also transports EDHOC message 3.
As EDHOC message 3 is too large for being contained in a CoAP Option, it would have to be transported in the CoAP payload.
The payload is formatted as a CBOR sequence of two CBOR wrapped items: the EDHOC message 3 and the OSCORE ciphertext, in this order.

When receiving such a request, the Server needs to execute the following processing, additional to EDHOC, OSCORE and CoAP processing:

1. Parse the signalling option to identify that this is an OSCORE + EDHOC request (more in {{sign-1}} and {{sign-2}}).

2. Extract the EDHOC message 3 from the payload.

3. Execute the EDHOC processing, including verifications and OSCORE Security Context derivation.

4. Decrypt and verify the remaining OSCORE protected CoAP request as defined by OSCORE.

5. Process the CoAP requests.

The following sections describe 2 ways of signalling that the EDHOC message is transported in the OSCORE message. In a next update of these document, we will 

## Signalling in a New EDHOC Option {#sign-1}

<!-- Malisa preferred option -->

One way to signal that the Server is to extract and process EDHOC message 3 before the OSCORE message is processed is to define a new CoAP Option, called the EDHOC Option.

This Option being present (either in a request or response) means the payload contains EDHOC data in the payload, that must be extracted and processed before the rest of the message can be executed.
The EDHOC message is to be extracted from the CoAP payload as the CBOR wrapped first element of a CBOR sequence.

The Option is critical, Safe-to-Forward, and part of the Cache-Key.

The Option value is always empty. If any value is sent, the value is simply discarded.

The Option must occurr at most once.

The Option is of Class U for OSCORE.

## Signalling in the OSCORE Option {#sign-2}

<!-- Klaus preferred option -->

Another way to signal that the EDHOC message is to be extracted from the CoAP payload as the CBOR wrapped first element of a CBOR sequence, and that the processing defined in {{edhoc-in-oscore}} is to be executed is to use one of the OSCORE Flag Bits.

Bit Postion: 8

Name: EDHOC

Description: Set to 1 if the payload is a sequence of EDHOC data and OSCORE payload.

Reference: this document


# OSCORE Request in a COAP message

<!-- Jim preferred option -->

Instead of transporting the EDHOC message inside an OSCORE message, the OSCORE protected data could be transported in a EDHOC message.
The request is in practice the CoAP POST Request containing EDHOC message 3 from {{fig-non-combined}}, sent to the unprotected resource andpoint reserved to EDHOC processing, with the addition that it also transports the OSCORE Option and Ciphertext.
The OSCORE Option and ciphertext contain all the information to reconstruct the original OSCORE Request, including CoAP method, options and payload.
Both OSCORE Option, EDHOC message_3 and ciphertext would have to be transported in the CoAP payload.
The payload is formatted as a CBOR sequence of three CBOR wrapped items: the EDHOC message 3, the OSCORE Option and the OSCORE ciphertext, in this order.

When receiving such a request, the Server needs to execute the following processing, additional to EDHOC, OSCORE and CoAP processing:

1. Parse the signalling option to identify that this is an EDHOC + OSCORE request (more in {{sign-3}} and {{sign-4}}).

2. Extract the EDHOC message 3 from the payload.

3. Execute the EDHOC processing, including verifications and OSCORE Security Context derivation.

4. Extract the OSCORE Option value and ciphertext from the payload and reconstruct the OSCORE protected CoAP request.

5. Decrypt and verify the reconstructed OSCORE protected CoAP request as defined by OSCORE.

6. Process the CoAP requests.

This processing requires one more step, as the server must build the protected request from the payload before being able to process it.

## Signalling in a New EDHOC Option {#sign-3}

One way to signal that the Server is to build and process the OSCORE request after EDHOC processing is to define a new CoAP Option, called the EDHOC Option.

This Option being present (either in a request or response) means the payload contains OSCORE option value and ciphertext in the payload, that must be extracted and processed after the EDHOC processing.
The OSCORE option and ciphertext are to be extracted from the CoAP payload as the CBOR wrapped second and third element of a CBOR sequence.

The Option is critical, Safe-to-Forward, and part of the Cache-Key.

The Option value is always empty. If any value is sent, the value is simply discarded.

The Option must occurr at most once.

The Option is of Class U for OSCORE.

## Predetermined {#sign-4}

Another way to support the combined mode and to mandate that the Server is to build and process the OSCORE request after EDHOC processing is to set up pre-determined policies in place in both the Client and Server.

A Client may be set up to support at the same time receiving only EDHOC message 3 or both EDHOC message 3 and OSCORE Option and ciphertext in the request.
The Client would be able to distinguish based on the number of CBOR elements in the payload, and process the message accordingly.

# Security Considerations

TODO Security

# IANA Considerations

This document has no IANA actions.



--- back

# Acknowledgments
{:numbered="false"}

Jim Schaad, Malisa Vucinic, Klaus Hartke, Christian Amsuess