# NuVotifier

NuVotifier is a Bukkit plugin whose purpose is to be notified (aka *votified*) when a vote is made on a Minecraft server top list for the server.  NuVotifier creates a *lightweight* server that waits for connections by Minecraft server lists and uses a simple protocol to get the required information.  NuVotifier is *secure*, and makes sure that all vote notifications are delivered by authentic top lists.

[![Build Status](https://travis-ci.org/NuVotifier/NuVotifier.svg?branch=master)](https://travis-ci.org/NuVotifier/NuVotifier)

## Why fork Votifier?

Votifier has lagged in activity, has severe security flaws in its protocol, and is vulnerable to denial of service attacks.

### Is NuVotifier a drop-in replacement for Votifier?

Yes. NuVotifier is compatible with Votifier server lists and Votifier listeners, with some differences:

* The `Vote` object's setters are deprecated.
* Support for standalone vote listeners (`.class` files) has been removed, as handling `VotifierEvent` is more flexible.

### Why a new protocol?

The old Votifier protocol has three major flaws, which we feel are corrected in the new protocol.

* The data is encrypted (with RSA) but not hashed to verify its integrity. The new protocol utilizes HMAC with SHA256 (but no encryption, given that the data sent is eventually revealed in some form or another).
* The canonical Votifier protocol is vulnerable to replay attacks.
* The canonical Votifier is one key for all clients, which is weak. While NuVotifier still supports a default key (for the less paranoid), this ability can be disabled and force per-server keys.

## Configuring NuVotifier

NuVotifier configures itself the first time it is run. If you have an existing Votifier configuration, NuVotifier will copy it over and use it.

If you want to customize NuVotifier, simply the edit `./plugins/(Nu)Votifier/config.yml` file.

An in-depth configuration guide is located in the [usage](https://github.com/NuVotifier/NuVotifier/blob/master/usage.md) document.

### Tokens

In `config.yml`, you will notice a `tokens` section. NuVotifier has a new protocol version that requires tokens. The `tokens` section can be used to easily grant or revoke access to a server list based on the keys it uses.

### RSA Keys

RSA keys for use with the old Votifier protocol are available under the `plugins/(Nu)Votifier/rsa` directory.

### Vote Forwarding

Starting from NuVotifier 2.2, there is a feature to forward votes from NuVotifier running on BungeeCord to your Bukkit servers without complicated setup.

You must enable the feature on both your Bukkit and BungeeCord servers. This feature is still in beta but is stable.

## Building NuVotifier

NuVotifier can be built by running `mvn clean package` from within the root directory of NuVotifier. A universal Bukkit/Spigot & BungeeCord plugin will be built to `universal/target/nuvotifier-universal-{version}.jar`

## Protocol Documentation

This documentation is for server lists that wish to add NuVotifier support.

There are two versions of the NuVotifier protocol.

### Handshake

A connection is made to the NuVotifier server by the server list, and immediately NuVotifier will send its version in the following packet:

	"VOTIFIER <version> <challenge>"

The challenge will not be present in version 1.x servers.

### Protocol v2

NuVotifier expects a message composed of `0x733A` (16-bit big-endian integer) and the length of the following JSON message (as a 16-bit big-endian integer) sent plus the message.

The message is JSON-encoded containing the following data:

* `payload` contains the fully encoded vote. It is a JSON message containing the `serviceName` (string), `username` (string), `address` (string, but must be a valid IPv4 or IPv6 address), `timestamp` (a long with the time this vote was taken in _milliseconds_), and `challenge` (from the handshake). It is optional, but recommended to supply a `uuid` field with the player's UUID (with dashes, e.g. `652a2bc4-e8cd-405d-b7b6-98156ee2dc09`).
* `signature` is a HMAC-SHA256 digest of the payload with a owner-provided key.

A response is given after the vote has been received and parsed, but not yet processed.

For your convenience, the NuVotifier team has provided client libraries in [JavaScript](https://github.com/NuVotifier/votifier2-js), [PHP](https://github.com/NuVotifier/votifier2-php) and [Go](https://github.com/minecrafter/go-votifier). If you are stuck, these libraries may help you.

### Protocol v1 (deprecated)

For better compatibility, NuVotifier supports the old Votifier protocol as well.

NuVotifier expects a 256 byte RSA encrypted block (the public key should be obtained by the NuVotifier user), with the following format:

<table>
  <tr>
	<th>Type</th>
	<th>Value</th>
  </tr>
  <tr>
	<td>string</td>
	<td>VOTE</td>
  </tr>
  <tr>
	<td>string</td>
	<td>serviceName</td>
  </tr>
  <tr>
	<td>string</td>
	<td>username</td>
  </tr>
  <tr>
	<td>string</td>
	<td>address</td>
  </tr>
  <tr>
	<td>string</td>
	<td>timeStamp</td>
  </tr>
</table>

The first string of value "VOTE" is an opcode check to ensure that RSA was encoded and decoded properly, if this value is wrong then NuVotifier assumes that there was a problem with encryption and drops the connection. `serviceName` is the name of the top list service, `username` is the username (entered by the voter) of the person who voted, `address` is the IP address of the voter, and `timeStamp` is the time stamp of the vote.  Each string is delimited by the newline character `\n` (byte value 10).  The `space` block is the empty space that is left over, **the block must be exactly 256 bytes** regardless of how much information it holds.