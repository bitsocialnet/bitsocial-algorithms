# X For You Algorithm Release — Bitsocial Research Notes

> **Status:** research snapshot, not an implementation plan
> **Upstream reviewed:** [`xai-org/x-algorithm` at `a389166`](https://github.com/xai-org/x-algorithm/tree/a389166f6cf5da70a286b568c87695d4dcdce3a1)
> **Upstream release date:** 2026-08-13
> **Reviewed:** 2026-08-13
> **Upstream license:** Apache-2.0

## Executive conclusion

The August 2026 X release is useful to Bitsocial as:

1. a concrete decomposition of a production For You pipeline;
2. source material for ranking, retrieval, diversity, visibility, and transparency experiments;
3. evidence that ranking, visibility policy, and feed blending should be modeled as separate stages;
4. a warning about the infrastructure and surveillance state a centralized recommender accumulates.

It is not a drop-in Bitsocial provider. X assumes one platform owns the global post corpus, follow
graph, user action history, labels, storage systems, event streams, experiments, and serving
infrastructure. Bitsocial must replace those assumptions with signed P2P state, disclosed provider
coverage, client-held personalization, and provider exit.

The first Bitsocial experiment should be a deterministic feed-skeleton provider, not an immediate
port of Phoenix.

## What the release contains

The upstream README describes two connected paths.

### Request path

1. **Query hydration** loads the viewer's action sequence, follows, blocks, mutes, topics, seen posts,
   and other context.
2. **Candidate sourcing** queries in-network and out-of-network sources in parallel.
3. **Candidate hydration** loads post, author, media, language, engagement, and label data.
4. **Pre-scoring filters** remove duplicates, old posts, blocked/muted authors, previously seen posts,
   and other ineligible candidates.
5. **Scoring** predicts multiple possible viewer actions, combines them with explicit weights, then
   applies diversity and network-position adjustments.
6. **Selection** keeps the highest-scoring candidates.
7. **Post-selection filtering** applies visibility decisions and conversation deduplication.
8. **Blending** inserts ads, account recommendations, prompts, and other non-post modules.
9. **Side effects** record impressions and other response events.

### Labeling path

Content and account models produce scores and labels. Rule systems apply or aggregate labels.
Visibility filtering then uses those labels plus viewer, author, country, and account state to return
allow, interstitial, or drop.

The release includes, among other components:

| X component | Role |
| --- | --- |
| `home-mixer/` | Orchestrates candidate sourcing, hydration, filtering, scoring, selection, and blending |
| `candidate-pipeline/` | Composable source/hydrator/filter/scorer/selector framework |
| `thunder/` | Recent in-network posts from followed accounts |
| `phoenix/` | Two-stage ML retrieval and engagement prediction/ranking |
| `simclusters/` | Cluster-based out-of-network candidate retrieval |
| `vm-ranker/` | Diversity-aware reranking |
| `visibility-filtering/` | Per-viewer allow, interstitial, or drop policy |
| `grox/`, `agatha/`, `bdsm/`, `user-cred-v2/` | Content/account understanding and reputation-related signals |
| `under-the-hood/` | Aggregates visibility-impacting labels for user-facing transparency |

The repository publishes scoring configuration and Phoenix training/serving code with synthetic data
generation. It deliberately omits some prompts and rules to reduce gaming, and some services retain
internal infrastructure imports or lack deployment manifests. The release is inspectable and partly
runnable, not a self-contained clone of X production.

## Mapping to Bitsocial

| X stage | Bitsocial analogue | Missing or changed assumption |
| --- | --- | --- |
| User action sequence | Client-held impressions, interactions, and preferences | Must not require permanent provider surveillance state |
| Followed-account source | Future signed profiles/follows plus fetched profile posts | Profile/follow BSIP and working profile nodes do not exist yet |
| Global recent-post store | Configured crawlers, indexers, graph providers, community sources | Bitsocial has no canonical global firehose |
| Phoenix/SimClusters retrieval | Competing graph/index/retrieval providers | Coverage must be disclosed and replaceable |
| Candidate hydration | Fetch signed objects through P2P or the selected node RPC; optional provider cache | Hydrated provider data is not canonical |
| Phoenix scoring | Provider-specific deterministic or ML algorithm | Model and weights remain service policy |
| Visibility filtering | Protocol validity + signed community moderation + provider labels + local user policy | No global platform label authority |
| Ads/prompt blending | Provider metadata rendered under client policy | Ads must not become signed protocol content |
| Impression side effects | Client-held history or explicitly authorized feedback | Portable/resettable rather than mandatory provider account history |
| Under the Hood | Provider manifest, per-item reasons, label/source disclosure | Competition and switching add an additional accountability mechanism |

### The candidate frontier is the hard dependency

X can query a global recent-post store and platform-wide indexes because it operates the whole
platform. Bitsocial cannot assume that input.

[bitsocial-indexer](https://github.com/bitsocialnet/bitsocial-indexer) is a useful crawler and API,
but intentionally knows zero communities until configured. [Bitsocial Graph](https://github.com/bitsocialnet/bitsocial-graph)
is intended to add recursive discovery, liveness, profile/follow observations, and portable
snapshots, but has no implementation yet. Curated [lists](https://github.com/bitsocialnet/lists),
community links, future profile publications, and operator-specific coverage remain necessary.

The first provider must state what it can see. A ranking score without candidate-coverage provenance
can look objective while silently excluding most of the network.

## Ideas worth carrying forward

### 1. A composable pipeline

Candidate sources, hydration, filters, scorers, selection, reranking, and side effects should be
separate interfaces inside a reference provider. Providers can then swap a deterministic scorer for
an ML model without changing the client-facing feed skeleton.

This is an implementation pattern, not a wire requirement. A small provider should not need X's
internal framework to interoperate.

### 2. Multi-action prediction

X predicts several positive and negative actions rather than one abstract relevance number, then
combines them with explicit weights. This makes the provider's objective easier to inspect: a system
optimizing replies is a different product from one optimizing dwell time or minimizing blocks.

A future Bitsocial model can use the same conceptual separation while letting providers publish
their objective categories and versioned weights when they choose.

### 3. Candidate isolation

Phoenix prevents one candidate from attending to other candidates during scoring. This makes a
candidate's score independent of the batch, improving consistency and cacheability. Diversity is
then handled as a later reranking step.

That separation is useful for reproducible provider results even if Bitsocial uses a different model.

### 4. Ranking and visibility are different questions

X ranks candidates and applies visibility decisions through a separate system. Bitsocial needs an
even sharper distinction:

- ranking is a provider's ordering policy;
- provider labels are optional service metadata;
- community moderation is signed community state;
- final blocks, mutes, and safety settings belong with the user/client;
- protocol validity is shared across compatible implementations.

A provider can refuse to recommend an object. It does not make that object disappear from other
clients, direct links, or providers.

### 5. User-visible label transparency

X pairs source publication with an Under the Hood report showing visibility-impacting labels. A
Bitsocial provider should similarly identify labels and reasons that affected a response.

Bitsocial can add a stronger structural guarantee: a user can select another provider or no provider.
Exit complements transparency; it does not replace it.

### 6. Client-held action sequences

The viewer action sequence is the strongest design input for the Bitsocial model precisely because
Bitsocial does not need to store it the way X does.

The client can keep raw impressions and interactions locally, derive a smaller preference summary,
send selected context ephemerally, or rank provider candidates locally. Hosted history remains an
opt-in convenience with export, deletion, and reset.

Portable personalization is what makes changing an algorithm meaningful instead of cosmetic.

## What not to copy

### Platform-wide surveillance assumptions

Do not make a durable provider account and raw interaction log prerequisites for useful feeds. That
would recreate the largest switching cost of centralized social media.

### One global visibility authority

X's label and filtering systems are controlled by the same platform that owns distribution.
Bitsocial must distinguish protocol validity, signed community policy, provider recommendation
policy, third-party labels, and local user policy.

### Internal infrastructure coupling

Several upstream components assume X-specific storage, telemetry, Kafka, experiment, or service
infrastructure. Reusing concepts or isolated Apache-licensed code is different from adopting the
production topology.

### Engagement as truth

Open networks make likes, follows, and accounts cheap to manufacture. Economic signals raise costs
but can be wash-traded and can privilege wealth. Every provider needs a disclosed Sybil policy;
none should become consensus reputation.

### One feed for every product

Twitter/X's objective fits its own surface. A WeTube provider, community-directory provider,
chronological Following feed, and public-conversation For You feed should not share one mandatory
objective merely because they use the same protocol.

## Licensing and reuse

The reviewed X repository is Apache-2.0. Apache-2.0 code can be incorporated into a GPLv3 project,
subject to the upstream license, attribution, notices, and provenance requirements. This is more
permissive for a future GPL-3.0 reference provider than the older AGPL-licensed
[`twitter/the-algorithm`](https://github.com/twitter/the-algorithm) release.

That does not make wholesale copying the right first step. Before adapting a component:

1. identify the exact files and transitive dependencies;
2. preserve copyright, license, and notice requirements;
3. remove or replace X-internal infrastructure assumptions;
4. verify that the component is actually buildable in the published tree;
5. document modifications and upstream commit provenance;
6. test it against Bitsocial data rather than assuming X behavior transfers.

The [Apache Software Foundation compatibility guidance](https://www.apache.org/licenses/GPL-compatibility.html)
explains the Apache-2.0 to GPLv3 direction. This note is architectural research, not legal advice.

## Other interface prior art

### Bluesky feed generators

[Bluesky custom feeds](https://docs.bsky.app/docs/starter-templates/custom-feeds) are the closest
deployed interface comparison. A declared feed generator returns `getFeedSkeleton` results containing
post references and an opaque cursor; the AppView hydrates the feed.

Bitsocial should reuse the narrow skeleton concept while changing the trust boundary:

- hydrate from signed P2P content or a selected non-custodial RPC;
- avoid making one AppView the mandatory network view;
- let node RPC and feed provider choices remain independent;
- support provider-free chronological modes.

### Nostr NIP-90

Nostr Data Vending Machines explored discovery, requests, results, and payment for arbitrary remote
services. The current official [NIPs index](https://github.com/nostr-protocol/nips) marks NIP-90
unrecommended and advises use-case-specific microstandards.

That is relevant negative evidence. Bitsocial should specify a narrow feed-provider contract after
real prototypes rather than generalizing immediately into an arbitrary AI or compute marketplace.

## Recommended first experiment

Do not begin by training Phoenix.

Build a small deterministic provider that:

1. reads explicitly configured communities and any available profile/follow test data;
2. sources recent in-network and discovery candidates separately;
3. scores recency, known follow relationships, activity, and configurable topic signals;
4. returns content references, an opaque cursor, algorithm/version, source, reasons, and sponsorship
   metadata;
5. accepts optional ephemeral client context without requiring an account history;
6. works beside a client-computed chronological feed;
7. can be replaced by a second endpoint without moving the profile node.

This proves the Bitsocial-specific architecture and produces the evidence needed for a later
interface BSIP. Phoenix retrieval/ranking can then be evaluated as one provider implementation,
using a synthetic or consented dataset and measured against deterministic baselines.

## Sources

- [`xai-org/x-algorithm` at the reviewed release commit](https://github.com/xai-org/x-algorithm/tree/a389166f6cf5da70a286b568c87695d4dcdce3a1)
- [Upstream README](https://github.com/xai-org/x-algorithm/blob/a389166f6cf5da70a286b568c87695d4dcdce3a1/README.md)
- [Phoenix README](https://github.com/xai-org/x-algorithm/blob/a389166f6cf5da70a286b568c87695d4dcdce3a1/phoenix/README.md)
- [Phoenix quickstart](https://github.com/xai-org/x-algorithm/blob/a389166f6cf5da70a286b568c87695d4dcdce3a1/phoenix/QUICKSTART.md)
- [X algorithm Apache-2.0 license](https://github.com/xai-org/x-algorithm/blob/a389166f6cf5da70a286b568c87695d4dcdce3a1/LICENSE)
- [Bluesky custom feed documentation](https://docs.bsky.app/docs/starter-templates/custom-feeds)
- [Nostr NIPs index](https://github.com/nostr-protocol/nips)
- [Apache License 2.0 and GPLv3 compatibility](https://www.apache.org/licenses/GPL-compatibility.html)
- [Bitsocial Graph](https://github.com/bitsocialnet/bitsocial-graph)
- [bitsocial-indexer](https://github.com/bitsocialnet/bitsocial-indexer)
- [Bitsocial lists](https://github.com/bitsocialnet/lists)
- [Bitsocial Phase 3 flagship-app direction](https://github.com/bitsocialnet/bitsocial-web/blob/master/docs/flagship-bitsocial-app.md)
