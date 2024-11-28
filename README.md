# Disclaimer 
This WebAuthn library supports the PQC Dilithium algorithm at levels 2, 3, and 5. In order to use this functionality, authorization for the Go version of the Crystals-Dilithium encryption algorithm library must be obtained. Additionally, the library must be installed in the Go library path, typically located at /go/src/crystals-dilithium. The actual path may vary depending on the user's specific Go project configuration.

# WebAuthn Library

[![GoDoc](https://godoc.org/github.com/go-webauthn/webauthn?status.svg)](https://godoc.org/github.com/go-webauthn/webauthn)
[![Go Report Card](https://goreportcard.com/badge/github.com/go-webauthn/webauthn)](https://goreportcard.com/report/github.com/go-webauthn/webauthn)


This library is meant to handle [Web Authentication](https://www.w3.org/TR/webauthn) for Go apps that wish to implement 
a passwordless solution for users. This library conforms as much as possible to the guidelines and implementation
procedures outlined by the document.

## Fork

This library is a hard fork of [github.com/duo-labs/webauthn] and is the natural successor to that library.

See the [migration](MIGRATION.md) guide for more information about how to migrate and the differences between the
libraries.

It is distributed under the same 3-Clause BSD license as the original fork, with the only amendment being the additional
3-Clause BSD license attributing license rights to this repository.

## Go Version Support Policy

This library will officially support versions of go which are currently supported by the go maintainers (usually 3
minor versions) with a brief transition time (usually 1 patch release of go, for example if go 1.21.0 is released, we
will likely still support go 1.17 until go 1.21.1 is released). 

This library in our opinion handles a critical element of security in a dependent project and we aim to avoid backwards
compatability at the cost of security wherever possible. We also consider this especially important in a language like
go where their backwards compatibility when upgrading the compile tools is usually flawless.

This policy means that users who wish to build this with older versions of go may find there are features being used
which are not available in that version. The current intentionally supported versions of go are as follows:

- go 1.21
- go 1.20
- go 1.19
- go 1.18

## Status

This library is still version 0, as per Semantic Versioning 2.0 rules there may be breaking changes without warning. 
While we strive to avoid such changes and strive to notify users they may be unavoidable.

## Quickstart

`go get github.com/go-webauthn/webauthn` and initialize it in your application with basic configuration values. 

Make sure your `user` model is able to handle the interface functions laid out in `webauthn/user.go`. This means also 
supporting the storage and retrieval of the credential and authenticator structs in `webauthn/credential.go` and 
`webauthn/authenticator.go`, respectively.

## Examples

The following examples show some basic use cases of the library. For consistency sake the following variables are used
to denote specific things:

- Variable `webAuthn`: the `webauthn.WebAuthn` instance you initialize elsewhere in your code
- Variable `datastore`: the pseudocode backend service that stores your webauthn session data and users such as PostgreSQL 
- Variable `session`: the webauthn.SessionData object
- Variable `user`: your webauthn.User implementation

We try to avoid using specific external libraries (excluding stdlib) where possible, and you'll need to adapt these
examples with this in mind.

### Initialize the request handler

```go
package example

import (
	"fmt"
	"time"

	"github.com/go-webauthn/webauthn/webauthn"
)

var (
	webAuthn *webauthn.WebAuthn
	err error
)

// Your initialization function
func main() {
	wconfig := &webauthn.Config{
		RPDisplayName: "Go Webauthn", // Display Name for your site
		RPID: "go-webauthn.local", // Generally the FQDN for your site
		RPOrigins: []string{"https://login.go-webauthn.local"}, // The origin URLs allowed for WebAuthn requests
	}
	
	if webAuthn, err = webauthn.New(wconfig); err != nil {
		fmt.Println(err)
	}
}
```

### Registering an account

```go
package example

func BeginRegistration(w http.ResponseWriter, r *http.Request) {
	user := datastore.GetUser() // Find or create the new user  
	options, session, err := webAuthn.BeginRegistration(user)
	// handle errors if present
	// store the sessionData values 
	JSONResponse(w, options, http.StatusOK) // return the options generated
	// options.publicKey contain our registration options
}

func FinishRegistration(w http.ResponseWriter, r *http.Request) {
	user := datastore.GetUser() // Get the user
	
	// Get the session data stored from the function above
	session := datastore.GetSession()
		
	credential, err := webAuthn.FinishRegistration(user, session, r)
	if err != nil {
		// Handle Error and return.

		return
	}
	
	// If creation was successful, store the credential object
	// Pseudocode to add the user credential.
	user.AddCredential(credential)
	datastore.SaveUser(user)

	JSONResponse(w, "Registration Success", http.StatusOK) // Handle next steps
}
```

### Logging into an account

```go
package example

func BeginLogin(w http.ResponseWriter, r *http.Request) {
	user := datastore.GetUser() // Find the user
	
	options, session, err := webAuthn.BeginLogin(user)
	if err != nil {
		// Handle Error and return.

		return
	}
	
	// store the session values
	datastore.SaveSession(session)
	
	JSONResponse(w, options, http.StatusOK) // return the options generated
	// options.publicKey contain our registration options
}

func FinishLogin(w http.ResponseWriter, r *http.Request) {
	user := datastore.GetUser() // Get the user 
	
	// Get the session data stored from the function above
	session := datastore.GetSession()
	
	credential, err := webAuthn.FinishLogin(user, session, r)
	if err != nil {
		// Handle Error and return.

		return
	}
	
	// If login was successful, handle next steps
	JSONResponse(w, "Login Success", http.StatusOK)
}
```

## Modifying Credential Options

You can modify the default credential creation options for registration and login by providing optional structs to the 
`BeginRegistration` and `BeginLogin` functions. 

### Registration modifiers

You can modify the registration options in the following ways:

```go
package example

import (
	"github.com/go-webauthn/webauthn/protocol"
	"github.com/go-webauthn/webauthn/webauthn"
)

var webAuthn webauthn.WebAuthn // init this in your init function

func beginRegistration() {
	// Updating the AuthenticatorSelection options. 
	// See the struct declarations for values
	authSelect := protocol.AuthenticatorSelection{
		AuthenticatorAttachment: protocol.AuthenticatorAttachment("platform"),
		RequireResidentKey: protocol.ResidentKeyNotRequired(),
		UserVerification: protocol.VerificationRequired,
	}

	// Updating the ConveyencePreference options. 
	// See the struct declarations for values
	conveyancePref := protocol.PreferNoAttestation

	user := datastore.GetUser() // Get the user  
	opts, session, err := webAuthn.BeginRegistration(user, webauthn.WithAuthenticatorSelection(authSelect), webauthn.WithConveyancePreference(conveyancePref))

	// Handle next steps
}
```

### Login modifiers

You can modify the login options to allow only certain credentials:

```go
package example

import (
	"github.com/go-webauthn/webauthn/protocol"
	"github.com/go-webauthn/webauthn/webauthn"
)

var webAuthn webauthn.WebAuthn // init this in your init function

func beginLogin() {
	// Updating the AuthenticatorSelection options. 
	// See the struct declarations for values
	allowList := make([]protocol.CredentialDescriptor, 1)
	allowList[0] = protocol.CredentialDescriptor{
		CredentialID: credentialToAllowID,
		Type: protocol.CredentialType("public-key"),
	}

	user := datastore.GetUser() // Get the user  

	opts, session, err := w.BeginLogin(user, webauthn.WithAllowedCredentials(allowList))

	// Handle next steps
}
```

## Timeout Mechanics

The library by default does not enforce timeouts. However the default timeouts sent to the browser are taken from the
specification. You can override both of these behaviours however.

```go
package example

import (
	"fmt"
	"time"
	
	"github.com/go-webauthn/webauthn/protocol"
	"github.com/go-webauthn/webauthn/webauthn"
)

func main() {
	wconfig := &webauthn.Config{
		RPDisplayName: "Go Webauthn",                               // Display Name for your site
		RPID:          "go-webauthn.local",                         // Generally the FQDN for your site
		RPOrigins:     []string{"https://login.go-webauthn.local"}, // The origin URLs allowed for WebAuthn requests
		Timeouts: webauthn.TimeoutsConfig{
			Login: webauthn.TimeoutConfig{
				Enforce:    true, // Require the response from the client comes before the end of the timeout.
				Timeout:    time.Second * 60, // Standard timeout for login sessions.
				TimeoutUVD: time.Second * 60, // Timeout for login sessions which have user verification set to discouraged.
			},
			Registration: webauthn.TimeoutConfig{
				Enforce:    true, // Require the response from the client comes before the end of the timeout.
				Timeout:    time.Second * 60, // Standard timeout for registration sessions.
				TimeoutUVD: time.Second * 60, // Timeout for login sessions which have user verification set to discouraged.
			},
		},
	}
	
	webAuthn, err := webauthn.New(wconfig)
	if err != nil {
		fmt.Println(err)
	}
}
```

## Acknowledgements

We graciously acknowledge the original authors of this library [github.com/duo-labs/webauthn] for their amazing
implementation. Without their amazing work this library could not exist.


[github.com/duo-labs/webauthn]: https://github.com/duo-labs/webauthn
