---
layout: post
title: "Making a Kafka Pipeline LLM-Ready Without Buying a New Platform"
date: 2026-07-06
description: "The agentic layer doesn't replace your streaming infrastructure — it's just another consumer on a topic you already have. A walk through the pattern using a small Kafka pipeline."
---

The instinct I keep running into when "AI" and "our existing data pipeline" come up in the same sentence is that they must be in tension — that doing anything agentic means a platform rewrite, a new team, or both.

They're usually not in tension at all. Most of the time the LLM layer is just another consumer on a topic you already have.

Here's the pattern, using a small Kafka pipeline I keep around for exactly this kind of thing.

---

## The Shape Almost Everyone Already Has

Strip away the specifics of whatever you're piping — trades, logins, sensor readings, doesn't matter — and most streaming setups reduce to the same shape: a consumer reads from a source topic, does something to the data, and produces to one or more downstream topics.

```python
consumer = create_kafka_consumer()  # reads SOURCE_TOPIC
producer = create_kafka_producer()  # writes to PROCESSED_TOPIC, AGGREGATION_TOPIC

for message in consumer:
    data = message.value

    processed_data = {
        "user_id": data["user_id"],
        "app_version": data["app_version"],
        "device_type": data["device_type"],
        # ...pass-through fields...
        "processed_timestamp": data["timestamp"],
    }
    producer.send(PROCESSED_TOPIC, value=processed_data)

    user_login_count[data["user_id"]] += 1
    producer.send(AGGREGATION_TOPIC, value={
        "user_id": data["user_id"],
        "login_count": user_login_count[data["user_id"]],
    })
```

That's the whole thing. One source topic, one consumer, two downstream topics, some light aggregation logic in between. It's a deliberately minimal example — real pipelines have more brokers, more partitions, more topics — but the shape doesn't change as they scale up. It's consumer, transform, produce, repeat.

The question that matters is: where does an LLM or agent actually go in this picture?

---

## Nowhere Near the Source

The wrong instinct is to put the agent in the middle of the existing consumer — have it read the raw event, decide what to do, and produce the result itself. That couples your working pipeline's correctness to a model's output on every single message, which is both slower and riskier than it needs to be.

The pattern that actually holds up: the agentic layer is its own consumer, reading from a topic that already exists — usually a downstream one, like the aggregation topic here — and producing to a new topic of its own. It never touches the source topic. It never competes with the existing consumer group. If it's wrong, or slow, or down entirely, the original pipeline doesn't notice.

```python
# A second, independent consumer group — reads the pipeline's own output,
# never the raw source topic.
agent_consumer = KafkaConsumer(
    AGGREGATION_TOPIC,
    bootstrap_servers=BOOTSTRAP_SERVERS,
    group_id="agent-layer",  # separate consumer group: no interference
    value_deserializer=lambda x: json.loads(x.decode("utf-8")),
)

for message in agent_consumer:
    aggregation = message.value
    finding = anomaly_agent.evaluate(aggregation)  # read-only reasoning over the data
    if finding.flagged:
        producer.send(AGENT_FINDINGS_TOPIC, value=finding.to_record())
```

Same rule as the multi-agent monitoring post: the agent can read anything downstream of the pipeline, but it doesn't get a tool that writes back upstream, restarts a consumer, or mutates the topics the rest of the business depends on. Its only output is a new topic that other things can choose to read — or ignore.

---

## The Part That's Actually Hard (And Isn't AI)

The README for this pipeline has a list of production considerations that has nothing to do with agents at all: run brokers across at least three availability zones, replicate topics at a factor of three, move from a bare Python consumer to Kafka Connect or Flink once volume actually demands it, partition for parallelism, monitor consumer lag, secure the cluster with TLS and real authentication.

That list is the actual hard part of running this at scale, and it's worth being honest that none of it changes because you bolted an agent onto the side. A fund that already has this infrastructure — and most do — has already paid this cost. The agentic layer doesn't ask them to pay it again. It asks for one more consumer group and one more topic.

Conflating "we need to mature our streaming infrastructure" with "we need a platform team to do AI" is where a lot of the fear comes from, and they're genuinely separate projects with separate timelines.

---

## What This Isn't

This example is intentionally small — a teaching pipeline, not a production system, and I want to be specific about the gap. A real deployment needs schema enforcement at the topic boundary (so the agent layer isn't parsing whatever shape happens to show up), backpressure handling if the agent is slower than the pipeline producing data, and the same audit logging from the monitoring post — every finding the agent layer produces should carry the evidence it was based on, not just the conclusion.

None of that requires a new platform. It requires treating the agent layer like any other consumer that has to earn its place in a system that was working fine before it showed up.

---

## The Principle, Restated Once More

The infrastructure question and the AI question are two different projects wearing the same conversation. If your Kafka or Spark pipeline already has room for another consumer group, it already has room for an agentic layer — the platform team you think you need is usually a rewrite of software that was already going to need maturing regardless of whether AI ever entered the picture.
