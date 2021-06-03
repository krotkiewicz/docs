---
title: Heystack app
description: Interacting with the wallet and smart contracts from a React application
tags:
  - example-app
images:
  large: /images/pages/todo-app.svg
  sm: /images/pages/todo-app-sm.svg
---

## Introduction

This example application demonstrates important features of the Stacks blockchain, and is an example of how a frontend
web application can interact with a Clarity smart contract. The full source of the application is provided, this page
highlights important code snippets and design patterns to help you learn how to develop your own Stacks application.

This app highlights the following platform features:

- Authenticating users with the web wallet
- Using a smart contract to store data on the blockchain
- Minting new fungible tokens with a SIP-010 compliant smart contract
- Creating and monitoring transactions on the Stacks blockchain using Stacks.js

You can access the [online version][heystack] of the Heystack app to interact with it. The source for Heystack is also
available on [Github][heystack_gh]. This page assumes that you are familiar with [React][].

## Review smart contracts

Heystack depends on two smart contracts to execute the backend functions of the app on the Stacks blockchain: a contract
for handling the messaging content, and a contract for minting and distributing the $HEY token.

### Content contract

The `hey.clar` contract provides two primary functions for the application, a function to publish content to
the blockchain and a function to like a piece of content based on its ID. This section reviews the implementation of
these primary functions, but is not a comprehensive discussion of the contract.

In order to accomplish the two primary functions, the contract relies on a data variable `content-index` and two
[data maps][], `like-state` and `publisher-state` which contain the number of likes a piece of content has received, and
the principal address of the account that published the content.

Note that all variables are defined at the top of the contract, which is a requirement of the Clarity language. These
include constants such as the `contract-creator`, error codes, and a treasury address.

```clarity
;;
;; Data maps and vars
(define-data-var content-index uint u0)

(define-read-only (get-content-index)
  (ok (var-get content-index))
)

(define-map like-state
  { content-index: uint }
  { likes: uint }
)

(define-map publisher-state
  { content-index: uint }
  { publisher: principal }
)
```

Read-only functions provide a method for getting the like count of a piece of content, and getting the principal address
of the message publisher.

```clarity
(define-read-only (get-like-count (id uint))
  ;; Checks map for like count of given id
  ;; defaults to 0 likes if no entry found
  (ok (default-to { likes: u0 } (map-get? like-state { content-index: id })))
)

(define-read-only (get-message-publisher (id uint))
  ;; Checks map for like count of given id
  ;; defaults to 0 likes if no entry found
  (ok (unwrap-panic (get publisher (map-get? publisher-state { content-index: id }))))
```

The `get-like-count` method accepts a content ID and returns the number of likes associated with that content. The
method uses the [`default-to`][] function to return `0` if the content ID isn't found in the map of likes.

The `get-message-publisher` method accepts a content ID and returns the principal address of the content publisher. The
method uses the [`unwrap-panic`][] function to halt execution of the method if the principal address isn't found in
the map of publishers.

The two primary public methods are the `send-message` and `like-message` functions. These methods allow the contract
caller to store a message on the blockchain (creating entries in the data maps for the message sender and the number
of likes). Note that the message itself isn't stored in a contract variable, the frontend application reads the content
of the message directly from the transaction on the blockchain.

```clarity
;;
;; Public functions
(define-public (send-message (content (string-utf8 140)))
  (let ((id (unwrap! (increment-content-index) (err u0))))
    (print { content: content, publisher: tx-sender, index: id })
    (map-set like-state
      { content-index: id }
      { likes: u0 }
    )
    (map-set publisher-state
      { content-index: id }
      { publisher: tx-sender }
    )
    (transfer-hey u1 HEY_TREASURY)
  )
)
```

The `send-message` method accepts a utf-8 string with a maximum length of 140 characters. The method defines an internal
variable `id` using the `let` function and assigns the next content ID to that variable by calling the
`increment-contract-index` method of the contract. The value assignment of this variable is bounded by the [`unwrap!`][]
function, which returns an error and exits the control-flow if the `increment-contract-index` function isn't
successfully called.

The method then assigns `u0` likes to the content in the `like-state` data map, and adds the principal address to the
`publisher-state` data map using the [`map-set`][] function. Finally, the private method `transfer-hey` is called to
transfer 1 $HEY token from the message sender to the $HEY treasury address stored in the `HEY_TREASURY` constant.

```clarity
(define-public (like-message (id uint))
  (begin
    ;; cannot like content that doesn't exist
    (asserts! (>= (var-get content-index) id) (err ERR_CANNOT_LIKE_NON_EXISTENT_CONTENT))
    ;; transfer 1 HEY to the principal that created the content
    (map-set like-state
      { content-index: id }
      { likes: (+ u1 (get likes (unwrap! (get-like-count id) (err u0)))) }
    )
    (transfer-hey u1 (unwrap-panic (get-message-publisher id)))
  )
)
```

### Token contract

TK

## Authentication

TK

## Transactions

TK

### Clarity types in JS

TK

### Reading pending transactions

TK

[heystack]: https://heystack.xyz
[react]: https://reactjs.org/
[heystack_gh]: https://github.com/blockstack/heystack
[data maps]: /references/language-functions#define-map
[`default-to`]: /references/language-functions#default-to
[`unwrap-panic`]: /references/language-functions#unwrap-panic
[`unwrap!`]: /references/language-functions#unwrap
[`map-set`]: /references/language-functions#map-set
