---
layout: post
title: "Making a Kafka Pipeline LLM-Ready Without Buying a New Platform"
date: 2026-07-06
description: "The agentic layer doesn't replace your streaming infrastructure — it's just another consumer on a topic you already have. A walk through the pattern using a small Kafka pipeline."
---

Whenever "AI" and "our existing data pipeline" come up in the same sentence, people tend to assume they're in conflict, and that doing anything agentic means a platform rewrite, a new team, or both.

Usually they're not in conflict at all. Most of the time the LLM layer is just another consumer on a topic you already have. Here's how that looks, using a small Kafka pipeline I keep around for this kind of thing.

---

## The Shape Almost Everyone Already Has

Whatever you're piping (trades, logins, sensor readings), most streaming setups come down to the same shape: a consumer reads from a source topic, does something to the data, and produces to one or more downstream topics.

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

That's all of it: one source topic, one consumer, two downstream topics, and some light aggregation in between. It's deliberately minimal. Real pipelines have more brokers, partitions and topics, but the shape stays the same as they grow: consume, transform, produce, repeat.

So where does an LLM or agent actually go in this picture?

---

## Nowhere Near the Source

The tempting move is to put the agent inside the existing consumer, so it reads the raw event, decides what to do, and produces the result itself. That ties your working pipeline's correctness to a model's output on every single message, which is slower and riskier than it needs to be.

What works better is making the agentic layer its own consumer. It reads from a topic that already exists, usually a downstream one like the aggregation topic here, and produces to a new topic of its own. It never touches the source topic or competes with the existing consumer group, so if it's wrong, slow, or completely down, the original pipeline doesn't notice.

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

It's the same rule as the multi-agent monitoring post. The agent can read anything downstream of the pipeline, but it doesn't get a tool that writes back upstream, restarts a consumer, or changes the topics the rest of the business depends on. Its only output is a new topic that other things can read or ignore.

---

## The Part That's Actually Hard (And Isn't AI)

The README for this pipeline has a list of production considerations that has nothing to do with agents at all: run brokers across at least three availability zones, replicate topics at a factor of three, move from a bare Python consumer to Kafka Connect or Flink once volume actually demands it, partition for parallelism, monitor consumer lag, secure the cluster with TLS and real authentication.

That list is the hard part of running this at scale, and none of it changes because you bolted an agent onto the side. A fund that already has this infrastructure (and most do) has already paid that cost, and the agentic layer doesn't make them pay it again. It needs one more consumer group and one more topic.

A lot of the fear comes from mixing up "we need to mature our streaming infrastructure" with "we need a platform team to do AI." Those are separate projects with separate timelines.

---

## What This Isn't

This example is intentionally small. It's a teaching pipeline, not a production system, and a real deployment would need more: schema enforcement at the topic boundary (so the agent layer isn't parsing whatever shape happens to show up), backpressure handling if the agent is slower than the pipeline feeding it, and the same audit logging from the monitoring post, so every finding carries the evidence it was based on and not just the conclusion.

None of that needs a new platform. You just treat the agent layer like any other new consumer that has to prove itself in a system that was working fine before it arrived.

---

## The Principle, Restated Once More

The infrastructure question and the AI question are two different projects that keep getting discussed as one. If your Kafka or Spark pipeline has room for another consumer group, it has room for an agentic layer. The platform work people think AI requires is usually maturing that the infrastructure needed anyway, AI or not.
