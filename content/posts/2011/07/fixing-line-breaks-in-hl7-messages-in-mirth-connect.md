+++
title = 'Fixing line breaks in HL7 messages in Mirth Connect'
date = 2011-07-12T15:00:00-05:00
draft = false
tags = ['programming', 'javascript', 'mirth connect', 'hl7']
+++
Anyone who has worked in the world of healthcare integration or with HL7 knows that if you have seen one HL7 message, you've seen <em>one</em> HL7 message.  Now, a common problem with some source systems is that a line break will sneak in the middle of a segment, rendering the whole message invalid.  How many times have you seen this message?

    MSH|^~\&|DDTEK LAB|ELAB-1|DDTEK OE|BLDG14|200502150930||ORU^R01^ORU_R01|CTRL-9876|P|2.4
    PID|||010-11-1111||Estherhaus^Eva^E^^^^L|Smith|19720520|F|||256 Sherwood Forest Dr.^^Baton Rouge^LA^70809||(225)334-5232|(225)752-1213||||AC010111111||76-B4335^LA^20070520
    OBR|1|948642^DDTEK OE|917363^DDTEK LAB|1554-5^GLUCOSE|||200502150730|||||||||020-22-2222^Levin-Epstein^Anna^^^^MD^^Micro-Managed
    Health Associates|||||||||F|||||||030-33-3333&Honeywell&Carson&&&&MD
    OBX|1|SN|1554-5^GLUCOSE^^^POST 12H CFST:MCNC:PT:SER/PLAS:QN||^175|mg/dl|70_105|H|||F

Notice the "Health Associates" segment? Obviously, this segment invalidates the message.  When this message is sent through a Mirth channel that expects incoming HL7, the message will error out when Mirth tries to parse it.  Here is how we can fix that.

<!--more-->

<h3>The Solution</h3>

To fix this, we're going to use a channel's preprocessor script to append seemingly broken segments to the previous segment.  Since the preprocessor script is invoked <em>before</em> Mirth's HL7 parser, we don't have to worry about Mirth's HL7 parser choking on it.

Instead of explaining the logic in a separate paragraph, I have left inline comments to explain what is happening.  I think, though, that it is fairly easy to follow.

Paste the following code into your channel's preprocessor script.

## The Code

    // initialize a first_iteration flag.
    // we don't want to add a carriage return
    // before the MSH segment
    var first_iteration = true;

    // initialize the new message as an empty string
    var new_message = '';

    // loop over each line in the message
    // by splitting on the hl7 standard carriage return
    for each (var line in message.split(/\r/)) {

      // if it is a valid hl7 segment, we'll
      // append the new line to the new message
      if(line.match(/^[a-zA-Z0-9]{3}\|/)) {
        // see the first comment
        if( ! first_iteration){
          new_message += '\r';
        }
        new_message += line;
      } else {
        // if it isn't a valid hl7 segment,
        // we'll append it to the end of the
        // previous segment instead
        new_message += ' ' + line;
      }

      // it will never be the first_iteration again
      first_iteration = false;
    }

    //send the fixed message to the channel
    message = new_message;

    return message;
