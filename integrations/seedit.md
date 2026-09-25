# Seedit Integration Note

> **Status:** integration exploration; nothing described here is implemented in Seedit
> **Date:** 2026-09-25
> **Scope:** how a provider-backed `best` feed sort could fit Seedit, and how it relates to
> configurable page sorts in pkc-js. Non-normative; follows [DECISION.md](../DECISION.md).
> **Checked against:** Seedit
> [`a243333a`](https://github.com/bitsocialnet/seedit/tree/a243333a46a0c31094f1192e7a3110eb233957df)
> and [pkc-js#346](https://github.com/pkcprotocol/pkc-js/pull/346) (open on this date)

## Summary

- Reddit's front-page `best` is personalized and ranks across subreddits. In Seedit that belongs to
  a feed provider, not to pkc-js page sorts.
- pkc-js page sorts ([pkc-js#73](https://github.com/pkcprotocol/pkc-js/issues/73)) cover ordering
  inside one community, identical for every visitor.
- The feed provider is its own client setting, independent of the PKC RPC. Forge can operate both,
  but changing one must not change the other.
- With no provider configured, Seedit's current sorts keep working and `best` falls back to `hot`.

## Seedit today

- Feed sorts are `hot`, `new`, `active`, and `top`
  ([`sort-types.ts`](https://github.com/bitsocialnet/seedit/blob/a243333a46a0c31094f1192e7a3110eb233957df/src/constants/sort-types.ts#L1)).
  An unknown sort in the URL routes to `/not-found`
  ([`home.tsx`](https://github.com/bitsocialnet/seedit/blob/a243333a46a0c31094f1192e7a3110eb233957df/src/views/home/home.tsx#L69)).
- `best` exists only as a reply sort on the post page: a Wilson score lower bound computed in the
  client
  ([`sortRepliesByBest`](https://github.com/bitsocialnet/seedit/blob/a243333a46a0c31094f1192e7a3110eb233957df/src/lib/utils/post-utils.ts#L41)).
  It is not personalized.
- Feed views are Home (the account's subscriptions), `/s/all`, `/s/mod`, `/s/:community`, and
  `/domain/:domain`. Multi-community views fetch each community's published pages through
  bitsocial-react-hooks `useFeed` and merge them in the client
  ([`use-feed-with-compatible-sort.ts`](https://github.com/bitsocialnet/seedit/blob/a243333a46a0c31094f1192e7a3110eb233957df/src/hooks/use-feed-with-compatible-sort.ts)).
  The same hook already handles communities that do not publish the requested sort.
- The RPC choice lives in the account's PKC options (Settings → Advanced). Subscriptions, votes, and
  blocks are held in the local account.

## Two ordering layers

| | pkc-js page sorts (#73) | Feed provider |
| --- | --- | --- |
| Computed by | The community node; a client can re-sort a page with the same sort file | A provider service, or the client over provider candidates |
| Scope | One community's posts or replies | Candidates across many communities |
| Per-viewer input | None; every visitor gets the same pages | Optional, under the DECISION.md privacy modes |
| Execution | Synchronous `score(comment)` per comment; `null` excludes it | Provider-defined; may be asynchronous or model-based |
| Output | Signed pages published by the community | Feed skeleton: content references plus provider metadata |
| Seedit use | Community-defined sorts on `/s/:community` | `best` on multi-community views |

Because a page-sort `score` is synchronous, sees one comment, and produces one order for everyone, it
cannot run per-viewer model inference. A community can still use it for a non-personalized quality
sort. Personalized and cross-community ranking needs the provider layer.

The two layers compose. A provider ranks references; Seedit then fetches the signed content through
its configured RPC or P2P path, as it does for any feed.

## Proposed integration

Nothing below is implemented.

### Settings

- Add a **Feed provider** setting separate from the PKC options: `None`, the shipped default, or a
  custom endpoint. `None` must always be available.
- Changing the provider does not touch the account, subscriptions, votes, or RPC.
- Whether the first-run default is Forge or `None` is a product decision; see Open questions.

### Routing

- Add `best` to the feed sorts only on multi-community views: Home and `/s/all` first, `/s/mod` and
  `/domain/:domain` later if useful. On `/s/:community`, the community's own page sorts apply.
- When `best` is requested and no provider is configured or the provider fails, show the `hot` feed
  for the same communities with a visible notice. Do not route to `/not-found`, so shared `best` links
  keep working.
- The `top` time filter does not apply to `best`.

### Request context

- `/s/all`: no viewer context is needed. The provider ranks its own candidate set.
- Home: the provider needs to know which communities to rank. Send the subscription list with each
  request as ephemeral context (DECISION.md privacy mode 3), or ask the provider for broader
  candidates and filter to subscriptions in the client (mode 2). Neither requires a provider-held
  history.
- Hosted history (mode 4) is opt-in only, with reset, deletion, and export.

### Rendering

- The provider returns references (`communityAddress`, `commentCid`) plus a cursor. Seedit loads each
  comment and its update through its existing hooks and renders it like any other post.
- Items that fail to load or verify are dropped. Local blocks and content filters apply after the
  provider's ranking.
- The feed shows the provider name and algorithm ID and version. Per-item reasons are shown when the
  provider supplies them.

### Sponsored items

- Sponsored items arrive as explicitly flagged skeleton items and render with a visible label. They
  are never written into posts or community records (DECISION.md, "Ads and blending").
- Targeting should start contextual, by community, directory, or view, rather than personalized.
  Personalized ads require the provider to keep a behavioral profile, which conflicts with the default
  privacy modes and increases switching cost. In the EU, profiling-based ads also need consent, and
  the Digital Services Act prohibits them for minors (Art. 28(2)) and when based on sensitive data
  (Art. 26(3)).
- Removing sponsored items remains possible in forks and with other providers. A provider can only
  control access to its own service.

## Implementation order

1. Build a deterministic reference provider (DECISION.md step 2) that serves `best` for `/s/all`
   using votes, recency, and vote velocity over an explicitly configured candidate set, such as the
   default community lists or an indexer.
2. In Seedit, add the provider setting and `best` on `/s/all`, with the `hot` fallback.
3. Add Home `best` using subscriptions as ephemeral context.
4. Render sponsored items and provider identity.
5. Run a second provider or a local ranker and test switching, failure, and fallback.
6. Consider embeddings or other models only after steps 1 to 5 work.

## Open questions

- Should the provider feed hook live in bitsocial-react-hooks so other clients can reuse it?
- Should the first-run default be a Forge provider, or `None` until a provider publishes a manifest
  that meets the DECISION.md transparency requirements?
- For Home, should Seedit send its subscriptions, or filter broader provider candidates locally? The
  trade-off is privacy against result quality and bandwidth.
- Should `best` ever appear on `/s/:community` through a provider, given that communities can publish
  their own sorts?
- How should a provider cursor interact with Seedit's infinite feed and "newer posts" behavior?
