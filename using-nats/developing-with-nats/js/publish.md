# Publishing to streams

{% tabs %}
{% tab title="Go" %}
```go
func ExampleJetStream() {
	nc, err := nats.Connect("localhost")
	if err != nil {
		log.Fatal(err)
	}

	// Use the JetStream context to produce and consumer messages
	// that have been persisted.
	js, err := nc.JetStream(nats.PublishAsyncMaxPending(256))
	if err != nil {
		log.Fatal(err)
	}

	js.AddStream(&nats.StreamConfig{
		Name:     "FOO",
		Subjects: []string{"foo"},
	})

	js.Publish("foo", []byte("Hello JS!"))

	// Publish messages asynchronously.
	for i := 0; i < 500; i++ {
		js.PublishAsync("foo", []byte("Hello JS Async!"))
	}
	select {
	case <-js.PublishAsyncComplete():
	case <-time.After(5 * time.Second):
		fmt.Println("Did not resolve in time")
	}
}

```
{% endtab %}
{% tab title="Java" %}
```java
package io.nats.examples.jetstream;

import io.nats.client.Connection;
import io.nats.client.JetStream;
import io.nats.client.Message;
import io.nats.client.Nats;
import io.nats.client.api.PublishAck;
import io.nats.client.impl.NatsMessage;
import io.nats.examples.ExampleArgs;
import io.nats.examples.ExampleUtils;

import java.nio.charset.StandardCharsets;

/**
 * This example will demonstrate JetStream publishing.
 */
public class NatsJsPub {
    static final String usageString =
            "\nUsage: java -cp <classpath> NatsJsPub [-s server] [-strm stream] [-sub subject] [-mcnt msgCount] [-m messageWords+] [-r headerKey:headerValue]*"
                    + "\n\nDefault Values:"
                    + "\n   [-strm] example-stream"
                    + "\n   [-sub]  example-subject"
                    + "\n   [-mcnt] 10"
                    + "\n   [-m]    hello"
                    + "\n\nRun Notes:"
                    + "\n   - msg_count < 1 is the same as 1"
                    + "\n   - headers are optional"
                    + "\n\nUse tls:// or opentls:// to require tls, via the Default SSLContext\n"
                    + "\nSet the environment variable NATS_NKEY to use challenge response authentication by setting a file containing your private key.\n"
                    + "\nSet the environment variable NATS_CREDS to use JWT/NKey authentication by setting a file containing your user creds.\n"
                    + "\nUse the URL in the -s server parameter for user/pass/token authentication.\n";

    public static void main(String[] args) {
        ExampleArgs exArgs = ExampleArgs.builder("Publish", args, usageString)
                .defaultStream("example-stream")
                .defaultSubject("example-subject")
                .defaultMessage("hello")
                .defaultMsgCount(10)
                .build();

        String hdrNote = exArgs.hasHeaders() ? ", with " + exArgs.headers.size() + " header(s)" : "";
        System.out.printf("\nPublishing to %s%s. Server is %s\n\n", exArgs.subject, hdrNote, exArgs.server);

        try (Connection nc = Nats.connect(ExampleUtils.createExampleOptions(exArgs.server))) {

            // Create a JetStream context.  This hangs off the original connection
            // allowing us to produce data to streams and consume data from
            // JetStream consumers.
            JetStream js = nc.jetStream();

            // Create the stream
            NatsJsUtils.createStreamOrUpdateSubjects(nc, exArgs.stream, exArgs.subject);

            int stop = exArgs.msgCount < 2 ? 2 : exArgs.msgCount + 1;
            for (int x = 1; x < stop; x++) {
                // make unique message data if you want more than 1 message
                String data = exArgs.msgCount < 2 ? exArgs.message : exArgs.message + "-" + x;

                // create a typical NATS message
                Message msg = NatsMessage.builder()
                        .subject(exArgs.subject)
                        .headers(exArgs.headers)
                        .data(data, StandardCharsets.UTF_8)
                        .build();

                // Publish a message and print the results of the publish acknowledgement.
                // We'll use the defaults for this simple example, but there are options
                // to constrain publishing to certain streams, expect sequence numbers and
                // more. See the NatsJsPubWithOptionsUseCases.java example for details.
                // An exception will be thrown if there is a failure.
                PublishAck pa = js.publish(msg);
                System.out.printf("Published message %s on subject %s, stream %s, seqno %d.\n",
                        data, exArgs.subject, pa.getStream(), pa.getSeqno());
            }
        }
        catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
{% endtab %}
{% tab title="JavaScript" %}
```javascript
import { connect, Empty } from "../../src/mod.ts";

const nc = await connect();

const jsm = await nc.jetstreamManager();
await jsm.streams.add({ name: "B", subjects: ["b.a"] });

const js = await nc.jetstream();
// the jetstream client provides a publish that returns
// a confirmation that the message was received and stored
// by the server. You can associate various expectations
// when publishing a message to prevent duplicates.
// If the expectations are not met, the message is rejected.
let pa = await js.publish("b.a", Empty, {
  msgID: "a",
  expect: { streamName: "B" },
});
console.log(`${pa.stream}[${pa.seq}]: duplicate? ${pa.duplicate}`);

pa = await js.publish("b.a", Empty, {
  msgID: "a",
  expect: { lastSequence: 1 },
});
console.log(`${pa.stream}[${pa.seq}]: duplicate? ${pa.duplicate}`);

await jsm.streams.delete("B");
await nc.drain();
```
{% endtab %}
{% tab title="Python" %}
```python
import asyncio

import nats
from nats.errors import TimeoutError


async def main():
    nc = await nats.connect("localhost")

    # Create JetStream context.
    js = nc.jetstream()

    # Persist messages on 'foo's subject.
    await js.add_stream(name="sample-stream", subjects=["foo"])

    for i in range(0, 10):
        ack = await js.publish("foo", f"hello world: {i}".encode())
        print(ack)

    await nc.close()

if __name__ == '__main__':
    asyncio.run(main())
```
{% endtab %}
{% tab title="C" %}
```C
#include "examples.h"

static const char *usage = ""\
"-stream        stream name (default is 'foo')\n" \
"-txt           text to send (default is 'hello')\n" \
"-count         number of messages to send\n" \
"-sync          publish synchronously (default is async)\n";

static void
_jsPubErr(jsCtx *js, jsPubAckErr *pae, void *closure)
{
    int *errors = (int*) closure;

    printf("Error: %u - Code: %u - Text: %s\n", pae->Err, pae->ErrCode, pae->ErrText);
    printf("Original message: %.*s\n", natsMsg_GetDataLength(pae->Msg), natsMsg_GetData(pae->Msg));

    *errors = (*errors + 1);

    // If we wanted to resend the original message, we would do something like that:
    //
    // js_PublishMsgAsync(js, &(pae->Msg), NULL);
    //
    // Note that we use `&(pae->Msg)` so that the library set it to NULL if it takes
    // ownership, and the library will not destroy the message when this callback returns.

    // No need to destroy anything, everything is handled by the library.
}

int main(int argc, char **argv)
{
    natsConnection      *conn  = NULL;
    natsStatistics      *stats = NULL;
    natsOptions         *opts  = NULL;
    jsCtx               *js    = NULL;
    jsOptions           jsOpts;
    jsErrCode           jerr   = 0;
    natsStatus          s;
    int                 dataLen=0;
    volatile int        errors = 0;
    bool                delStream = false;

    opts = parseArgs(argc, argv, usage);
    dataLen = (int) strlen(payload);

    s = natsConnection_Connect(&conn, opts);

    if (s == NATS_OK)
        s = jsOptions_Init(&jsOpts);

    if (s == NATS_OK)
    {
        if (async)
        {
            jsOpts.PublishAsync.ErrHandler           = _jsPubErr;
            jsOpts.PublishAsync.ErrHandlerClosure    = (void*) &errors;
        }
        s = natsConnection_JetStream(&js, conn, &jsOpts);
    }

    if (s == NATS_OK)
    {
        jsStreamInfo    *si = NULL;

        // First check if the stream already exists.
        s = js_GetStreamInfo(&si, js, stream, NULL, &jerr);
        if (s == NATS_NOT_FOUND)
        {
            jsStreamConfig  cfg;

            // Since we are the one creating this stream, we can delete at the end.
            delStream = true;

            // Initialize the configuration structure.
            jsStreamConfig_Init(&cfg);
            cfg.Name = stream;
            // Set the subject
            cfg.Subjects = (const char*[1]){subj};
            cfg.SubjectsLen = 1;
            // Make it a memory stream.
            cfg.Storage = js_MemoryStorage;
            // Add the stream,
            s = js_AddStream(&si, js, &cfg, NULL, &jerr);
        }
        if (s == NATS_OK)
        {
            printf("Stream %s has %" PRIu64 " messages (%" PRIu64 " bytes)\n",
                si->Config->Name, si->State.Msgs, si->State.Bytes);

            // Need to destroy the returned stream object.
            jsStreamInfo_Destroy(si);
        }
    }

    if (s == NATS_OK)
        s = natsStatistics_Create(&stats);

    if (s == NATS_OK)
    {
        printf("\nSending %" PRId64 " messages to subject '%s'\n", total, stream);
        start = nats_Now();
    }

    for (count = 0; (s == NATS_OK) && (count < total); count++)
    {
        if (async)
            s = js_PublishAsync(js, subj, (const void*) payload, dataLen, NULL);
        else
        {
            jsPubAck *pa = NULL;

            s = js_Publish(&pa, js, subj, (const void*) payload, dataLen, NULL, &jerr);
            if (s == NATS_OK)
            {
                if (pa->Duplicate)
                    printf("Got a duplicate message! Sequence=%" PRIu64 "\n", pa->Sequence);

                jsPubAck_Destroy(pa);
            }
        }
    }

    if ((s == NATS_OK) && async)
    {
        jsPubOptions    jsPubOpts;

        jsPubOptions_Init(&jsPubOpts);
        // Let's set it to 30 seconds, if getting "Timeout" errors,
        // this may need to be increased based on the number of messages
        // being sent.
        jsPubOpts.MaxWait = 30000;
        s = js_PublishAsyncComplete(js, &jsPubOpts);
        if (s == NATS_TIMEOUT)
        {
            // Let's get the list of pending messages. We could resend,
            // etc, but for now, just destroy them.
            natsMsgList list;

            js_PublishAsyncGetPendingList(&list, js);
            natsMsgList_Destroy(&list);
        }
    }

    if (s == NATS_OK)
    {
        jsStreamInfo *si = NULL;

        elapsed = nats_Now() - start;
        printStats(STATS_OUT, conn, NULL, stats);
        printPerf("Sent");

        if (errors != 0)
            printf("There were %d asynchronous errors\n", errors);

        // Let's report some stats after the run
        s = js_GetStreamInfo(&si, js, stream, NULL, &jerr);
        if (s == NATS_OK)
        {
            printf("\nStream %s has %" PRIu64 " messages (%" PRIu64 " bytes)\n",
                si->Config->Name, si->State.Msgs, si->State.Bytes);

            jsStreamInfo_Destroy(si);
        }
    }
    if (delStream && (js != NULL))
    {
        printf("\nDeleting stream %s: ", stream);
        s = js_DeleteStream(js, stream, NULL, &jerr);
        if (s == NATS_OK)
            printf("OK!");
        printf("\n");
    }
    if (s != NATS_OK)
    {
        printf("Error: %u - %s - jerr=%u\n", s, natsStatus_GetText(s), jerr);
        nats_PrintLastErrorStack(stderr);
    }

    // Destroy all our objects to avoid report of memory leak
    jsCtx_Destroy(js);
    natsStatistics_Destroy(stats);
    natsConnection_Destroy(conn);
    natsOptions_Destroy(opts);

    // To silence reports of memory still in used with valgrind
    nats_Close();

    return 0;
}
```
{% endtab %}
{% endtabs %}