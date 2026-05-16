# 9 · Recommendations and ML — The Engagement Engine

A video without distribution is a video nobody watches. The recommendation system is the discovery layer that decides what each user sees, and at YouTube / Netflix / Hotstar scale it's one of the highest-leverage ML systems in software. The canonical reference is the 2016 YouTube two-tower paper (Covington et al.), which still defines the dominant architectural pattern.

## 9.1 The problem

For each user × time, pick which items (from a catalog of millions or billions) to show. Optimize for some downstream objective — historically click-through-rate (CTR), now usually **expected watch time** or a multi-objective composite (watch time + likes + shares - dislikes - reports).

Two big challenges:
- **Scale**: corpus too large for any single model to score every item.
- **Long tail**: cold-start for new items / new users.
- **Feedback loop**: today's recommendations shape tomorrow's training data.

## 9.2 The two-tower architecture

Almost every modern industrial recommender uses some variant of this two-stage pipeline:

```
                    +--------------------------+
                    |   Candidate Generation    |
                    |   millions  -> thousands  |
                    +-----------+--------------+
                                |
                                v
                    +--------------------------+
                    |        Ranking            |
                    |  thousands -> tens        |
                    +-----------+--------------+
                                |
                                v
                          Re-ranking
                  (diversity, freshness, business rules)
                                |
                                v
                          User sees a list
```

### Candidate generation

- Input: user features (watch history, search history, demographics, recent context).
- Output: a few thousand candidate items relevant to this user.
- Frame as **extreme multi-class classification** — softmax over the whole corpus during training (sampled softmax in practice; full softmax is infeasible).
- Architecture: a deep neural network with the user features encoded, projected into a shared embedding space; items are embedded into the same space; nearest-neighbor (e.g., FAISS, ScaNN) finds the top-K.
- High recall, low latency. Many millions of items reduced to thousands in <100ms.

### Ranking

- Input: the K candidates + user + context features (time of day, device, location, recent activity).
- Architecture: deep neural network; final layer predicts the objective (e.g., expected watch time via weighted logistic regression).
- The YouTube paper's twist: weight positive examples (watched videos) by their actual watch time. The model's learned odds then approximate expected watch time. This optimizes engagement rather than just clicks.
- Lower recall, very high precision; ranks the K candidates and returns the top-N.

### Re-ranking

- Apply diversity rules ("don't show three baby-shark videos in a row").
- Apply freshness boosts ("new uploads from creators they follow").
- Apply business rules (sponsored content, premium content surfacing).
- Apply experiments (A/B test slots).

## 9.3 Features that matter

User features used:
- Watch history embeddings (average of watched-item embeddings).
- Search history embeddings.
- Recent context (last 10 watches, time since each).
- Demographics (age band, gender — with privacy constraints).
- Device + network signal.
- "Example age" — a feature that biases toward freshness.

Item features used:
- Title and description text embeddings.
- Thumbnail image embedding.
- Audio embedding (for music / podcast).
- Creator features.
- Topic / category.
- Engagement statistics (CTR, watch-time ratio, lifetime metrics).
- Freshness (upload age).

Context features:
- Time of day, day of week.
- Device type.
- Current page / surface ("homepage" vs. "watch-next").

## 9.4 Embedding stores at scale

Embeddings for millions of items must be:
- Computed offline (batch update daily or hourly).
- Stored in an embedding store accessible to candidate generation (FAISS, ScaNN, Vespa, Pinecone, Milvus).
- Refreshed without breaking running serving.

User embeddings are computed online (per request, as a function of recent user state) — typically a TensorFlow Serving / TorchServe model inference.

Approximate nearest-neighbor (ANN) search returns top-K in low milliseconds even over billions of vectors. ScaNN, HNSW, IVF-PQ are common indexes.

## 9.5 Training data — and the feedback loop

Training data is the *log of what users did*: which videos they watched, for how long, at what surface.

Problems:
- **Positional bias** — users click the top of the list more, not because it's better but because it's at the top.
- **Selection bias** — users only watch what they were shown; the recommender's choices shape the data.
- **Mute / skip signal** — disliked items get less training signal because users skip them quickly.

Mitigations:
- **Position-aware ranking** — feature includes the position; at inference set to a neutral value.
- **Counterfactual logging** — log the propensity score of each shown item; weight training examples inversely.
- **Exploration** — randomly insert items from outside the recommender's top picks; use that data for "uncontaminated" learning.
- **Off-policy evaluation** — estimate the performance of a candidate model before A/B testing it.

## 9.6 Real-time vs. batch

Three loops:

1. **Daily/weekly batch** — re-train embeddings; refresh item index; recompute global popularity priors.
2. **Hourly micro-batch** — fresh items injected into the corpus, fresh creator data refreshed.
3. **Online / real-time** — per-request inference based on freshest user state. Click on a video → next homepage refresh reflects it.

For YouTube/Netflix, all three loops run continuously. The cheap, infrequent thing (batch training) sets the big direction; the expensive, frequent thing (real-time inference + lightweight features) personalizes the moment.

## 9.7 Serving infrastructure

At inference time, a request goes through:

1. **Edge / API gateway** — auth, request routing.
2. **Candidate generation service** — user-embedding inference + ANN search.
3. **Feature service** — fetches item-side and user-side features (cached, hot-path).
4. **Ranking service** — neural network inference (TF-Serving / TorchServe / custom).
5. **Re-ranker** — diversity + business rules.
6. **Response cache** — short TTL (seconds) for identical requests.

P99 latency target: 100–200 ms for a homepage rail.

## 9.8 Cold-start

For new users: rely on context (device, geo, time) + popularity priors. Personalization improves rapidly with the first few interactions.

For new items: use content-only features (text, thumbnail, audio embeddings) until engagement signals accumulate. "Cold-start in days, warm in hours" is the goal.

For new creators: boost first-uploads slightly in re-ranking to give them a chance to earn watch time.

## 9.9 Multi-objective optimization

Modern recommenders don't optimize a single metric. They balance:
- Watch time.
- User satisfaction (surveys, explicit feedback).
- Long-term retention (proxied by 7-day return rate).
- Creator equity (don't always show the same handful of creators).
- Diversity / serendipity.
- Brand safety (no extreme content surfaced).

The ranking model often produces multiple heads (one per objective) and the re-ranker linearly combines them with tunable weights.

YouTube has publicly described shifting away from pure watch-time and toward "satisfied watch time" + "responsible recommendations" — a recognition that engagement is not the same as user benefit.

## 9.10 Content moderation pipeline (related)

Recommendations and moderation share infrastructure:
- Item embeddings → similarity search → flag near-duplicates of known-problem content.
- ML classifiers (hate speech, nudity, copyright via fingerprinting like ContentID) score items at upload.
- Items above thresholds are escalated to human review; deprioritized in re-ranking; or removed.

The moderation pipeline must scale to ingest rate (YouTube: 500+ hours/min uploaded). It's parallel to recommendations.

## 9.11 ContentID / copyright matching

YouTube's ContentID is itself a major ML system:
- Studios submit reference material.
- Every upload is **fingerprinted** (audio and video) and matched against the reference database.
- Match → owner is notified; can monetize, block, or track.
- Trillions of fingerprint comparisons over the lifetime of the system.

Equivalent systems exist at Facebook (Rights Manager), at music services (Audible Magic), etc.

## 9.12 Live recommendations

For live (JioHotstar's domain): recommendations are about which live events to surface, who's about to go live, what trending live streams a user might enjoy. Latency and freshness matter more than long-term personalization.

- A new live event must surface in homepage seconds after going live.
- A creator going live triggers a notification + homepage refresh for subscribers.
- For sports: tournament-aware ranking (knockouts, derbies, milestones).

## 9.13 Recommendation as a CDN signal

Mentioned in [03-vod-streaming-deep-dive.md](03-vod-streaming-deep-dive.md): recommendations actively shape what's hot. The cache placement system can read the recommendation system's outputs and pre-warm popular candidates per region.

This co-design is a high-leverage staff-engineering project: connecting two systems (recommender and CDN) that normally don't talk.

## 9.14 Privacy and recommendations

A staff-level answer to "how do you do personalization while respecting privacy":

- **On-device** computation where possible (federated learning, federated analytics).
- **Differential privacy** for aggregate signals (e.g., DP-trained popularity scores).
- **Data minimization** in training data — only what's needed.
- **Retention limits** — drop user history older than N months.
- **Opt-out controls** — "incognito mode" that doesn't contribute to personalization.

YouTube ships incognito mode and watch-history controls. Netflix exposes "viewing activity" with a delete option.

## 9.15 Worked design — recommend live streams to a user

**Prompt**: "Design a recommendation system for a live-streaming platform. The user opens the app; show me 10 streams to watch."

A staff answer:

- **Corpus**: all currently-live streams (typically thousands to hundreds of thousands).
- **Candidate generation**: ANN over stream embeddings using user embedding. Filter to "currently live". Add boosts for streams the user's followed creators are running.
- **Ranking**: neural net taking user features + stream features + context. Predict expected watch time.
- **Re-ranking**: diversity (don't all-cricket if user watches multiple sports), freshness (just-started streams get a small boost), business rules (sponsored / promoted slots).
- **Latency**: target 150 ms p99 for the rail.
- **Refresh**: every 30 s while the user is on the homepage (so newly-live streams surface).
- **Cold start**: for new users, geo-popularity + trending live; for new streams, content-feature-only embedding + boost for first 10 minutes.
- **Feedback**: log impressions and watch-time; nightly batch refines models.
- **Privacy**: opt-out drops user features; system falls back to popularity-only.

## 9.16 Must-internalize

- Two-tower: candidate generation (recall) + ranking (precision) + re-ranking (rules).
- Train ranking to predict expected watch time, not CTR.
- ANN search (FAISS / ScaNN / HNSW) is the workhorse at candidate generation.
- Address positional bias, selection bias, feedback loops.
- Three loops: batch / micro-batch / real-time.
- Cold-start solved by content-only features + popularity priors.
- Multi-objective optimization is the current frontier.
- Recommendations co-design with CDN cache placement.
- Privacy: on-device, differential privacy, data minimization.

---

## Sources

- [Deep Neural Networks for YouTube Recommendations — Covington et al., RecSys 2016](https://dl.acm.org/doi/10.1145/2959100.2959190)
- [Recommendation systems overview — Google ML](https://developers.google.com/machine-learning/recommendation/overview/types)
- [YouTube Video Recommendation Systems — PyImageSearch](https://pyimagesearch.com/2023/09/25/youtube-video-recommendation-systems/)
- [ScaNN: Scalable Nearest Neighbors](https://github.com/google-research/google-research/tree/master/scann)
- [FAISS](https://github.com/facebookresearch/faiss)
- [Netflix Tech Blog — recommendations](https://netflixtechblog.com/tagged/recommendations)
