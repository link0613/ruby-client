# BitPay Library for Ruby [![](https://secure.travis-ci.org/bitpay/ruby-client.png)](http://travis-ci.org/bitpay/ruby-client) [![Gem Version](https://badge.fury.io/rb/bitpay-sdk.svg)](http://badge.fury.io/rb/bitpay-sdk)
Powerful, flexible, lightweight interface to the BitPay Bitcoin Payment Gateway API.

The `bitpay-sdk` gem provides all the programattic tools required to implement a ruby client application for the BitPay REST API.  For developers who prefer the ease of command-line pairing during the development or deployment process, BitPay provides a complementary [Ruby CLI gem](https://github.com/bitpay/ruby-cli) which can be used in conjunction with this gem.

## Installation

```bash
gem install bitpay-sdk
```

In your Gemfile:

```ruby
gem 'bitpay-sdk', :require => 'bitpay_sdk'
```

Or directly:
```ruby
require 'bitpay_sdk'
```

## Configuration

The bitpay client creates a cryptographically secure connection to your server by pairing an API code with keys generated by the library. The client can be initialized with pre-existing keys passed in as a pem file, or paired if initialized with a pem file and a tokens hash. Examples can be found in the cucumber step helpers.

## Client Setup

### Pairing with BitPay.com

Most calls to the BitPay REST API require that your client is paired with the bitpay.com server.  To pair with bitpay.com you need to have an approved merchant account.

Your client can be paired via the `pos` (point-of-sale) or `merchant` facade (or both).  The `pos` facade allows for invoices to be created.  The `merchant` facade has broader privileges to view all invoices, bills, and ledger entries, as well as to issue refunds.  Consider the level of access required when you pair your client.

_For development or quick deployment purposes, consider the [BitPay Ruby Command-Line Interface](https://github.com/bitpay/ruby-cli) to simplify the deployment process_

### Pairing Programattically

If you are developing a client with built-in pairing capability, you can pair programattically using the `pair_client` method.  This method can be called in two ways:

  * `pair_client()` will perform a client-initiated pairing, and will provide a pairing code that can be entered at https://bitpay.com/dashboard/merchant/api-tokens to assign either `merchant` or `pos` facade.
  * `pair_client('pairing_code')` will complete a server-initiated pairing, when provided a pre-generated pairing code from https://bitpay.com/dashboard/merchant/api-tokens.  In this case, the `pos` facade will be automatically assigned.

The example below demonstrates this using a locally generated PEM file using OpenSSL and the irb tool.

```bash
$ gem install bitpay-sdk
Successfully installed bitpay-sdk-2.2.0
1 gem installed
$ openssl ecparam -genkey -name secp256k1 -noout -out bitpaykey.pem
$ irb
2.1.1 :001 > require 'bitpay_sdk'
 => true 
2.1.1 :002 > client = BitPay::SDK::Client.new(api_uri: 'https://test.bitpay.com', pem: File.read('bitpaykey.pem'), insecure: true)
 => #<BitPay::SDK::Client:0x000000019c6d40 @pem="---... @tokens={}> 
2.1.1 :003 > client.pair_client()
 => {"data"=>[{"policies"=>[{"policy"=>"id", "method"=>"inactive", "params"=>["Tf49SFeiUAtytFEW2EUqZgWj32nP51PK73M"]}], "token"=>"BKQyVdaGQZAArdkkSuvtZN5gcN2355c8vXLj5eFPkfuK", "dateCreated"=>1422474475162, "pairingExpiration"=>1422560875162, "pairingCode"=>"Vy76yTh"}]} 
```

As described above, using the value from the `pairingCode` element, visit https://test.bitpay.com/api-tokens and search to register for the appropriate facade

## General Usage

### Initialize the client

```ruby
client = BitPay::SDK::Client.new(pem: File.read('bitpaykey.pem')
```
    
Optional parameters:
 * `api_uri` - specify a different api endpoint (e.g. 'https://test.bitpay.com').  Ensure no trailing slash.
 * `tokens` - pass a stored hash of bitpay API tokens
 * `user-agent` - specify a custom user-agent value
 * `debug: true` - enable HTTP request logging to $stdout
 * `insecure: true` - disable HTTPs certificate validation (for local test environments)

### Create a new bitcoin invoice

```ruby
invoice = client.create_invoice (price: <price>, currency: <currency>)
```

With invoice creation, `price` and `currency` are the only required fields. If you are sending a customer from your website to make a purchase, setting `redirectURL` will redirect the customer to your website when the invoice is paid.

Response will be a hash with information on your newly created invoice. Send your customer to the `url` to complete payment:

```javascript
{
  "url": "https://bitpay.com/invoice?id=NKaqMuZWy3BAcP77RdkEEv",
  "paymentUrls": {
    "BIP21": "bitcoin:mvYRECDxKPaPHnjNz9ZxiTpbx29xYNoRy4?amount=0.3745",
    "BIP72": "bitcoin:mvYRECDxKPaPHnjNz9ZxiTpbx29xYNoRy4?amount=0.3745&r=https://bitpay.com/i/NKaqMuZWy3BAcP77RdkEEv",
    "BIP72b": "bitcoin:?r=https://bitpay.com/i/NKaqMuZWy3BAcP77RdkEEv",
    "BIP73": "https://bitpay.com/i/NKaqMuZWy3BAcP77RdkEEv"
  },
  "status": "new",
  "btcPrice": "0.3745",
  "btcDue": "0.3745",
  "price": 148,
  "currency": "USD",
  "exRates": {
    "USD": 395.20000000000005
  },
  "invoiceTime": 1415987168612,
  "expirationTime": 1415988068612,
  "currentTime": 1415987168629,
  "guid": "438e8237-fff1-483c-81b4-dc7dba28922a",
  "id": "NKaqMuZWy3BAcP77RdkEEv",
  "transactions": [

  ],
  "btcPaid": "0.0000",
  "rate": 395.2,
  "exceptionStatus": false,
  "token": "9kZgUXFb5AC6qMuLaMpP9WopbM8X2UjMhkphKKdaprRbSKgUJNE6JNTX8bGsmgxKKv",
  "buyer": {
  }
}
```

There are many options available when creating invoices, which are listed in the [BitPay API documentation](https://bitpay.com/bitcoin-payment-gateway-api).

### Get invoice status
The ruby library provides two methods for fetching an existing invoice:

```ruby
# For authorized clients with a 'merchant' token
client.get_invoice(id: 'PvVhgBfA7wKPWhuVC24rJo')
    
# For non-authenticated clients (public facade)
# Returns the public subset of invoice fields
client.get_public_invoice(id: 'PvVhgBfA7wKPWhuVC24rJo')
```

### Create a refund request

Clients with a `merchant` token can initiate a refund request for a paid invoice:

```ruby
client.refund_invoice(id: '6pbV13VBZfGFJ8BBmXmLZ8', params: {amount: 10, currency: 'USD'})
```
    
Refund rules:

 * Invoices cannot be refunded prior to 6 blockchain confirmations
 * Invoices without `["flags"]["refundable"] == true` must specify a `bitcoinAddress` param (one was not provided as part of the transaction)
 * Invoices that are paid in full must specify an `amount` and `currency` param to indicate the amount to be refunded

### View Refund Requests

The ruby library provides two methods for viewing refund requests.  Both require a `merchant` token.

```ruby
# To get an array of all refunds against a specific invoice
client.get_all_refunds_for_invoice(id: 'PvVhgBfA7wKPWhuVC24rJo')
    
# To get a specific refund for a specific invoice
client.get_refund(id: 'JB49z2MsDH7FunczeyDS8j', request_id: '4evCrXq4EDXk4oqDXdWQhX')
```

### Make a HTTP request directly against the REST API

For API tasks which lack a dedicated library method, BitPay provides a method that will automatically apply the proper cryptographic parameters to a request.

```ruby
client.send_request("GET", "/invoices/JB49z2MsDH7FunczeyDS8j", facade: 'merchant')
```
    
Usage:
 * Specify HTTP verb and REST endpoint
 * Specifying a `facade` will fetch and apply the corresponding `token`
 * Alternatively provide a `token` explicitly
 * For `POST` requests, the `params` hash will be included as the message body

## Testnet Usage

During development and testing, take advantage of the [Bitcoin TestNet](https://en.bitcoin.it/wiki/Testnet) by passing a custom `api_uri` option on initialization:

```ruby
BitPay::SDK::Client.new({api_uri: "https://test.bitpay.com/api"})
```

Note that in order to pair with testnet, you will need a pairing code from test.bitpay.com and will need to use the bitpay client with the --test option.

## API Documentation

API Documentation is available on the [BitPay site](https://bitpay.com/api).

## Running the Tests

In order to run the tests, you must have phantomjs installed and on your PATH.

The tests require that environment variables be set for the bitpay server, user name, and password. First run:

```bash 
$ source ./spec/set_constants.sh https://test.bitpay.com <yourusername> <yourpassword>
$ bundle install
$ bundle exec rake
```

Tests are likely to run up against rate limiters on test.bitpay.com if used too frequently. Rake tasks which interact directly with BitPay will not run for the general public.

## Found a bug?
Let us know! Send a pull request or a patch. Questions? Ask! We're here to help. We will respond to all filed issues.

## Contributors
[Click here](https://github.com/bitpay/ruby-client/graphs/contributors) to see a list of the contributors to this library.
