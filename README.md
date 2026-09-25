# Bitsocial Algorithms

> **Status:** thesis / project exploration — no provider implementation yet.
> **Date:** 2026-08-13
> **One-liner:** open architecture for user-selectable feed and recommendation providers across
> Bitsocial clients.
> **Architecture decision:** [DECISION.md](DECISION.md) records the accepted boundaries for provider
> authority, personalization, transparency, ads, and protocol scope. It is pre-BSIP and
> non-normative.

Bitsocial Algorithms is deliberately a docs-first repository. Read this thesis for the project
direction, then read [DECISION.md](DECISION.md) for the accepted boundaries and
[research/x-algorithm-2026.md](research/x-algorithm-2026.md) for the first upstream case study.

---

## 1. Thesis

A peer-to-peer social network still needs useful feeds.

Following feeds, discovery feeds, video recommendations, community directories, search results, and
spam-aware ranking all require choices about what to show first. If one company permanently owns
those choices, it can recapture the network even when the posts and profiles remain peer-to-peer.

Bitsocial should therefore treat feed algorithms as **replaceable services**:

- a mainstream client can ship with a strong default;
- a user can change the feed provider without moving their profile node;
- a provider can offer several algorithms and versions;
- a community, company, or anonymous operator can run an alternative;
- a user can run a personal provider or rank locally;
- every client retains a provider-free chronological floor where the available protocol data permits
  it.

The short doctrine is:

> Feed providers discover and rank. They do not own identities, posts, social state, or client
> policy.

Bitsocial Forge can operate the first default provider. The architecture must still make Forge one
provider among many.

---

## 2. Why this is a separate project

Recommendation is not only a feature of the future flagship Twitter/X-style client.

- the flagship app needs Following and For You feeds;
- WeTube can use video and creator recommendations;
- Seedit can add optional personalized or topic feeds without changing community state;
- 5chan can experiment with optional discovery surfaces while keeping directory and chronological
  views;
- search, discovery, moderation, and AI services can contribute labeled signals without becoming
  protocol authorities.

The cross-client architecture therefore does not belong to one app or to one commercial provider.
It also should not be folded into [Bitsocial Graph](https://github.com/bitsocialnet/bitsocial-graph):
the graph supplies observations and candidate inputs, while an algorithm provider makes a policy
choice over those inputs.

---

## 3. System boundaries

```text
signed P2P profiles, communities, posts, follows
                       │
              ┌────────▼────────┐
              │ graph / index   │  observations, search, candidate frontier
              │ providers       │
              └────────┬────────┘
                       │
              ┌────────▼────────┐
client context│ feed provider   │  candidate sourcing, scoring, optional labels
─────────────►│ + algorithm     │
              └────────┬────────┘
                       │ feed skeleton: signed-content references + metadata
              ┌────────▼────────┐
              │ client          │  hydrate, verify, apply local policy, render
              └────────┬────────┘
                       │
              selected P2P node / public RPC
```

| Layer | Responsibility | Authority it must not gain |
| --- | --- | --- |
| Profile or community node | Publishes signed social state and content | No ranking authority |
| Node RPC provider | Operates or reaches a P2P node for the user | No mandatory control over feed choice |
| Graph or index provider | Observes public state and supplies candidate inputs | No protocol truth or canonical reputation |
| Feed provider | Selects and ranks references under a named algorithm | No ownership of profiles or posts |
| Label provider | Supplies optional safety, quality, or context signals | No global deletion authority |
| Client | Verifies content, applies the user's final policy, and renders the feed | Must not present provider policy as protocol consensus |

One operator can bundle several layers. A default client may bundle them into one simple preset. The
configuration and wire boundaries should nevertheless remain separable so changing a feed does not
require exporting an identity, rotating a key, or migrating a hosted node.

---

## 4. User experience

The first-run experience should be simple enough for a mainstream user:

```text
Feed
  Forge For You
  Following — chronological
  Community-curated
  Custom provider
  Local / personal algorithm
```

Advanced settings can expose the provider and algorithm independently:

```text
Node RPC:       Forge RPC
Graph source:   Bitsocial Graph provider A
Feed provider:  Forge Feeds
Algorithm:      for-you / version 3
Labels:         provider defaults + local blocks and mutes
```

A **feed preset** can select a convenient bundle. It must not turn the bundle into one indivisible
account system.

---

## 5. Feed skeleton, not content custody

The canonical provider response should identify signed Bitsocial content rather than invent a new
provider-owned post object. A future interface may resemble:

```json
{
  "provider": "https://feeds.example",
  "algorithm": { "id": "for-you", "version": "3" },
  "asOf": "2026-08-13T18:00:00Z",
  "items": [
    {
      "communityAddress": "example.bso",
      "commentCid": "bafy...",
      "score": 0.82,
      "reasons": ["followed-by-people-you-follow", "topic-match"],
      "labels": [],
      "sponsored": false
    }
  ],
  "cursor": "opaque-provider-cursor"
}
```

This shape is illustrative, not a specification. The stable ideas are:

- return content references and an opaque pagination cursor;
- identify the provider, algorithm, and version;
- distinguish reasons, labels, and sponsorship from signed post data;
- let the client fetch and verify current content through its selected P2P or RPC path.

A provider may optionally cache or hydrate signed content for performance. Its copy is a convenience,
not a new source of truth, and clients should be able to verify the original signatures and content
addresses.

---

## 6. Portable personalization

Modern recommendation models commonly use a viewer's recent action sequence: what they opened,
liked, replied to, skipped, hid, muted, or followed. Centralized platforms accumulate that sequence
as surveillance state tied to the platform account.

Bitsocial should start from the opposite direction: **personalization state belongs with the
client/user**.

Providers and clients can support progressively more revealing modes:

1. **No provider:** chronological feeds derived from known follows, communities, or subscriptions.
2. **Candidate service only:** the provider supplies candidates and the client ranks locally.
3. **Ephemeral personalization:** the client sends selected recent actions or preferences with a
   request; the provider does not need a durable behavioral account.
4. **Opt-in hosted personalization:** the user authorizes provider-side history for convenience,
   with clear reset, deletion, export, and retention controls.

The common interface must not require a provider to hold the user's signing keys or permanent social
identity. Authentication and payment may exist at the service layer without becoming ownership of
the underlying Bitsocial account.

---

## 7. Candidate inputs

A useful provider needs a candidate frontier before it can rank anything. Bitsocial has pieces of
that system, but not a global firehose:

- [Bitsocial Graph](https://github.com/bitsocialnet/bitsocial-graph) is a docs-first plan for public
  social-graph observations, liveness, discovery, and portable snapshots;
- [bitsocial-indexer](https://github.com/bitsocialnet/bitsocial-indexer) is a working crawler and
  search API, but it intentionally starts empty and indexes only configured communities;
- [lists](https://github.com/bitsocialnet/lists) supplies today's curated discovery seeds;
- future signed profiles and follows can provide in-network candidates;
- community nodes and normal P2P fetches remain the source of signed content.

Candidate coverage is provider provenance, not protocol truth. Providers should disclose which
communities, time range, graph snapshot, and source classes they cover.

---

## 8. Ranking is not moderation

A feed provider necessarily chooses what it recommends. That does not make its choice canonical
moderation.

Bitsocial should keep four decisions distinct:

1. **Protocol validity:** is the object correctly signed and valid under shared protocol rules?
2. **Community state:** has the community owner or moderators applied signed community policy?
3. **Provider policy:** does this provider include, demote, label, or exclude it from a particular
   feed?
4. **Client/user policy:** do local blocks, mutes, safety settings, or preferences hide it?

"Not recommended by this provider" must never be presented as "removed from Bitsocial." Direct
addresses, chronological views, other providers, and compatible clients remain separate paths.

---

## 9. Transparency and competition

Switching providers is the structural backstop. It does not eliminate the need for disclosure,
especially because defaults will serve most users.

A provider manifest should eventually disclose:

- operator and service endpoint;
- algorithm IDs, versions, and update policy;
- open-source, source-available, or closed implementation status;
- candidate coverage and freshness;
- personalization inputs and retention;
- ranking objectives and available explanation metadata;
- label, moderation, and safety dependencies;
- advertising and sponsorship behavior;
- payment, authentication, rate-limit, and availability terms.

Open and proprietary providers can compete. The default provider should meet a high disclosure bar,
and clients should clearly label which provider produced each feed.

---

## 10. Sybil resistance is provider policy

Signed activity proves that a key performed an action; it does not prove that the key represents a
unique human. Providers may weigh liveness, account age, interaction diversity, community roles,
moderation history, tips, awards, or other economic signals.

No one signal becomes canonical reputation:

- economic actions cost more to fake but can be wash-traded and can favor wealth;
- activity and follows can be farmed;
- centralized identity checks undermine pseudonymity;
- AI classifiers introduce model and training-data policy.

Providers should identify their ruleset and expose enough provenance for clients to compare results.
Bitsocial protocol consensus should not choose one Sybil model for every social surface.

---

## 11. What belongs in a future BSIP

The models, scoring weights, training data, ads, and commercial service are not protocol rules.

A future Interface or Application BSIP may standardize only what independent clients and providers
need for interoperability:

- provider discovery and capability manifests;
- algorithm identity and versioning;
- request context and consent semantics;
- feed-skeleton references and cursors;
- optional reasons, labels, sponsorship, and provenance;
- authentication, errors, limits, and privacy expectations.

That BSIP should follow working prototypes and at least one credible alternative implementation. It
should not freeze a transformer architecture or turn one provider's ranking objective into protocol
law.

---

## 12. Prior art

- [X For You algorithm, August 13, 2026 release](https://github.com/xai-org/x-algorithm/tree/a389166f6cf5da70a286b568c87695d4dcdce3a1)
  publishes candidate sourcing, scoring, visibility, labels, and a transparency-report path. See the
  [research note](research/x-algorithm-2026.md).
- [Bluesky custom feeds](https://docs.bsky.app/docs/starter-templates/custom-feeds) use a feed
  generator that returns a skeleton of post references and a cursor for later hydration. This is the
  closest deployed interface precedent.
- [Nostr NIP-90 Data Vending Machines](https://github.com/nostr-protocol/nips) explored a paid market
  for arbitrary services, but is currently marked unrecommended in the official NIPs index. That is
  a warning to keep the first Bitsocial interface narrow and feed-specific.

---

## 13. Roadmap

1. Record the cross-client architecture and upstream research.
2. Define a non-normative feed-skeleton experiment.
3. Build a deterministic reference provider before attempting a large ML model.
4. Integrate one profile-based client path with a provider-free chronological fallback.
5. Run the first hosted provider through Forge without coupling it to profile-node custody.
6. Add a local or independently operated provider and test real switching.
7. Use implementation evidence to draft a BSIP for the stable interface only.

## Repository contents

| File | Purpose |
| --- | --- |
| [README.md](README.md) | Thesis, project boundaries, and roadmap |
| [DECISION.md](DECISION.md) | Accepted architecture direction, pre-BSIP and non-normative |
| [research/x-algorithm-2026.md](research/x-algorithm-2026.md) | Dated analysis of the 2026 X algorithm release |
| [integrations/seedit.md](integrations/seedit.md) | Seedit `best` feed via a provider, and how it relates to pkc-js page sorts |

## Contributing

This repository is at the research and architecture stage. Issues and pull requests should bring
concrete prior art, privacy and abuse analysis, or implementation evidence. Do not describe an
unimplemented provider, feed interface, profile format, or marketplace as shipped.

## License

Bitsocial Algorithms is licensed under the [GNU General Public License v3.0 or later](LICENSE).
