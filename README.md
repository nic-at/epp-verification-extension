# .at EPP Verification Extension

v1.0, 20240924, Contact: Alexander Mayrhofer <alexander.mayrhofer@nic.at>

This Extensible Provisioning Protocol (EPP) Extension provides the EPP protocol transport for the `verificationReport` and the corresponding, required status values associated to Domain and Contact objects.

The extension is a command/response extension, loosely inspired by RFC 3915 ("Domain Registry Grace Period Mapping"), especially its "Restore Report". 

## Workflow

The intended workflow for this extension is as follows:

1. Registry identifies Domain objects for which registrant must be verified (selection mechanism is out of scope of this extension)
2. Registry notifies registrar about that verification request (preferrably by the EPP Message Queue mechanism - message format is out of scope of this extension)
3. Registrar receives verification request, and verifies registrant contact for the given domain name
4. Registrar creates Verification Rerport and provisions this with the registry via a <contact:update> transform command on the registrant, using the extension described in this document.
5. Registry acts upon the Verification Report contents, with actual procedures being subject to local server policy

During that procedure, the Domain and Contact objects show additional status values, as specified in the `statusValueType` of the XML schema.

## The Verification Report

When a registrar has completed verification process for a registrant, the result will be communicated back to the registry using the `<contact:create>` or `<contact:update>` transform commands, via a `<verification:report>` element (assuming the extension namespace is tied to the `verification` name space prefix). The `<verification:report>` can occur at most once in those commands, however, a registrar may submit multiple Verification Reports in multiple commands. The registry will consider the most recently submitted Verification Report for the verification status of the registrant (and associated domains), but actual interpretation is local server policy.

A `<verification:report>` element contains the following elements:

- one `result` element, containing information about the outcome of the verification process, containing the string "success" or "failure"
- one `verificationDate` element, containing the point in time of the completion of the verification process, in `dateTime` format
- one optional `method` element, describing the method of verification in free text form
- one  optional `reference` element, provinding the representation of an internal reference to the verification documentation at the registrar (eg. a ticket number)
- one  optional `agent` element, containing the name of the entity that has performed the verification (eg. in case the verification was outsourced to a third party)

Interpretation and restriction of those fields is subject to server policy, however a server MUST NOT accept a command that includes a Verification Report with a future `verificationDate` element. Also note that the elements defined as optional in the Schema can still be required based on local server policy.

When a server returns a Verification Report in the response to a `<contact:info>` command, the `<verification:report>` element may contain the following attributes:

- `receivedDate` - The point in time at which the Verification Report was received from the client
- `clID` - The identifier of the client that originally submitted the Verification Report (as contacts may be transferred between clients)

These attributes cannot be used when the Verification Report is submitted by the client, and attempts to do so MUST be rejected by the server with a policy error.

## Verification Status

The extension provides for Verification Status to be included in `info` query command responsed, with the following possible values:

- `none` indicating that no verification process is or was performed on that object yet. 
- `pending` reflecting that an ongoing verification process is underway, on either the object itself, or on the associated registrant object (in case of a domain object)
- `serverHold` (only sensible on the Domain object), indicating that the verification process has led to the domain being excluded from the DNS.
- `verified` reflecting that the verification process for the object has successfully concluded.
- `failed` indicating that the Verification Process for the object has failed.

Note that Verification Status refers to the most recent Verification Process.

## Security Considerations

Registrars must consider that contents of the Verification Report might be visible to other clients, especially in situations where contact objects (and hence their associated verification report) is transferred to a different client. However, as the Verification Report itself does not contain registrant data, the privacy impact is likely very limited.

## Examples

The following examples are (incomplete) XML snippets of potential client / server interactions using the specified extension. "C:" indicates XML sent by the client, while "S:" indicates XML sent by the server.

### Contact Update with verificationReport

Example of a Registrar submitting a successful `verificationReport` for an existing contact object via a `<contact:update>` transform command:

```xml
C: <?xml version="1.0" encoding="UTF-8" standalone="no"?>
C:  <epp xmlns="urn:ietf:params:xml:ns:epp-1.0" 
C:        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
C:        xsi:schemaLocation="urn:ietf:params:xml:ns:epp-1.0 epp-1.0.xsd">
C:  <command>
C:   <update>
C:    <contact:update xmlns:contact="urn:ietf:params:xml:ns:contact-1.0" xsi:schemaLocation="urn:ietf:params:xml:ns:contact-1.0 contact-1.0.xsd"> 
C:      <contact:id>myhandle</contact:id>
C:      <contact:chg/>
C:    </contact:update> 
C:   </update>
C:   <extension>
C:    <verification:update 
C:           xmlns:verification="http://www.nic.at/xsd/at-ext-verificationReport-1.0" 
C:           xsi:schemaLocation="urn:ietf:params:xml:ns:verificationReport-1.0 
C:           verificationReport-1.0.xsd">  
C:     <verification:report>
C:      <verification:result>success</verification:result>
C:      <verification:verificationDate>2023-11-26T22:00:00.0Z</verification:verificationDate>
C:      <verification:method>ID Austria</verification:method>
C:      <verification:reference>Process#321</verification:reference>
C:      <verification:agent>SnakeOil used Domains and Certificates</verification:agent>
C:     </verification:report>
C:    </verification:update>
C:   </extension>
C:   <clTRID>ABC-12345</clTRID>
C:  </command>
C: </epp>
```

### Contact Info, returning a verificationReport

Example of a server responding to a `<contact:info>` query command, with the response containing a `verificationReport` structure. Note that the "standard" contact object response is truncated for brevity.

```xml
S: <?xml version="1.0" encoding="UTF-8" standalone="no"?>
S: <epp xmlns="urn:ietf:params:xml:ns:epp-1.0">
S:  <response>
S:    <result code="1000">
S:      <msg>Command completed successfully</msg>
S:    </result>
S:    <resData>
S:      <contact:infData
S:          xmlns:contact="urn:ietf:params:xml:ns:contact-1.0">
S:  {{ standard contact info data goes here }}
S:      </contact:infData>
S:    </resData>
S:    <extension>
S:      <verification:infData xmlns:verification="http://www.nic.at/xsd/at-ext-verification-1.0"
S:             xsi:schemaLocation="http://www.nic.at/xsd/at-ext-verification-1.0.xsd">
S:        <verification:report receivedDate="2024-03-26T22:00:00.0Z" clID="reg123">
S:          <verification:result>success</verification:result>
S:          <verification:verificationDate>2023-11-26T22:00:00.0Z</verification:verificationDate>
S:          <verification:method>ID Austria</verification:method>
S:          <verification:reference>Process#321</verification:reference>
S:          <verification:agent>RegistrarA</verification:agent>
S:        </verification:report>
S:        <verification:status s="verified"/>
S:      </verification:infData>
S:    <trID>
S:      <clTRID>ABC-12345</clTRID>
S:      <svTRID>54322-XYZ</svTRID>
S:    </trID>
S:  </response>
S: </epp>
```

Note that in addition to the Verification Report, that response also contains the `status` element that reflects the verification status of the object.

### Domain Info example

Example response to a `<domain:info>` query command, returning verification status information alongside the standard domain info response data. Note that standard domain information is redacted for brevity.

```xml
S: <?xml version="1.0" encoding="UTF-8" standalone="no"?>
S: <epp xmlns="urn:ietf:params:xml:ns:epp-1.0"
S:        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
S:        xsi:schemaLocation="urn:ietf:params:xml:ns:epp-1.0
S:        epp-1.0.xsd">
S:  <response>
S:    <result code="1000">
S:      <msg>Command completed successfully</msg>
S:    </result>
S:    <resData>
S: {{ usual domain info data removed }}
S:    </resData>
S:    <extension>
S:      <verification:infData xmlns:verification="http://www.nic.at/xsd/at-ext-verification-1.0"
S:          xsi:schemaLocation="http://www.nic.at/xsd/at-ext-verification-1.0.xsd">
S:        <verification:status s="pending"/>
S:        <verification:actionDate>2025-11-26T22:00:00.0Z</verification:actionDate>
S:      </verification:infData>
S:    </extension>
S:    <trID>
S:      <clTRID>ABC-12345</clTRID>
S:      <svTRID>54322-XYZ</svTRID>
S:    </trID>
S:  </response>
S: </epp>
```
It is local server policy when Verification Status information is included in a domain object, but it is RECOMMENDED to at least include the following two status values:

- `pending` when a verification process on the associated registrant object is in pending state.
- `serverHold` when the verification process on the associated registrant required to remove the Domain from the DNS.

