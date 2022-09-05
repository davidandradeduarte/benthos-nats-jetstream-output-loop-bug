# Reproducing a bug in Benthos when using NATS JetStream output

[**GitHub issue**](TBA)

## Reproduction steps

1. Install [NATS CLI](https://github.com/nats-io/natscli):

   ```sh
   go install github.com/nats-io/natscli/nats@latest
   ```

1. Start a NATS server with Jetstream enabled:

   ```sh
   docker run -d --name nats-js -p 4222:4222 -p 6222:6222 -p 8222:8222 nats -js
   ```

1. Subscribe to foo subject:

   ```sh
   nats context add nats --server localhost:4222 --description "NATS on Docker" --select; nats sub foo
   ```

1. Run the Benthos pipeline:

   ```sh
   go run main.go -c pipeline.yaml
   ```

The first 5 messages are delivered without any error (as expected - the `generate` input is configured to emit 5 messages). After that, it starts looping over the same 5 messages but this time it fails with the following error:

```log
   ERRO Failed to send message to nats_jetstream: nats: timeout  @service=benthos label="" path=root.output
```

Funny enough, the messages are still delivered to the `foo` subject. Our nats cli subscriber keeps receiving the same 5 messages over and over again.

```sh
$ nats sub foo
11:58:54 Subscribing on foo
[#1] Received on "foo" with reply "_INBOX.ObYQYcGbc2B3dowRbIfwq8.ArOPmGy3"
{"id":"61277f5b-37df-4dc6-9dfb-43d30399471e","test":"message"}

[#2] Received on "foo" with reply "_INBOX.ObYQYcGbc2B3dowRbIfwq8.J7TS8coB"
{"id":"87014029-1ed9-41fb-8f0b-95cf1d9f8fbc","test":"message"}

[#3] Received on "foo" with reply "_INBOX.ObYQYcGbc2B3dowRbIfwq8.gkkYf0xJ"
{"id":"7ad76a4d-99ae-4fa0-bea2-18ca63dc3d38","test":"message"}

[#4] Received on "foo" with reply "_INBOX.ObYQYcGbc2B3dowRbIfwq8.L1OQMv46"
{"id":"c076fd48-d11e-4460-8be4-34d9332419ef","test":"message"}

[#5] Received on "foo" with reply "_INBOX.gIbB589BB6a6CZDIgCSsoK.HyUKsQF6"
{"id":"af04dab9-cf91-487b-8208-cddc517dc4c5","test":"message"}

[#6] Received on "foo" with reply "_INBOX.gIbB589BB6a6CZDIgCSsoK.8Ge1PEyy"
{"id":"d0e87429-95da-439e-bc15-099e47060dc7","test":"message"}

[#7] Received on "foo" with reply "_INBOX.gIbB589BB6a6CZDIgCSsoK.EqFFinHa"
{"id":"ffcacb25-8c05-4b08-ac21-e0cf55bf17c4","test":"message"}

[#8] Received on "foo" with reply "_INBOX.gIbB589BB6a6CZDIgCSsoK.m577qYxS"
{"id":"6614890d-aad9-4d9f-a415-c9f679a5fadd","test":"message"}

[#9] Received on "foo" with reply "_INBOX.gIbB589BB6a6CZDIgCSsoK.u1ksVTPv"
{"id":"2aaeea69-2d42-4d1d-ad5f-21ca9e3bc5fd","test":"message"}

[#10] Received on "foo" with reply "_INBOX.gIbB589BB6a6CZDIgCSsoK.ssPzyWxR"
{"id":"af04dab9-cf91-487b-8208-cddc517dc4c5","test":"message"}

[#11] Received on "foo" with reply "_INBOX.gIbB589BB6a6CZDIgCSsoK.51up6lCI"
{"id":"d0e87429-95da-439e-bc15-099e47060dc7","test":"message"}

[#12] Received on "foo" with reply "_INBOX.gIbB589BB6a6CZDIgCSsoK.QjXqC7h8"
{"id":"ffcacb25-8c05-4b08-ac21-e0cf55bf17c4","test":"message"}

[#13] Received on "foo" with reply "_INBOX.gIbB589BB6a6CZDIgCSsoK.GdktbeRB"
{"id":"6614890d-aad9-4d9f-a415-c9f679a5fadd","test":"message"}

[#14] Received on "foo" with reply "_INBOX.gIbB589BB6a6CZDIgCSsoK.oPlKJyp8"
{"id":"2aaeea69-2d42-4d1d-ad5f-21ca9e3bc5fd","test":"message"}

[#15] Received on "foo" with reply "_INBOX.gIbB589BB6a6CZDIgCSsoK.icI05QMg"
{"id":"af04dab9-cf91-487b-8208-cddc517dc4c5","test":"message"}

...
```

This seems to be caused by a bug in the [nats_jetstream](https://www.benthos.dev/docs/components/outputs/nats_jetstream/) output component not acknowledging the messages after they are successfully published.
