# Natural-Language Personalized Playlist Service --- Staff-Level System Design

> **Problem:** Design a service on top of a music platform such as Apple
> Music that allows users to generate personalized playlists from
> natural-language prompts such as:
>
> **"I'm feeling sad, give me a playlist."**
>
> The system uses an LLM to understand user intent, retrieves valid
> tracks from the music catalog, personalizes and ranks them, constructs
> a coherent playlist, and optionally saves it to the user's music
> account.

------------------------------------------------------------------------

## 1. Requirements

### Functional Requirements

1.  Accept natural-language playlist prompts.
2.  Interpret:
    -   Mood
    -   Genre
    -   Activity
    -   Era / year range
    -   Language
    -   Artists
    -   Energy
    -   Duration
    -   Exclusions
    -   Playlist progression
3.  Personalize results using user preferences and listening behavior.
4.  Retrieve only tracks that actually exist in the music catalog.
5.  Respect regional/catalog availability.
6.  Rank candidate tracks according to intent and user preferences.
7.  Construct an ordered playlist rather than simply returning
    individually relevant songs.
8.  Support refinements such as:
    -   "Make it more upbeat."
    -   "Less Taylor Swift."
    -   "Add some Hindi songs."
    -   "Give me more songs like track 4."
9.  Save the generated playlist to the user's music account.
10. Learn from user feedback such as play, skip, like, dislike, and
    regeneration.

### Non-Functional Requirements

-   Low playlist-generation latency, ideally a few seconds.
-   High availability.
-   Horizontal scalability.
-   Global operation.
-   Privacy for listening history and profile information.
-   Regional licensing/catalog correctness.
-   LLM cost control.
-   Graceful degradation if the LLM is unavailable.
-   High recommendation quality.
-   Explainable/debuggable recommendation pipeline.
-   Avoid hallucinated or nonexistent tracks.

### Out of Scope

The underlying music platform owns:

-   Audio streaming.
-   Music licensing.
-   Core music catalog.
-   Playback.
-   Subscription management.

Our service focuses on **intent understanding + retrieval +
personalization + ranking + playlist construction**.

------------------------------------------------------------------------

# 2. Core Design Principle

The most important architectural decision is:

> **The LLM should interpret the user's intent, not act as the
> authoritative music catalog or recommendation engine.**

Avoid:

``` text
"I'm feeling sad"
       |
       v
      LLM
       |
       v
"Here are 20 songs"
```

Problems:

-   Hallucinated songs.
-   Stale catalog knowledge.
-   Regionally unavailable tracks.
-   Poor personalization.
-   Repetitive/popularity-biased recommendations.
-   Difficult ranking control.

Instead:

``` text
Natural-Language Prompt
          |
          v
         LLM
          |
          v
   Structured Intent
          |
          v
 Candidate Retrieval
          |
          v
 Personalized Ranking
          |
          v
 Playlist Construction
          |
          v
 Valid Music Platform Tracks
```

The LLM becomes a **query-understanding layer**.

------------------------------------------------------------------------

# 3. API Design

## Generate Playlist

``` http
POST /v1/playlists/generate
```

Request:

``` json
{
  "prompt": "I'm feeling sad, give me something calm for about an hour",
  "durationMinutes": 60
}
```

Response:

``` json
{
  "playlistId": "pl_123",
  "title": "Quiet Evening",
  "tracks": [
    {
      "trackId": "track_1",
      "title": "...",
      "artist": "..."
    }
  ]
}
```

------------------------------------------------------------------------

## Refine Playlist

``` http
POST /v1/playlists/{playlistId}/refine
```

``` json
{
  "prompt": "Make it slightly more upbeat and add some Hindi songs"
}
```

The existing playlist and previous normalized intent provide context for
refinement.

------------------------------------------------------------------------

## Fetch Playlist

``` http
GET /v1/playlists/{playlistId}
```

------------------------------------------------------------------------

## Save Playlist

``` http
POST /v1/playlists/{playlistId}/save
```

The service calls the underlying music-platform API to persist the
playlist.

------------------------------------------------------------------------

# 4. High-Level Architecture


![Natural-Language=Personalised-Playlist - High Level Design](./images/playlist-generator-hld-v2.png.png)

> The diagram above is the original design used as the basis for the detailed discussion below.

``` text
                             USER
                              |
                              | natural-language prompt
                              v
                     +------------------+
                     |   API Gateway    |
                     | Auth / RateLimit |
                     +---------+--------+
                               |
                               v
                    +----------------------+
                    | Playlist Orchestrator|
                    +----------+-----------+
                               |
                +--------------+--------------+
                |                             |
                v                             v
       +------------------+          +------------------+
       | Intent Service   |          | User Profile     |
       | LLM / Small ML   |          | Service          |
       +--------+---------+          +---------+--------+
                |                              |
                | structured intent            | preferences
                +--------------+---------------+
                               |
                               v
                    +----------------------+
                    | Candidate Retrieval  |
                    | Service              |
                    +----------+-----------+
                               |
             +-----------------+-----------------+
             |                 |                 |
             v                 v                 v
       +-----------+     +-------------+    +------------+
       | Metadata  |     | Vector      |    | User       |
       | Search    |     | Search      |    | History    |
       +-----------+     +-------------+    +------------+
             |                 |                 |
             +-----------------+-----------------+
                               |
                               v
                     +------------------+
                     | Ranking Service  |
                     +--------+---------+
                              |
                              v
                    +--------------------+
                    | Playlist Builder   |
                    | Diversity/Ordering |
                    +---------+----------+
                              |
                              v
                    +--------------------+
                    | Music Platform API |
                    | Catalog / Playlist |
                    +--------------------+
```

Feedback path:

``` text
User
 |
 | play / skip / like / dislike / regenerate
 v
Event Pipeline / Kafka
        |
        +---------------------+
        |                     |
        v                     v
User Profile Updates     Offline ML Training
                              |
                              v
                         Ranking Models
```

------------------------------------------------------------------------

# 5. Request Walkthrough

Consider:

``` text
"I'm driving through the mountains at sunset.
Give me nostalgic but hopeful songs,
mostly 90s and early 2000s,
and no heavy metal."
```

The request enters the Playlist Orchestrator.

Two operations can begin concurrently:

``` text
                    Playlist Orchestrator
                     /                 \
                    /                   \
                   v                     v
            Intent Service         User Profile
                 LLM                  Service
```

This reduces end-to-end latency.

------------------------------------------------------------------------

# 6. Intent Understanding

The LLM converts ambiguous language into a strict structured
representation.

Example:

``` json
{
  "moods": ["nostalgic", "hopeful"],
  "activity": "driving",
  "energy": "medium",
  "yearRange": [1990, 2005],
  "excludedGenres": ["heavy metal"],
  "durationMinutes": 60
}
```

Another prompt:

``` text
"I'm feeling sad. Give me quiet acoustic songs,
but I don't want breakup songs."
```

could become:

``` json
{
  "moods": ["sad", "calm"],
  "energy": "low",
  "genres": ["acoustic"],
  "excludedThemes": ["breakup"]
}
```

Use a strict schema / structured-output interface instead of parsing
arbitrary LLM prose.

### Why structured intent?

Downstream services should consume:

``` text
mood=sad
energy=low
genre=acoustic
excludeTheme=breakup
```

rather than needing to understand free-form LLM output.

It also improves:

-   Validation.
-   Observability.
-   Caching.
-   Testing.
-   Model replacement.
-   Cost optimization.

------------------------------------------------------------------------

# 7. User Profile Service

Natural-language intent is only one side of the recommendation.

The other side is the user.

Profile signals may include:

``` text
Favorite genres
Favorite artists
Liked tracks
Recently played tracks
Frequently skipped tracks
Disliked artists
Language preferences
Listening frequency
Discovery preference
Repeated-track tolerance
```

For example:

``` text
Prompt:
"I'm feeling sad"

Intent:
mood = sad
energy = low

User profile:
likes indie
likes Hindi music
frequently listens to Artist A
frequently skips heavy metal
```

Two users issuing the same prompt should therefore be able to receive
different playlists.

### Profile storage

The online profile store should contain precomputed features needed for
low-latency ranking rather than scanning the user's complete listening
history during every request.

Conceptually:

``` text
Raw listening events
        |
        v
Stream / Batch Processing
        |
        v
Derived User Features
        |
        v
Online Profile Store
```

------------------------------------------------------------------------

# 8. Candidate Retrieval

Do not retrieve exactly the number of songs required for the playlist.

For a 20-song playlist, retrieve perhaps:

``` text
500–2000 candidates
```

and rank them.

Candidate retrieval uses multiple sources:

``` text
                     Structured Intent
                            |
             +--------------+--------------+
             |              |              |
             v              v              v
        Metadata         Semantic       Personalized
         Search           Search        Candidates
             |              |              |
             +--------------+--------------+
                            |
                            v
                       Candidate Set
```

------------------------------------------------------------------------

# 9. Metadata Retrieval

Metadata retrieval handles explicit constraints well.

Examples:

``` text
genre = rock
year >= 1990
year <= 2005
artist != X
language = Hindi
region = India
explicit = false
```

This can be backed by the music catalog/search infrastructure.

Hard constraints should normally be applied before expensive ranking.

------------------------------------------------------------------------

# 10. Semantic Retrieval and Embeddings

Some concepts do not map cleanly to catalog fields:

``` text
"rainy Sunday morning"

"driving through mountains at sunset"

"feeling nostalgic but hopeful"

"music for deep concentration"
```

Track/catalog enrichment can create embeddings representing semantic
properties.

Conceptually:

``` text
Track
 |
 +-- title
 +-- artist
 +-- genre
 +-- year
 +-- language
 +-- mood
 +-- energy
 +-- tempo
 +-- popularity
 +-- semantic embedding
```

The prompt or normalized intent can also be embedded:

``` text
"melancholic quiet acoustic music"
              |
              v
           Embedding
              |
              v
        Vector Search
              |
              v
 Semantically Similar Tracks
```

Vector similarity is a **candidate-generation signal**, not the final
ranking criterion.

------------------------------------------------------------------------

# 11. Candidate Merging

Candidate sources may overlap.

Example:

``` text
Metadata search       -> 500 tracks
Vector search         -> 500 tracks
User-history retrieval-> 300 tracks
Trending/contextual   -> 200 tracks
```

Merge and deduplicate:

``` text
Candidate Sources
       |
       v
Dedup / Merge
       |
       v
~1000 unique candidates
```

Every candidate should ultimately map to a valid catalog track ID.

------------------------------------------------------------------------

# 12. Ranking Service

Candidate retrieval asks:

> "Which tracks might be relevant?"

Ranking asks:

> "Which of these tracks are best for this user and this request?"

Conceptually:

``` text
score =
      intent_match
    + semantic_similarity
    + user_artist_affinity
    + user_genre_affinity
    + contextual_match
    + discovery_score
    + quality/popularity_signal
    - recent_play_penalty
    - skip_penalty
    - repetition_penalty
```

In production this would normally be learned rather than permanently
hand-coded.

Possible features:

``` text
Prompt <-> track similarity
Mood match
Energy match
Genre match
User <-> artist affinity
User <-> genre affinity
Language preference
Recent play count
Skip rate
Like probability
Novelty
Popularity
Regional availability
```

Pipeline:

``` text
~1000 candidates
       |
       v
Ranking Model
       |
       v
Top ~50–100 candidates
```

------------------------------------------------------------------------

# 13. Ranking Is Not Playlist Construction

This distinction is important.

Suppose the top-ranked tracks are:

``` text
1. Artist A - Track 1
2. Artist A - Track 2
3. Artist A - Track 3
4. Artist A - Track 4
5. Artist A - Track 5
```

Individually these may be excellent recommendations.

Together they may create a poor playlist.

The Playlist Builder therefore applies global constraints:

-   Artist diversity.
-   Genre diversity.
-   Duplicate prevention.
-   Target duration.
-   Explicit-content settings.
-   Regional availability.
-   Recently played limits.
-   Energy progression.
-   Discovery vs familiarity balance.

Conceptually:

``` text
Ranked Candidates
        |
        v
Constraint / Diversity Optimization
        |
        v
Sequencing
        |
        v
Final Playlist
```

------------------------------------------------------------------------

# 14. Playlist Progression

Consider the prompt:

> "I'm going through a breakup. Start very sad, but gradually make the
> playlist hopeful."

This is not just a retrieval problem.

The LLM can create a **playlist plan**:

``` json
{
  "durationMinutes": 60,
  "segments": [
    {
      "position": "start",
      "mood": "very sad",
      "energy": "low",
      "percentage": 30
    },
    {
      "position": "middle",
      "mood": "reflective",
      "energy": "medium-low",
      "percentage": 40
    },
    {
      "position": "end",
      "mood": "hopeful",
      "energy": "medium",
      "percentage": 30
    }
  ]
}
```

Then retrieve/rank candidates for each segment:

``` text
             Playlist Plan
                  |
       +----------+----------+
       |          |          |
       v          v          v
     SAD      REFLECTIVE   HOPEFUL
   candidates  candidates  candidates
       |          |          |
       +----------+----------+
                  |
                  v
           Playlist Builder
                  |
                  v
       Global sequencing optimization
```

The builder still ensures artist diversity and smooth transitions
between segments.

This is a strong example of using the LLM for **planning**, while
keeping actual song retrieval grounded in the catalog.

------------------------------------------------------------------------

# 15. Music Platform Integration

The underlying music platform remains responsible for:

``` text
Catalog
Playback
Licensing
Regional availability
User music library
Playlist persistence
```

Our service owns:

``` text
Prompt understanding
Intent planning
Personalization
Candidate retrieval
Ranking
Playlist construction
```

After generation:

``` text
trackId1
trackId2
trackId3
...
```

the Playlist Service calls the platform's playlist API to persist the
playlist.

Never depend on the LLM's internal knowledge to establish that a song is
playable.

------------------------------------------------------------------------

# 16. Latency Budget

Suppose the target is:

``` text
P95 playlist generation < 3–5 seconds
```

Example sequential budget:

``` text
LLM intent extraction       1.2 sec
Profile lookup              0.2 sec
Candidate retrieval         0.5 sec
Ranking                     0.4 sec
Playlist construction       0.2 sec
------------------------------------
Total                       2.5 sec
```

But profile retrieval does not depend on intent extraction.

Run them concurrently:

``` text
                Playlist Orchestrator
                  /             \
                 v               v
            LLM Intent      User Profile
                 \               /
                  +-------------+
                         |
                         v
                 Candidate Retrieval
                         |
                         v
                      Ranking
```

Cache:

-   User profile features.
-   Track metadata.
-   Track embeddings.
-   Common normalized intents.
-   Popular candidate sets where appropriate.

Do **not** cache the final playlist solely by prompt when
personalization is expected.

------------------------------------------------------------------------

# 17. LLM Cost Optimization

At millions of requests per day, sending every prompt to a large model
can be expensive.

Use model routing:

``` text
                     Prompt
                       |
                       v
                 Intent Router
                  /         \
                 /           \
                v             v
          Simple Prompt    Complex Prompt
                |             |
                v             v
          Small Model      Large LLM
```

Example:

``` text
"workout playlist"
```

may be handled by a lightweight classifier.

But:

``` text
"I'm driving through the mountains at sunset.
Give me something nostalgic but hopeful,
mostly 90s and early 2000s."
```

may benefit from the LLM.

Other optimizations:

-   Cache common intent parsing results.
-   Use smaller models for classification.
-   Limit output tokens.
-   Use strict structured output.
-   Avoid asking the LLM to produce song-by-song explanations unless
    requested.

------------------------------------------------------------------------

# 18. Graceful Degradation

## What if the LLM is unavailable?

The whole playlist service should not fail.

Fallback:

``` text
Prompt
  |
  v
LLM unavailable
  |
  v
Embedding / Keyword / Intent Classifier
  |
  v
Candidate Retrieval
  |
  v
Ranking
```

Common intents such as:

``` text
workout
sad
happy
focus
sleep
party
romantic
```

can often be handled by lightweight models.

The result may be less nuanced, but the service remains available.

------------------------------------------------------------------------

# 19. Feedback Loop

Capture user interactions:

``` text
Play
Skip
Like
Dislike
Add to library
Remove
Replay
Regenerate
Refine prompt
```

Architecture:

``` text
User Interaction
       |
       v
 Event Collection
       |
       v
      Kafka
      /   \
     /     \
    v       v
Profile    Offline ML
Updater    Pipeline
    |          |
    v          v
Online      Training /
Profile     Evaluation
Store          |
               v
          Ranking Model
```

This gives both:

-   Near-real-time profile updates.
-   Longer-term ranking-model improvements.

------------------------------------------------------------------------

# 20. Storage

Possible logical stores:

``` text
User Profile Store
------------------
userId -> derived recommendation features

Playlist Store
--------------
playlistId
userId
prompt
normalizedIntent
trackIds
createdAt
version

Catalog / Search Index
----------------------
track metadata
availability
semantic features
embeddings

Event Store / Stream
--------------------
plays
skips
likes
dislikes
regenerations
```

The exact database technology should follow access patterns rather than
being selected first.

------------------------------------------------------------------------

# 21. Reliability

The generation request is mostly a synchronous read/compute path:

``` text
Prompt
 -> Intent
 -> Retrieval
 -> Ranking
 -> Construction
```

Noncritical work should be asynchronous:

``` text
Playlist Generated
       |
       +--> analytics
       |
       +--> recommendation feedback
       |
       +--> model-training events
```

Saving a playlist to an external music platform should have clear
retry/idempotency semantics.

For example:

``` text
saveRequestId
```

can prevent repeated client retries from creating duplicate playlists.

------------------------------------------------------------------------

# 22. Privacy

Listening history can reveal significant personal preferences, so access
should be minimized.

Principles:

-   Obtain appropriate authorization before using account history.
-   Separate authentication identity from recommendation features where
    practical.
-   Encrypt sensitive data at rest and in transit.
-   Apply retention policies.
-   Restrict internal access.
-   Do not send unnecessary raw listening history to the LLM.

Instead of sending:

``` text
User's last 5,000 listening events
```

send derived features such as:

``` text
preferredGenres = [...]
preferredLanguages = [...]
artistAffinities = [...]
discoveryPreference = ...
```

This also reduces token usage and latency.

------------------------------------------------------------------------

# 23. Observability

Monitor the pipeline by stage.

### API

``` text
Request QPS
P50/P95/P99 latency
Error rate
Rate limiting
```

### LLM

``` text
Latency
Error rate
Token usage
Cost/request
Schema-validation failure
Fallback rate
```

### Retrieval

``` text
Candidate count
Vector-search latency
Metadata-search latency
Empty-result rate
```

### Ranking

``` text
Ranking latency
Feature lookup failures
Model version
```

### Playlist Quality

Online signals:

``` text
Skip rate
Completion rate
Like rate
Save rate
Regeneration rate
Playlist abandonment
```

These are more meaningful than infrastructure metrics alone.

------------------------------------------------------------------------

# 24. Staff-Level Question: Why Not Let the LLM Pick the Songs?

### Answer

Because the LLM is not an authoritative, continuously updated
representation of the music catalog.

It may hallucinate songs, return unavailable tracks, miss regional
licensing constraints, and does not have sufficient access to real-time
user behavior.

Use:

``` text
LLM
= understand intent / produce plan

Retrieval
= find valid candidate tracks

Ranking
= personalize

Playlist Builder
= optimize the whole playlist
```

This also makes each layer independently testable and replaceable.

------------------------------------------------------------------------

# 25. Staff-Level Question: How Do You Handle "Sad → Hopeful"?

A single vector search for:

``` text
sad hopeful breakup
```

is insufficient because the requirement concerns **ordering over time**.

Use the LLM to generate a structured playlist plan:

``` text
0–30%   very sad / low energy
30–70%  reflective
70–100% hopeful / increasing energy
```

Retrieve candidates for each stage.

Then the Playlist Builder optimizes:

``` text
segment relevance
+
transition smoothness
+
artist diversity
+
user preference
+
duration
```

The LLM plans the trajectory; the recommendation system chooses valid
tracks.

------------------------------------------------------------------------

# 26. Staff-Level Question: What if Candidate Retrieval Returns 10,000 Songs?

Use multi-stage ranking.

``` text
10,000 candidates
       |
       v
Cheap Filtering
       |
       v
~2,000
       |
       v
Lightweight Ranker
       |
       v
~200
       |
       v
Expensive Personalized Ranker
       |
       v
~50
       |
       v
Playlist Builder
       |
       v
20 tracks
```

Do not run the most expensive model against the entire catalog.

This is similar to large-scale search/recommendation architectures.

------------------------------------------------------------------------

# 27. Staff-Level Question: What if Retrieval Finds Too Few Songs?

Relax **soft constraints**, not hard constraints.

Suppose:

``` text
Hindi
1990–1992
sad
acoustic
female vocalist
not previously played
```

returns only five songs.

Classify constraints:

``` text
Hard:
regional availability
explicit-content preference
explicit artist exclusions

Soft:
exact year
preferred genre
novelty
energy
```

Then progressively relax soft constraints.

For example:

``` text
1990–1992
    |
    v
1988–1995
    |
    v
similar genre
```

The system should not silently violate hard safety/user constraints.

------------------------------------------------------------------------

# 28. Staff-Level Question: How Do You Avoid Every User Getting the Same Popular Songs?

Popularity should be only one ranking signal.

Use:

``` text
Personal affinity
Semantic relevance
Novelty
Discovery preference
Recent-play penalty
Artist diversity
Popularity
```

A user's exploration preference can control the familiar/discovery
balance.

For example:

``` text
80% familiar/relevant
20% discovery
```

versus a discovery-oriented user:

``` text
50% familiar
50% discovery
```

The exact percentages would be learned/tuned rather than permanently
hard-coded.

------------------------------------------------------------------------

# 29. Staff-Level Question: How Do You Handle a Brand-New User?

This is the cold-start problem.

Available signals may include:

-   Current prompt.
-   Region/language.
-   Explicit onboarding preferences, if provided.
-   Globally/locally popular tracks.
-   Contextual trends.

Flow:

``` text
No listening history
       |
       v
Prompt Intent
       +
Context
       +
Popularity / Quality
       |
       v
Initial Playlist
```

As interactions arrive:

``` text
plays / skips / likes
        |
        v
User profile becomes personalized
```

The prompt itself provides unusually strong cold-start context.

------------------------------------------------------------------------

# 30. Staff-Level Question: How Do You Keep Regional Availability Correct?

Availability must be checked against catalog data, not the LLM.

Candidate retrieval should filter by:

``` text
user storefront / region
```

and the final Playlist Builder should perform a final availability
validation before returning/saving tracks.

Why validate twice?

Catalog availability can change between retrieval and final playlist
creation.

------------------------------------------------------------------------

# 31. Staff-Level Question: What if the Ranking Service Is Down?

Graceful degradation:

``` text
Intent
  |
  v
Candidate Retrieval
  |
  v
Ranking unavailable
  |
  v
Fallback heuristic ranking
```

Fallback signals:

``` text
semantic relevance
+
catalog quality/popularity
+
basic profile affinity
-
recent-play penalty
```

Recommendation quality may decline, but playlist generation remains
available.

------------------------------------------------------------------------

# 32. Staff-Level Question: How Do You Prevent Prompt Injection?

Treat the user's prompt as **data**, not trusted system instructions.

The LLM receives:

``` text
System-defined schema
+
User prompt in a clearly separated field
```

The LLM's only permitted output is a constrained intent/plan schema.

It cannot directly:

-   call arbitrary internal APIs;
-   execute code;
-   modify authorization;
-   bypass catalog constraints.

All downstream fields are validated.

For example, an LLM-produced:

``` text
userId = another-user
```

must never influence authorization.

Identity comes from authenticated request context, not model output.

------------------------------------------------------------------------

# 33. Staff-Level Question: What Happens if the User Refines Repeatedly?

Example:

``` text
"Sad acoustic music"
        |
        v
"More upbeat"
        |
        v
"Mostly Hindi"
        |
        v
"No Artist X"
```

Do not repeatedly reinterpret the entire conversation blindly.

Maintain a structured playlist session:

``` json
{
  "mood": "sad",
  "energy": "medium",
  "genres": ["acoustic"],
  "languages": ["Hindi"],
  "excludedArtists": ["Artist X"]
}
```

Each refinement produces a validated delta:

``` text
Previous Intent
      +
Intent Delta
      |
      v
Updated Intent
```

Then retrieve/rank again as necessary.

This provides predictable behavior and reduces token usage.

------------------------------------------------------------------------

# 34. Staff-Level Question: How Do You Measure Whether the System Is Good?

Offline model metrics alone are insufficient.

Important online metrics include:

``` text
Playlist save rate
Track skip rate
Playlist completion rate
Like rate
Regeneration rate
Prompt refinement rate
Listening duration
```

Run controlled experiments/A-B tests on:

-   Ranking models.
-   Candidate sources.
-   Diversity strategies.
-   Playlist sequencing.
-   Intent models.

Guardrails should include:

-   Latency.
-   LLM cost.
-   Error rate.
-   Catalog-validity rate.

The objective is not simply maximizing clicks; recommendation quality
should reflect sustained user satisfaction.

------------------------------------------------------------------------

# 35. Staff-Level Question: Where Do You Use Caching?

Useful caches:

``` text
User profile features
Track metadata
Track embeddings
Popular candidate retrievals
Normalized common intents
```

Be careful caching:

``` text
prompt -> final playlist
```

because:

``` text
same prompt
+
different users
=
different desired playlists
```

A candidate set for a common intent may be cacheable, while final
personalized ranking remains user-specific.

------------------------------------------------------------------------

# 36. Staff-Level Question: What Happens at 10× Traffic?

Suppose a viral feature causes generation traffic to jump 10×.

Protect expensive dependencies independently.

``` text
                API Gateway
                     |
               Admission Control
                     |
              Playlist Service
                /          \
               v            v
          LLM Service    Retrieval
               |
          concurrency limits
               |
             queue /
          graceful fallback
```

Strategies:

-   Rate limits.
-   Per-user quotas.
-   Autoscaling stateless orchestrators.
-   LLM concurrency limits.
-   Small-model fallback.
-   Cache common intent parsing.
-   Degrade optional explanation generation.
-   Protect ranking/search dependencies with timeouts and circuit
    breakers.

The system should degrade recommendation sophistication before becoming
unavailable.

------------------------------------------------------------------------

# 37. Staff-Level Question: Synchronous or Asynchronous Generation?

For normal playlists, synchronous generation is preferable because users
expect an immediate result.

``` text
Request
   |
   v
2–5 sec generation
   |
   v
Playlist
```

For unusually complex/long requests, an asynchronous model can be
supported:

``` http
POST /playlists/generate
```

``` json
{
  "requestId": "req123",
  "status": "PROCESSING"
}
```

followed by polling/push notification.

But avoid adding asynchronous complexity unless latency or workload
justifies it.

------------------------------------------------------------------------

# 38. Staff-Level Question: How Would You Rebuild the Vector Index?

The vector index should be treated as a derived serving structure.

Canonical track metadata/features live in the catalog/data platform.

``` text
Catalog + Track Features
          |
          v
Embedding Pipeline
          |
          v
Vector Index
```

If corrupted:

``` text
Catalog
   +
Stored track features
   |
   v
Recompute / replay embeddings
   |
   v
New Vector Index
   |
   v
Atomic traffic switch
```

Version embeddings and indexes so old and new model versions can coexist
during migration.

------------------------------------------------------------------------

# 39. Staff-Level Question: How Do Model Updates Work Safely?

Version everything:

``` text
intentModelVersion
embeddingModelVersion
rankingModelVersion
playlistBuilderVersion
```

Example request trace:

``` text
intent=v12
embedding=v7
ranker=v31
builder=v8
```

This enables:

-   Reproducibility.
-   Debugging.
-   A/B tests.
-   Canary rollout.
-   Rollback.

Avoid changing the embedding model in-place without
rebuilding/versioning the associated vector index because vectors from
incompatible embedding spaces cannot safely be compared.

------------------------------------------------------------------------

# 40. Staff-Level Question: How Do You Debug "This Playlist Is Bad"?

Every generated playlist should have an internal recommendation trace.

For each selected track record:

``` text
retrieval source
intent-match score
semantic score
profile affinity
ranking score
diversity adjustment
segment assignment
constraint decisions
model versions
```

Not necessarily exposed to the user, but available for debugging.

This lets engineers answer:

``` text
Why was this track retrieved?
Why was it ranked highly?
Why did another track disappear?
Was a user preference stale?
Did the LLM misunderstand the prompt?
```

Without this observability, an ML/LLM recommendation system becomes
extremely difficult to operate.

------------------------------------------------------------------------

# 41. End-to-End Failure Strategy

``` text
LLM failure
   -> lightweight intent fallback

Profile failure
   -> generate non-personalized playlist

Vector search failure
   -> metadata/catalog retrieval

Ranking failure
   -> heuristic ranking

Playlist Builder issue
   -> simple diversity-constrained top-N

External music API failure
   -> retain generated playlist internally and retry save when appropriate
```

The important principle:

> **Recommendation quality can degrade in stages; availability should
> not depend on every intelligent component being healthy.**

------------------------------------------------------------------------

# 42. Interview-Ready Summary

> I would treat this primarily as a recommendation and retrieval system
> with an LLM-powered intent layer, rather than asking the LLM to
> directly generate song names.
>
> The Playlist Orchestrator sends the prompt to an intent service while
> fetching the user's precomputed recommendation profile in parallel.
> The intent service converts natural language into a validated
> structured representation containing mood, genre, activity, energy,
> era, exclusions, duration and potentially a multi-stage playlist plan.
>
> Candidate generation then retrieves valid, regionally available tracks
> from metadata search, semantic/vector search, and personalized
> candidate sources. A multi-stage ranking system scores those
> candidates using intent relevance, semantic similarity and user
> affinity. A separate Playlist Builder optimizes the selected tracks
> globally for artist diversity, duration, repetition and sequencing.
>
> The underlying music platform remains authoritative for catalog
> validity, regional availability, playback and playlist persistence.
>
> For more complex prompts such as "start sad and gradually become
> hopeful," the LLM produces a structured trajectory, but retrieval and
> ranking still select the actual tracks.
>
> I would also design explicit fallbacks so failures in the LLM, profile
> service, vector search or ranking model reduce recommendation quality
> rather than making playlist generation unavailable. At scale, I would
> use model routing, caching, multi-stage ranking, precomputed user
> features and strict observability around both infrastructure and
> recommendation-quality metrics.

------------------------------------------------------------------------

# 43. Key Design Principles to Remember

``` text
Natural language
      |
      v
LLM = understand intent
      |
      v
Structured Intent
      |
      +----------------------+
      |                      |
      v                      v
Metadata Retrieval      Semantic Retrieval
      |                      |
      +----------+-----------+
                 |
                 v
           Candidate Set
                 |
                 v
Personalized Ranking
                 |
                 v
Playlist Builder
                 |
                 v
Valid Catalog Tracks
                 |
                 v
Music Platform
```

Remember:

``` text
LLM
!=
Music Catalog

LLM
!=
Recommendation Engine

LLM
=
Intent Understanding + Planning
```

And:

``` text
Retrieval
= find plausible valid tracks

Ranking
= decide which tracks fit this user

Playlist Builder
= decide which tracks work well together and in what order
```

That separation is the core of the design.
