# HTTP over SMS Tunnel Protocol Specs

**Client:** The mobile device which wants to make an HTTP request using only SMS.  
**Relay Server:** Listens for SMS messages from the Client, relays request to Server; and relays the Server’s response to the Client.  
**Server:** Has a REST API the Client wants to use.

## 1. Preconditions
1. Server knows phone number(s) of Client (Must support more than one phone number per user so that the user can work on multiple devices at once.)
2. Client and Server store a shared secret AES-128 key (generated by server; unique to each client; retrieved by client using HTTP (not over SMS) via the REST API.
3. Client and Server store the last request number used by this Client (incremented by Client for each new request).
4. Client has the phone number of the Relay Server.
5. Relay Server has a valid login credential for the server’s REST API. It authenticates via the REST API and then likely has an access token to communicate with the server.

## 2. Process
1. Client builds HTTP request, and decides to use SMS to transmit the request and receive the response.	 
2. Client creates a JSON Request Structure for HTTP request:  
	_Field names may be truncated to “r”, “m”, “e”, “h”, and “b”; empty fields are omitted._
	```
	{
	  "requestNumber": 8581,	(See below for details)
	  "method": "POST",
	  "endpoint": "...",		(Full path and query string to endpoint; not including server.)
	  "header": "...",		(May be omitted if empty)
	  "body": "..."			(May be a JSON object, or a quoted string; omitted if empty)
	}
	```
3. Client compresses & encrypts data.
	1. Client uses GZip to compress JSON format of HTTP request
	2. Client encrypts the compressed request using shared AES key
	3. Client encodes encrypted data into Base64
4. Client constructs the full text to send by including additional headers:
	1. Version of this HTTP over SMS Tunnel Protocol (“01”), and delimiter (“-”)
	2. Magic string (“CRADLE”), and delimiter (“-”)
	3. Request # (6 digits), and delimiter (“-”)
	4. Total # of Fragments in this message (3 digits), and delimiter (“-”)
	5. HTTP request data (compressed, encrypted, Base64-encoded JSON data)  
		_Example (junk data):_
		> 01-CRADLE-008581-003-Rm9yIHNoYW1lIGRlbnkgdGhhdCB0aG91IGJlYXLigJlzdCBsb3ZlIHRvIGFueSwKV2hvIGZvciB0aHkgc2VsZiBhcnQgc28gdW5wcm92aWRlbnQuCkdyYW50LCBpZiB0aG91IHdpbHQsIHRob3UgYXJ0IGJlbG92ZWQgb2YgbWFueSwKQnV0IHRoYXQgdGhvdSBub25lIGxvduKAmXN0IGlzIG1vc3QgZXZpZGVudDoKRm9yIHRob3UgYXJ0IHNvIHBvc3Nlc3NlZCB3aXRoIG11cmRlcm91cyBoYXRlLApUaGF0IOKAmGdhaW5zdCB0aHkgc2VsZiB0aG91IHN0aWNr4oCZc3Qgbm90IHRvIGNvbnNwaXJlLApTZWVraW5nIHRoYXQgYmVhdXRlb3VzIHJvb2YgdG8gcnVpbmF0ZQpXaGljaCB0byByZXBhaXIgc2hvdWxkIGJlIHRoeSBjaGllZiBkZXNpcmUuCk8hIGNoYW5nZSB0aHkgdGhvdWdodCwgdGhhdCBJIG1heSBjaGFuZ2UgbXkgbWluZDoKU2hhbGwgaGF0ZSBiZSBmYWlyZXIgbG9kZ2VkIHRoYW4gZ2VudGxlIGxvdmU/CkJlLCBhcyB0aHkgcHJlc2VuY2UgaXMsIGdyYWNpb3VzIGFuZCBraW5kLApPciB0byB0aHlzZWxmIGF0IGxlYXN0IGtpbmQtaGVhcnRlZCBwcm92ZToKTWFrZSB0aGVlIGFub3RoZXIgc2VsZiBmb3IgbG92ZSBvZiBtZSwKVGhhdCBiZWF1dHkgc3RpbGwgbWF5IGxpdmUgaW4gdGhpbmUgb3IgdGhlZS4g
5. Client fragments the message to fit into SMS messages. Max fragment size is 306 characters (an SMS MultiPart message using two segments) (TBD if a 2-segment multipart SMS message is best for reliability, and efficiency)
	1. The first fragment can contain 306 characters, which includes our custom HTTP over SMS Tunnel Protocol header plus some data.
	2. Later fragments reserve 4 characters for the fragment number, leaving 302 characters for data.  
		_Example Fragmented Data (not including fragment numbers):_
		> 01-CRADLE-008581-003-Rm9yIHNoYW1lIGRlbnkgdGhhdCB0aG91IGJlYXLigJlzdCBsb3ZlIHRvIGFueSwKV2hvIGZvciB0aHkgc2VsZiBhcnQgc28gdW5wcm92aWRlbnQuCkdyYW50LCBpZiB0aG91IHdpbHQsIHRob3UgYXJ0IGJlbG92ZWQgb2YgbWFueSwKQnV0IHRoYXQgdGhvdSBub25lIGxvduKAmXN0IGlzIG1vc3QgZXZpZGVudDoKRm9yIHRob3UgYXJ0IHNvIHBvc3Nlc3NlZCB3aXRoIG11cmRl
		> cm91cyBoYXRlLApUaGF0IOKAmGdhaW5zdCB0aHkgc2VsZiB0aG91IHN0aWNr4oCZc3Qgbm90IHRvIGNvbnNwaXJlLApTZWVraW5nIHRoYXQgYmVhdXRlb3VzIHJvb2YgdG8gcnVpbmF0ZQpXaGljaCB0byByZXBhaXIgc2hvdWxkIGJlIHRoeSBjaGllZiBkZXNpcmUuCk8hIGNoYW5nZSB0aHkgdGhvdWdodCwgdGhhdCBJIG1heSBjaGFuZ2UgbXkgbWluZDoKU2hhbGwgaGF0ZSBiZSBmYWlyZXIgbG9kZ
		> 2VkIHRoYW4gZ2VudGxlIGxvdmU/CkJlLCBhcyB0aHkgcHJlc2VuY2UgaXMsIGdyYWNpb3VzIGFuZCBraW5kLApPciB0byB0aHlzZWxmIGF0IGxlYXN0IGtpbmQtaGVhcnRlZCBwcm92ZToKTWFrZSB0aGVlIGFub3RoZXIgc2VsZiBmb3IgbG92ZSBvZiBtZSwKVGhhdCBiZWF1dHkgc3RpbGwgbWF5IGxpdmUgaW4gdGhpbmUgb3IgdGhlZS4g
6. Client sends SMS messages (the fragments) one at a time to the Relay Server.
	1. First fragment includes tunnel header info plus some of the data. No fragment number.
	2. Format of 2nd (and later) fragments: `<3-digit fragment #>-<encrypted data; Base64 encoded>`  
	First fragment has no fragment number; 2nd fragment and onward numbered 001, 002, 003, …  
		_Example Fragment Packets (including fragment numbers)_  
		_(Header and fragment numbers shown in bold)_
		> **01-CRADLE-008581-003**-Rm9yIHNoYW1lIGRlbnkgdGhhdCB0aG91IGJlYXLigJlzdCBsb3ZlIHRvIGFueSwKV2hvIGZvciB0aHkgc2VsZiBhcnQgc28gdW5wcm92aWRlbnQuCkdyYW50LCBpZiB0aG91IHdpbHQsIHRob3UgYXJ0IGJlbG92ZWQgb2YgbWFueSwKQnV0IHRoYXQgdGhvdSBub25lIGxvduKAmXN0IGlzIG1vc3QgZXZpZGVudDoKRm9yIHRob3UgYXJ0IHNvIHBvc3Nlc3NlZCB3aXRoIG11cmRl
		> **001**-cm91cyBoYXRlLApUaGF0IOKAmGdhaW5zdCB0aHkgc2VsZiB0aG91IHN0aWNr4oCZc3Qgbm90IHRvIGNvbnNwaXJlLApTZWVraW5nIHRoYXQgYmVhdXRlb3VzIHJvb2YgdG8gcnVpbmF0ZQpXaGljaCB0byByZXBhaXIgc2hvdWxkIGJlIHRoeSBjaGllZiBkZXNpcmUuCk8hIGNoYW5nZSB0aHkgdGhvdWdodCwgdGhhdCBJIG1heSBjaGFuZ2UgbXkgbWluZDoKU2hhbGwgaGF0ZSBiZSBmYWlyZXIgbG9kZ
		> **002**-2VkIHRoYW4gZ2VudGxlIGxvdmU/CkJlLCBhcyB0aHkgcHJlc2VuY2UgaXMsIGdyYWNpb3VzIGFuZCBraW5kLApPciB0byB0aHlzZWxmIGF0IGxlYXN0IGtpbmQtaGVhcnRlZCBwcm92ZToKTWFrZSB0aGVlIGFub3RoZXIgc2VsZiBmb3IgbG92ZSBvZiBtZSwKVGhhdCBiZWF1dHkgc3RpbGwgbWF5IGxpdmUgaW4gdGhpbmUgb3IgdGhlZS4g
7. Relay server receives a fragment and adds it to its data structure to later reassemble the complete request.
8. Relay Server acknowledges the fragment by sending back an SMS message to the Client using the same header info as original message, but body is ACK:  
	`01-CRADLE-008581-000-ACK` 	(response to initial message)  
	`01-CRADLE-008581-001-ACK` 	(to fragment 001)  
	`01-CRADLE-008581-002-ACK` 	(to fragment 002)  
9. If the Relay Server has now received all fragments of the full request, it reassembles them into a full (encrypted, compressed, Base64 encoded) message and sends it to the Server’s SMS relay endpoint (REST API).
	1. Data transmitted to the server is still compressed, encrypted, and Base64 encoded.
	2. Format of body of the request to the server:  
	
	```
	{
	   "phoneNumber": "555-555-5555",		(Phone number of the Client)
	   "encryptedData": "<Base64 encrypted data>"
	}
	```
10. Timeouts
	1. If the Client does not receive an ACK within 3 seconds (after sending message finishes) then it will retransmit the message up to 6 re-transmission attempts. Each retransmission attempt doubles the time of the previous wait, up to a maximum of 30s wait.
		1. This retry functionality also exists with the exact same behavior when the SMS relay server attempts to send the data to the Flask server, and when the SMS relay server attempts to send the response data from the Flask server back to the mobile client
	2. After these retransmission attempts if the Client still does not get an ACK, it will cancel the operation.
	3. If the Relay Server does not receive a new fragment to an HTTP request it is assembling within 200 seconds of the most previous fragment then it records this as a failed request and discards it.

## 3. Server Processing
1. Server’s SMS Relay endpoint receives the REST API request
2. Server looks up the user by phone number and decrypts the encryptedData field using the stored AES key.
	1. If the message fails to decode from Base64 encoding, then the server replies with an unencrypted error message:  
	`“Server detected invalid message format (Base64); message may have been corrupted. Retry the action or contact your administrator.”`
	2. If the message fails to decrypt, or unzip then the server replies with an error message in unencrypted data:  
	`“Unable to verify message from {{phoneNumber}}. Either the phone number is not associated with a user, or the App and server don’t agree on the security key, or the message was corrupted. Retry the action or resync with the server using an internet connection (WiFi, 3G, …)”`
3. Server processes the decrypted Json Request Structure. It will:
	1. Check the request number to ensure it is valid. If invalid (see Request Number section below) then reply with:
		```
		HTTP Response Status: 425 (“Too Early”).
		Body: “Invalid request number; expected {{expected request number}}”
		```
4. Using the decrypted JSON Request Structure, the server executes the request.
	1. The request is executed using the Client user’s credentials; not the SMS Relay Server’s user’s credentials.
5. Server completes the operation leading to either:
	1. a successful conclusion and generates a successful HTTP status code, plus any required data (likely in JSON format).
	2. an error condition and generates an HTTP error status code, plus a meaningful error message to show to the end user (if possible).  
	
	Possible error conditions to be aware of include:  
	- Authorization failure: user is not authorised to use this endpoint / operation.  
	
	**Response String**  
	* If decryption failed, then we cannot encrypt the response because there is no agreed on encryption. Therefore the reply may be a single clear-text message:  
	```Some unencrypted error message```  
	* All other response (success or failure) will be encoded in the following JSON structure:  
		```
		{
		   "code": "200",
		   "body": "..." (may be a JSON object, or a quoted string)
		}
		```
	Then the message will be compressed, encrypted, and converted to Base64 yielding a string.
	
## Response Process
1. The Server’s HTTP response to the SMS Relay app may be:
	- An HTTP error for the SMS Relay app, such as the SMS relay app’s REST API call failed because the SMS Relay app did not have a valid user logged in.
	- An HTTP 200 response (OK) with a body that is a string to deliver to the Client. The body may be a plaintext error message (such as for an encryption failure), or it may be a (compressed, encrypted, and converted to Base64) JSON Response Structure. 
2. Send a response to the Client
	1. If the response was an HTTP error, or had a timeout (60s) trying to contact the Server  (i.e., the SMS Relay app had a problem talking to the Server), then it sends a reply to the Client with a plaintext error message: “ERROR: SMS Relay App unable to communicate with the server {{Server URL}}. Please contact an administrator.”
	2. If the response was “success” (i.e., the Server processed the Client’s request and it either worked or failed), then the SMS Relay Server will fragment the data, and use the same request number as the Client used when sending in the request.
		1. Relay Server will ignore replies which come in more than 60s after the Relay Server has sent a request to the server.  
		Reply format for Error:  
		```<Version>-CRADLE-<Request #>-REPLY_ERROR-<# Fragments total>-ERR<HTTP Status code>-<Base64 data>```

		Reply format for Success:  
		```<Version>-CRADLE-<Request #>-REPLY-<# Fragments total>-<Base64 data>```

		Example for Error:  
		```01-CRADLE-008581-REPLY_ERROR-001-ERR400-Sorry, something went wrong.```

		Example for Success (reply big enough to need 2 fragments):  
		```01-CRADLE-008581-REPLY-002-<Base64 data>```  
		
		Messages are fragmented the same way as described above, if needed.
3. Client receives each fragment, one-by-one, and acknowledges it using the same protocol used to send SMS fragments from the Client to the SMS Relay App.
4. Once Client has received full message:
	1. Client tries to process the reply data (decode Base64, decrypt, and unzip).
	2. If data cannot be decoded as Base64, it assumes data is plaintext and shows it to the user (perhaps limiting the length of what is shown).
	3. If another error occurs, it prints an error message.

## Request Numbers
1. Server stores the next expected request number for each User's phone number; Initially, the Server sets the expected request number to be `0`, meaning the Client's first request should use request number `0`.
	- Storing request numbers by phone number rather than by user reduces the amount of failures in scenarios where a user may send in requests from different devices.
3. Request number is included in both the plain-text SMS header, and in the encrypted message body. Both of these numbers should be identical to ensure consistency.
4. SMS Relay app uses the plain-text SMS header’s request number to:
	- Identify which request each received fragment belongs to.
	- Track when the user sends a new request.
5. Server uses encrypted request number to guard against replay attacks:
	1. Server stores the next expected request number per user.
	2. Each request from a user must have a request number that is within the range of `0 to 100` above the expected request number.
		
		- Allowing the server to accept a range of request numbers reduces the amount of failures by minimizing the amount of request number mismatches 		and prevents unnecessary retransmissions. It also helps handle cases where request numbers become out of sync due to cancelled or failed requests.
	3. Request numbers may wrap-around after the maximum value (`Rmax`) and start back at `0`. This wrap-around is treated as preserving the greater-than relationship.  
	
		We determine if a request number is valid using the formula:

			0 <= (Rc - Re) mod (Rmax + 1) <= 100

			Where:
			- Re = expected request number from the Client
			- Rc = current request number received from the Client
			- Rmax = maximum request number representable (`999,999` by default)
		This formula ensures that request numbers are valid even after wrapping around.

		Examples:

		- Basic Scenario:
			
			```If the expected request number from the user is `1000`, the server will accept request numbers from `1000` to `1100`.```

		- Wrap-Around Scenario  (Assume Rmax is `999,999`): 
			
			```If the Server's expected request number for the Client is `999,998`, it will accept request numbers in the range [999,998, 999,999] and [0,98].```
		
	4. If a request uses an invalid request number, the Server sends an encrypted error response with the expected request number. The Client can then resume by 		sending requests from the expected request number and incrementing from there.

	5. There is no need to have a mechanism to reset the request numbers because the protocol can handle when a user’s request number wraps around. 
	
6. On Mobile (Client side), the request number increments with every request sent, regardless of whether the server response was a success or failure. The request number also increments even if the request is cancelled or fails before receiving a response from the server. This ensures every unique request has a different request number.

## Future Ideas
1. Allow mechanism for Client and Server to exchange encryption key without needing HTTPS connection.
	1. Use full RSA key distribution protocol (like uses HTTPS) to exchange temporary encryption tokens
	2. This saves us from having to sync keys during the sync process and may allow full operation over SMS all the time (including setup)
