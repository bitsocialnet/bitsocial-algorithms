# Bitsocial Algorithms Architecture Decision

> **Status:** accepted direction; pre-BSIP and non-normative
> **Decision date:** 2026-08-13
> **Last reviewed:** 2026-08-13
> **Related research:** [X algorithm release, August 2026](research/x-algorithm-2026.md)

## Decision

Bitsocial will treat personalized feeds and recommendation algorithms as optional, replaceable
services above the peer-to-peer protocol.

The default flagship client may use a hosted Forge feed provider, but it will not make that provider
the owner of profiles, follows, posts, or protocol state. The profile/node RPC provider and feed
provider remain logically independent choices even when one operator bundles both.

The short doctrine is:

> Feed providers discover and rank. They do not own identities, posts, social state, or client
> policy.

## Terms

- **Feed provider:** a service or local process that returns an ordered collection of Bitsocial
  content references.
- **Algorithm:** a named and versioned policy used by a provider to source, score, filter, diversify,
  or blend feed candidates.
- **Feed preset:** a client-facing bundle that may select a node RPC, graph source, feed provider,
  algorithm, label source, and ad policy together for convenience.
- **Feed skeleton:** ordered references to signed Bitsocial content plus provider metadata, before or
  independently of hydration.

## Layer boundaries

| Layer | Owns | Does not own |
| --- | --- | --- |
| P2P protocol and nodes | Signed profiles, communities, content, and shared validity rules | Global ranking policy |
| Node RPC | Remote access to node capabilities under a non-custodial service relationship | Mandatory feed selection |
| Graph/index provider | Observations, indexes, coverage, and candidate inputs | Canonical reputation or protocol truth |
| Feed provider | Candidate selection and ordering under a disclosed ruleset | Identities, signing keys, or source content |
| Label provider | Optional classifications and contextual signals | Global deletion authority |
| Client | Verification, final local policy, provider selection, and presentation | Authority to rewrite valid signed state |

An operator may expose several capabilities at one endpoint. Clients and future interfaces should
still identify them separately. Moving from one feed algorithm to another must not require profile
migration or key rotation.

## Provider-free client floor

Once signed profiles and public follows exist, compatible clients should be able to construct a
chronological Following feed from public state without a recommendation provider. Clients can also
continue to expose direct community, subscription, directory, and address-based views.

This is a client capability derived from protocol data, not a globally agreed feed encoded in
consensus. Availability and discovery remain real constraints: a client cannot order content it has
not found or fetched.

A provider outage or removal must therefore degrade the personalized surface, not make the user's
profile or direct content inaccessible.

## Canonical output: references plus metadata

The common provider output should be a feed skeleton containing stable references to signed
Bitsocial objects. The exact schema remains open, but interoperable implementations will likely
need:

- community/profile context plus content CID or the eventual stable post reference;
- an opaque provider-controlled pagination cursor;
- provider, algorithm ID, algorithm version, and response time;
- optional score, source, reason, label, and explanation fields;
- explicit sponsorship or advertising metadata;
- coverage, snapshot, or ruleset provenance where useful.

Hydrated content may be returned as an optimization. Clients must be able to validate it against the
signed/content-addressed object and must not treat a provider's mutable representation as canonical.

Providers choose what enters their own feeds. They do not gain an API for changing the source post,
the author's identity, or community state.

## Personalization state

The default architecture keeps the user's action sequence, preferences, impressions, blocks, and
mutes client-held unless the user opts into provider storage.

Supported privacy modes should include:

1. no provider and no personalization upload;
2. server-side candidate generation with client-side ranking;
3. ephemeral client-supplied context used for one request or short session;
4. explicit hosted personalization with documented retention, reset, deletion, and export.

The shared provider interface must not require:

- a user signing key;
- a provider-owned copy of the Bitsocial identity;
- permanent raw engagement history;
- public publication of blocks, mutes, or private recommendation feedback.

A commercial provider may require service authentication, payment, rate-limit credentials, or abuse
controls. Those credentials must remain service-layer access, not ownership of the user's protocol
identity. Losing them cannot invalidate signed P2P state.

## Ranking, labels, and moderation

The architecture separates:

1. protocol validity;
2. signed community moderation state;
3. provider feed policy;
4. optional third-party labels;
5. local client and user policy.

A feed provider can omit or demote a candidate in its own result. It cannot declare the underlying
post nonexistent or invalid for other providers and direct clients.

Local blocks and mutes should be enforceable by the client. A user may choose to disclose them to a
provider for candidate filtering, but provider disclosure is not required for the local policy to
work.

Provider-generated labels must identify their source and ruleset. A client must not render them as
community moderation or protocol consensus unless they actually derive from signed community state
or shared validity rules.

## Ads and blending

Providers may monetize feeds through clearly identified sponsored items or service subscriptions.

Ad metadata remains service metadata. It must not be written into signed posts, community records,
profile state, moderation logs, or other canonical protocol objects merely to make a client render
it. Clients decide how provider sponsorship metadata is displayed, and an ad-free or provider-free
feed remains possible.

## Transparency requirements

Provider competition is necessary but not sufficient because defaults can dominate user behavior.
A discoverable provider manifest should eventually state:

- operator identity and endpoint;
- available algorithms and versions;
- source-code availability and license, if published;
- candidate sources, coverage, freshness, and known exclusions;
- input categories, retention, and personalization modes;
- ranking objectives and explanation support;
- label/moderation dependencies;
- advertising behavior;
- payment, authentication, quotas, and availability policy;
- export, reset, deletion, and exit behavior for provider-held user state.

Per-item reasons and scores can be optional capabilities. Provider and algorithm identity is not
optional: the client must be able to say which service produced the feed.

## Sybil and reputation policy

Providers may use public liveness, follows, interactions, community roles, moderation outcomes,
tips, awards, or on-chain signals. These are inputs to a provider policy, not shared reputation
truth.

Economic signals can increase attack cost but do not prove personhood or quality. They can be
wash-traded, purchased, or biased toward wealth. Social signals can be automated or farmed. Identity
systems can damage pseudonymity. Providers should disclose signal classes and rulesets rather than
presenting a weighted score as protocol fact.

The chain may contribute optional economic observations. Ordinary recommendation feedback and raw
social graphs do not belong in consensus.

## Provider discovery

The initial client may ship with a curated default. Later provider discovery can use a public list,
signed registry, community curation, or another replaceable directory.

Discovery must not imply endorsement or objective quality. A provider list can describe endpoints
and capabilities; clients and users still decide what to trust. A provider can also be configured
directly without appearing in a default directory.

## BSIP boundary

This decision is intentionally pre-BSIP.

A future BSIP can standardize the minimum interface required for independent clients and providers:

- capability discovery;
- provider and algorithm identity;
- feed request and context semantics;
- content references and cursors;
- optional reasons, labels, sponsorship, and provenance;
- authentication, error, limit, privacy, and version negotiation behavior.

It must not standardize:

- one model family;
- one training dataset;
- one scoring objective or weight set;
- one provider registry as canonical;
- Forge commercial policy;
- global recommendation or reputation consensus.

Drafting should follow a working reference provider, a real client integration, and at least one
credible alternative or local implementation. The goal is demonstrated interoperability, not an
API imagined in advance.

## Implementation sequence

1. Publish the thesis, this decision, and dated prior-art research.
2. Prototype a deterministic provider over explicitly configured candidate sources.
3. Return feed skeletons with algorithm/version, cursor, reasons, and sponsorship fields.
4. Integrate a profile-based client with a chronological fallback and independently selectable node
   RPC.
5. Add client-held impressions and optional ephemeral personalization context.
6. Operate a first Forge provider without making its endpoint protocol-canonical.
7. Run a second provider or local implementation and test switching, failure, and state reset.
8. Draft the stable interface as a BSIP.

Large-model training is not the first milestone. The first proof is exit: the same client and
identity can move between a deterministic hosted provider, chronological mode, and another provider
without losing social state.

## Rejected alternatives

### Put one ranking algorithm in the protocol

Rejected. Ranking objectives are subjective, change frequently, and vary by product. Consensus
would turn one product policy into network law.

### Make the node RPC provider the algorithm provider

Rejected as a required coupling. One operator may bundle both, but changing feed preferences should
not move the node that holds or serves the user's profile state.

### Require provider-side engagement history

Rejected. It recreates platform surveillance and makes personalization the switching cost. Hosted
history can exist only as an opt-in service mode with exit controls.

### Let providers return only proprietary content objects

Rejected. The stable result should point to signed Bitsocial objects. Optional hydration must remain
verifiable and replaceable.

### Build a generic arbitrary-compute marketplace first

Rejected. Feed interoperability is already a large problem. A narrow feed interface is easier to
reason about, secure, implement, and revise than a universal remote-job protocol.

## Open questions

- What is the smallest stable content reference across profiles, crossposts, and communities?
- Which personalization inputs can be standardized without encouraging oversharing?
- Should portable client-held state be raw events, aggregates, embeddings, or several negotiated
  forms?
- How should provider manifests and algorithm versions be signed and discovered?
- Which explanation fields are useful without making providers expose attack-sensitive internals?
- How should paid providers authenticate while preserving pseudonymous use and easy exit?
- Where should candidate coverage and snapshot provenance live?
- Which labels are safe to share across providers, and which must remain local?
