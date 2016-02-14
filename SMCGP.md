%%%%%%%%%%%%%%%%%%%%%%%%%%%

    Title = "Short Message Chess Game Protocol"
    abbrev = "Short Message Chess Game Protocol"
    category = "info"
    docName = "smcgpv1-draft"
    ipr = "full3978"

    [pi]
    private = "yes"
    footer = ""
    header = "smcgp"

    date = 2016-02-14T00:00:00Z

    [[author]]
    initials="M."
    surname="Parks"
    fullname="Michael Parks"
    organization="TKWare Enterpises"
      [author.address]
      email = "mparks@tkware.info"
      phone = "+1 307 335 3132"


%%%%%%%%%%%%%%%%%%%%%%%%%%%


.# Abstract
This document describes the Short Message Chess Game Protocol, an application
layer protocol based on ASCII text to allow for standardized transmission of
chess gameplay over any limited bandwith medium, such as SMS, Twitter, packet
radio, or any other protocol where space is at a premium.

The key words "**MUST**", "**MUST** NOT", "REQUIRED", "SHALL", "SHALL NOT", "**SHOULD**",
"**SHOULD** NOT", "RECOMMENDED",  "**MAY**", and "OPTIONAL" in this document are to be
interpreted as described in [@RFC2119].

{mainmatter}

# Status of this Memo

This document is not an Internet Standards Track specification; it is
published for examination, experimental implementation, and
evaluation.

This document defines an Experimental Protocol for the Internet
community.  This is a contribution to the RFC Series,
independently of any other RFC stream.  The RFC Editor has chosen to publish this
document at its discretion and makes no statement about its value
for implementation or deployment.  Documents approved for publication
by the RFC Editor are not a candidate for any level of Internet
Standard; see Section 2 of [@RFC5741].

<!--
Information about the current status of this document, any
errata, and how to provide feedback on it may be obtained at
http://www.rfc-editor.org/info/rfc<rfc-no>.
-->

# Copyright Notice

Copyright (C) TKWare Enterprises (2014). Some rights reserved;
Licensed under the Creative Commons Attribution-Share Alike license.

# Introduction
Chess, being one of the world's most popular games, has a storied history going
back as far as the sixth century. A common pastime is "chess by mail", where
individual players would set up a board,  make their moves, write that move
down, and mail it to another player.

While innumerable applications of chess over Internet networks exist, this
document formalizes a short message protocol intended as a universally
applicable spec for long distance chess games over any data network.


#  Message Format

## Basic Layout

At a base level, a valid SMCGP message **MUST** contain the following, each element separated by ASCII colon ":" characters, including the end of the message:

1. Six (6) character randomly chosen identifier.

2. Two (2) character action flag.

3. Twenty or less (<=20) character description of the action being taken, the exact format varies by the type of message. Some messages add additional fields.

4. Any number of any other characters to be used for commentary or future expansion. Version 1 of this protocol allows this field to be used for any purpose.

In any case, the total length of the SMCGP message **MUST** NOT exceed fifty (50) bytes.

## Identifier

The identifier string is composed of six (6) randomly chosen alphanumeric characters of either casing. In other words, it **MUST** match the regular expression `[A-Za-z0-9]{6}` This allows for 2,176,782,336 unique possible identifiers.

A client **SHOULD** choose these characters at random using a good source of entropy.

A client **MAY** choose to either ignore, or reject with the action flag "ER"
any message which:

1. Omits the identifier

2. Uses an identifier that is not exactly six (6) characters in length.

A client **MUST** reject with the action flag `ER` any SMCGP message which uses
an identifier that is identical to another identifier being used on the same
channel, frequency, group message, etc.

*Note*: The identifier string **MUST** NOT be taken to identify a player, it is only used for identifying a conversation. This allows it to be used in broadcast environments. SMCGP is only concerned with facilitating transmission of game moves, not the identity of the players.

## Action Flags

The following types of action flags **MUST** be supported by any compliant client.

### CH (Challenge)

A challenge message indicates that the player is looking for a match. The
message **MUST** be formatted as:

    IDENTIFIER:CH:RATING:COLOR:COMMENTS

`RATING` is the numeric FIDE rating of the player submitting the challenge. A player who is not FIDE rated, or who does not wish to play this match for a rating change, **MUST** transmit either `UNR`, or no characters at all in this field.

`COLOR` is the color side the player making the challenge wishes to play. A client **MUST** use a single character for this field, either:

* `W` (white)
* `B` (black)

If the client making the request does not care what color they play, the client **MUST** still choose either W or B - the client **MAY** flip a coin or use another good source of entropy to select randomly.

### CA (Challenge Accepted)
A Challenge Accepted message is **MUST** be sent in reply to a CH message, indicating acceptance of the proposed match. A CA message **MUST** be formatted as:

    IDENTIFIER:CA:NEWIDENTIFIER

Where `NEWIDENTIFIER` is another randomly chosen identifier as laid out above.


### AA (Acceptance Acknowledged)
An Acceptance Acknowledge message **MUST** be sent in reply to a CA message,
indicating reception of the new identifier. An AA message **MUST** be formatted as:

    IDENTIFIER:AA:NEWIDENTIFIER

Where `IDENTIFIER` is the old identifier, and `NEWIDENTIFIER` is the new
identifier transmitted in the previous CA message.

After transmitting a CA message, all further messages by either party **MUST** be
transmitted with the newly chosen identifier. After the CA message and the
first move has been taken, the game is considered to be `ACTIVE`.

### MV (Move)
A Move message indicates any valid move according to the laws of chess [@LAW].

A MV message **MUST** be formatted as:

    IDENTIFIER:MV:MOVETEXT

Where `MOVETEXT` is further formatted as algebraic chess notation according to the FIDE Standard Algebraic Notation [@LAW].

A MV message **MUST** only be sent in response to another MV message. If retransmission is required, a RT message, described in (#rt-retransmit) can be used.

A MV message sent out of turn **MUST** be responded to with an ER message.

### DR (Draw Request)
A Draw Request message is a request to terminate the current game, awarding
both players one half point and appropriate rating changes.

A DR message may be sent in reply to any other message during an `ACTIVE` game.

A DR message **MUST** be formatted as:

    IDENTIFIER:DR

A DR message **MUST** either be acknowledged with a DA message or a DD message.

### DA (Draw Acknowledge)
A Draw Acknowledge message **MUST** be sent only in reply to a corresponding DR
message.

A DA message must be formatted as:

    IDENTIFIER:DA

After a DA message has been sent, an identical message **MUST** be sent by the
client who requested the draw, serving as a final confirmation.

After this confirmation is sent, the game is considered `INACTIVE`. Both players are considered to have drawn the game.

### DD (Draw Decline)
A Draw Decline message **MUST** be sent only in reply to a corresponding DR message.

A DD message **MUST** be formatted as:

    IDENTIFIER:DR

A client receiving a DD message **MUST** NOT send further draw messages until after another MV has been sent.

### CC (Concede)
A Concede message indicates that the sender of the message wishes to terminate the game immediately. The game is considered `INACTIVE` immediately after this message is sent, with no further requirements on the part of the transmitting party.

A CC message must be formatted as:

    IDENTIFIER:CC

The receiving client **SHOULD** verify the concede message, but the sending client is under no obligation to respond.

The sender of the CC message **MUST** be considered the loser of the game for scoring and ratings adjustment purposes.

The physical equivalent would be getting up and walking away from the board.

### ER (Error)
An Error message indicates that the last received transmission could not be
validly processed according to this protocol. It **SHOULD** be sent in response
to, but not limited to, the following conditions:

* A game move where no game has been set up
* Any message with a missing or invalid identifier
* An invalid (illegal) game move

An ER message **MUST** include a human-readable description of the error in the
designated comments section.

An ER message **MUST** be formatted as follows:

    IDENTIFIER:ER:ERTYPE:MESSAGE

In case the ER is in response to a missing or invalid identifier, the identifier used in the reply **SHOULD** be the same as the message it is in response to, even if invalid.

If an ER message is recieved in response to another ER message, another ER message **MUST** NOT be sent.

#### Error Types

A SMCGP-compliant client **MUST** be aware of the following classes of error.

##### MVIL (Move Illegal)

A MVIL error indicates that the previously received MV message is invalid according to the laws of chess.

The client receiving this error **SHOULD** send a RT action message in case the invalidity was caused by transmission problems.

The client who sent the invalid move **MUST** send a valid move for the game to continue.

##### IDIL (ID Illegal)

A IDIL error indicates that the client does not recognize the ID sent. This error **SHOULD** be used in cases where:

* An ID is invalid according to this proposal
* An ID transmitted conflicts with an ID already in use by the client

##### MSIL (Message Illegal)

A MSIL error indicates that the last message received was not validily formatted according to this proposal

##### GSIL (Game State Illegal)

A GSIL error indicates that a message is being rejected due to the state of the game, e.g. a MV after a CC, or a MV when it is not the transmitting player's turn.


### RT (Retransmit)
A Retransmit message indicates a desire for the recipient to resend its last communication. The receiving client **MUST** repeat its last transmission verbatim, with no changes.

A RT message **MUST** be formatted as:

    IDENTIFIER:RT

### KI (Kibitz)

A Kibitz message is simply player to player communication.

A KI message **MUST** NOT be used to control the state of the game.

A KI message **MUST** be formatted as follows:

    IDENTIFIER:KI:MESSAGE

A compliant client **SHOULD** show the MESSAGE to the recipient.

## Future Expansion

A client **MAY** implement their own action flags or error types, so long as they do not conflict with the basic format, or with any of the action flags laid out in this proposal.

## Example Session

What follows is a very short game using the proposed protocol:


    ABCDE1:CH:W:UNR:Newbie looking for a game:
    ABCDE1:CA:TUVWX1:
    ABCDE1:AA:TUVWX1:
    TUVWX1:MV:1.f3:Good luck:
    TUVWX1:MV:e5:
    TUVWX1:MV:2.g4:
    TUVWX1:MV:Qh4#:Bad move:
    TUVWX1:KI:Ouch, should have seen that coming:

Note that only the player playing White transmits the move number of the game (1. 2. etc.) - the Black player only transmits the move.

<reference anchor='LAW' target="http://www.fide.com/component/handbook/?id=171&amp;view=article">
  <front>
   <title>Laws of Chess</title>
  <author>
    <organization>World Chess Federation (FIDE)</organization>
  </author>
   <date year="2014" month="July"></date>
  </front>
</reference>

{backmatter}
