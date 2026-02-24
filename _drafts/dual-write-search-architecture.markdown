---                                                                                                                                                                            
  # Dual-Write Search Architecture with PostgreSQL Generated Columns                                                                                                             
                                                                                                                                                                                 
  ## The Problem                                                                                                                                                                 
                                                                                                                                                                                 
  The platform used a third-party search engine (Algolia) as its exclusive backend for indexed profiles. Algolia is an excellent fit for the public-facing search experience —   
  full-text, faceting, geo-ranking, typo-tolerance — but it created friction for backend use cases:

  - **Complex cross-domain queries** (e.g. "find all candidates matching a given job posting's learning path, within a geographic bounding box") required either multiple Algolia
   round-trips with client-side merging, or duplicating filtering logic that already existed in SQL.
  - **Availability dependency**: an Algolia outage blocked any background job that needed to inspect the search index.
  - **Cost and rate limits** made it unattractive to hammer the external API from batch indexation processes.

  The goal was to keep Algolia for everything the frontend needed, while introducing a **local database replica** of the same indexed data for backend queries.

  ---

  ## Schema Design: `search_talent`

  The table is deliberately minimal. There are only two "real" columns that the application ever writes:

  ```sql
  CREATE TABLE search_talent (
      id   INT  NOT NULL,   -- equals user id, PK + FK to users table
      data JSONB DEFAULT '{}' NOT NULL,
      PRIMARY KEY(id)
  );

  ALTER TABLE search_talent
      ADD CONSTRAINT search_talent_user_id_fkey
      FOREIGN KEY (id) REFERENCES users (id) ON DELETE CASCADE;
  ```

  `data` is a `JSONB` column that stores the **exact same normalised document** that is pushed to Algolia. The serialiser runs once and its output is fanned out to both backends
   simultaneously. There is a single source of truth for the document shape.

  ---

  ## Generated Columns: Extracting Structure from JSON

  Once `data` contains the full document, certain scalar fields need to be queryable efficiently. You cannot put a B-tree index on an arbitrary JSON path without materialising
  it. PostgreSQL's **generated stored columns** solve this cleanly:

  ```sql
  ALTER TABLE search_talent
      ADD COLUMN updated_at  BIGINT  GENERATED ALWAYS AS ((data->>'updatedAt')::BIGINT)  STORED,
      ADD COLUMN created_at  BIGINT  GENERATED ALWAYS AS ((data->>'createdAt')::BIGINT)  STORED NOT NULL,
      ADD COLUMN path_id     INT     GENERATED ALWAYS AS ((data->>'pathId')::INT)         STORED,
      ADD COLUMN country     VARCHAR GENERATED ALWAYS AS ((data->>'country')::VARCHAR)    STORED;
  ```

  ### How it works

  Every time a row is inserted or the `data` column is updated, PostgreSQL automatically recomputes these columns from the JSON content. The values are **physically stored on
  disk** alongside the row — not computed at query time — so they can be indexed like any ordinary column:

  ```sql
  CREATE INDEX idx_search_talent_path_id ON search_talent(path_id);
  CREATE INDEX idx_search_talent_country  ON search_talent(country);
  ```

  ### Enforcement at the ORM level

  These columns are declared `GENERATED ALWAYS`, meaning application code cannot write to them directly. In Doctrine's XML mapping this is enforced explicitly:

  ```xml
  <field
      name="updatedAt"
      type="bigint"
      insertable="false"
      updatable="false"
      column-definition="BIGINT GENERATED ALWAYS AS ((data->>'updatedAt')::BIGINT) STORED"
      generated="ALWAYS"
  />

  <field
      name="pathId"
      type="integer"
      insertable="false"
      updatable="false"
      column-definition="INT GENERATED ALWAYS AS ((data->>'pathId')::INT) STORED"
      generated="ALWAYS"
  />
  ```

  The application only ever writes `data`. The database maintains all derived columns automatically, with zero risk of them drifting out of sync with the JSON.

  ### Trade-offs

  | Pros | Cons |
  |---|---|
  | Single source of truth — no parallel columns to keep in sync with the JSON | Generated expressions reference JSON field names by string — renaming a key in the serialiser
  requires a schema migration |
  | Generated columns behave like ordinary columns for indexes and `WHERE` clauses | `STORED` physically duplicates data on disk — many generated columns on wide documents
  increases storage |
  | Adding a new filterable field only requires one `ALTER TABLE` | PostgreSQL-specific feature — not portable to MySQL or SQLite |
  | Automatically consistent: the expression re-evaluates on every write to `data` | |

  ---

  ## Geographic Search: `search_talent_box`

  Candidate profiles include one or more job search areas, each expressed as a geographic bounding box (northeast corner + southwest corner). Matching a point (recruiter
  location) against these is a **point-in-box inclusion** check.

  A separate child table holds the boxes:

  ```sql
  CREATE SEQUENCE search_talent_box_id_seq INCREMENT BY 1 MINVALUE 1 START 1;

  CREATE TABLE search_talent_box (
      id                INT NOT NULL,
      search_talent_id  INT NOT NULL,
      bounding_box      BOX NOT NULL,
      PRIMARY KEY(id),
      FOREIGN KEY (search_talent_id) REFERENCES search_talent(id) ON DELETE CASCADE
  );

  CREATE INDEX idx_search_talent_box_talent_id ON search_talent_box(search_talent_id);
  ```

  `BOX` is a native PostgreSQL geometric type representing a rectangle. The `@>` operator tests point containment:

  ```sql
  WHERE stb.bounding_box @> point(:latitude, :longitude)
  ```

  Because a candidate can declare multiple search areas, `DISTINCT` is applied to avoid duplicate results:

  ```sql
  SELECT DISTINCT(st.id), st.*
  FROM search_talent AS st
  INNER JOIN search_talent_box AS stb ON st.id = stb.search_talent_id
  WHERE stb.bounding_box @> point(:latitude, :longitude)
    AND st.path_id IN (:pathIds)
  ORDER BY st.updated_at DESC
  LIMIT :limit OFFSET :offset
  ```

  Note that `updated_at` in the `ORDER BY` clause is the **generated column** — no JSON path expression is needed in the query itself.

  ### Why a separate table instead of a JSON array?

  Containment queries on PostgreSQL `BOX` columns can use GiST indexes. Storing coordinates as a JSON array would require manual arithmetic or `ST_Contains` expressions — far
  more expensive, harder to index, and incompatible with the native `@>` operator.

  `ON DELETE CASCADE` ensures boxes are automatically removed when a candidate is deleted, and the application explicitly clears all boxes before re-persisting updated ones to
  avoid orphaned rows.

  ### Custom Doctrine DBAL type

  To handle the `BOX` PostgreSQL type transparently, a custom Doctrine DBAL type bridges the gap between the database wire format and a PHP value object:

  ```php
  // Serialisation: PHP value object → PostgreSQL BOX string
  // Output: '((lat_ne,lon_ne),(lat_sw,lon_sw))'
  public function convertToDatabaseValue($value, AbstractPlatform $platform): string
  {
      return sprintf(
          '((%F,%F),(%F,%F))',
          $value->northeast->getLatitude(),
          $value->northeast->getLongitude(),
          $value->southwest->getLatitude(),
          $value->southwest->getLongitude(),
      );
  }

  // Deserialisation: PostgreSQL BOX string → PHP value object
  public function convertToPHPValue($value, AbstractPlatform $platform): BoundingBox
  {
      [$neLat, $neLon, $swLat, $swLon] = sscanf($value, '(%f,%f),(%f,%f)');
      return new BoundingBox(
          new GeographicalCoordinates((float) $neLat, (float) $neLon),
          new GeographicalCoordinates((float) $swLat, (float) $swLon),
      );
  }
  ```

  This keeps all geometric concerns out of application code. The `BoundingBox` value object also provides static factories for building a box from a centre point and a radius
  (in kilometres), using standard geodesic approximations.

  ---

  ## The Composite Gateway: Dual-Write in Practice

  All write operations go through a `CompositeTalentSearchGateway` that holds an `iterable` of all registered search gateway implementations, wired via a Symfony service tag:

  ```php
  interface TalentSearchGateway
  {
      public function insert(Talent $talent): void;
      public function update(Talent $talent): void;
      public function bulkUpdate(array $talents, bool $createObjectIfNotExists = true): void;
      public function delete(int $userId): void;
      public function bulkDelete(array $userIds): void;
      public function deleteAll(): void;
      public function search(array $filters, int $itemsPerPage, int $page): Paginated;
  }
  ```

  ```php
  class CompositeTalentSearchGateway implements TalentSearchGateway
  {
      /** @param iterable<TalentSearchGateway> $talentSearchGateways */
      public function __construct(
          private readonly iterable $talentSearchGateways
      ) {}

      public function update(Talent $talent): void
      {
          foreach ($this->talentSearchGateways as $gateway) {
              $gateway->update($talent);
          }
      }
      // identical fan-out pattern for insert, bulkInsert, bulkUpdate, delete, bulkDelete, deleteAll
  }
  ```

  Each implementation self-registers with the tag:

  ```php
  #[AutoconfigureTag(name: 'talent.search.gateway')]
  class DoctrineTalentSearchGateway extends ServiceEntityRepository implements TalentSearchGateway { ... }

  #[AutoconfigureTag(name: 'talent.search.gateway')]
  class AlgoliaTalentSearchGateway implements TalentSearchGateway { ... }
  ```

  Adding a new search backend (e.g. Elasticsearch, Typesense) requires only implementing the interface and adding the tag — zero changes to the composite or to any use case.

  The composite intentionally **does not** implement `findById` or `search`. Those are called directly on the specific implementation that owns the query. The composite is
  write-only by design.

  ---

  ## Upsert and the `skipUpdatedAt` Flag

  ### Upsert

  `DoctrineTalentSearchGateway::insert` implements an upsert: it first attempts an update, and falls back to a real insert only if the row does not exist:

  ```php
  public function insert(Talent $talent): void
  {
      try {
          $this->update(talent: $talent);
      } catch (TalentSearchNotFoundException) {
          $this->_em->persist($this->createSearchTalent($talent));
          $this->_em->flush();
      }
  }
  ```

  This makes the operation idempotent — safe to call from a re-indexation batch regardless of whether a row already exists.

  Bulk operations use a **load-then-merge** strategy: existing rows are fetched in one `SELECT IN` query, updated in memory, and flushed in a single batch. Missing IDs trigger a
   fresh `persist`. Rows are processed in chunks of 500 to avoid memory exhaustion:

  ```php
  public function bulkUpdate(array $talents, bool $createObjectIfNotExists = true): void
  {
      foreach (array_chunk($talents, 500) as $chunk) {
          $this->_em->clear(); // free memory between chunks

          $ids = array_map(fn (Talent $t) => $t->getUser()->getId(), $chunk);
          $existingMap = $this->findIndexedById($ids); // one SELECT IN query

          foreach ($chunk as $talent) {
              $searchTalent = $existingMap[$talent->getId()] ?? $this->createSearchTalent($talent);
              $searchTalent->setData($this->normalizeTalent($talent));
              $this->_em->persist($searchTalent);
          }
          $this->_em->flush();
      }
  }
  ```

  ### `skipUpdatedAt`

  Certain batch operations — re-indexation for a new attribute, administrative corrections, backfills — should **not** reset the `updatedAt` timestamp, because that would
  incorrectly reset the profile expiration clock. A `skipUpdatedAt` flag on the domain entity handles this:

  ```php
  if ($talent->getSkipUpdatedAt() === true) {
      // discard the freshly serialised updatedAt,
      // restore the previously stored value from the JSON
      $normalizedTalent['updatedAt'] = $searchTalent->getData()['updatedAt'] ?? null;
  }
  $searchTalent->setData($normalizedTalent);
  ```

  Because `updated_at` is a **generated column** derived from `data->>'updatedAt'`, preserving the JSON value automatically preserves the generated column — no special SQL
  handling is required.

  ---

  ## End-to-End Data Flow

  ```
  User action
      └─▶ Use case executes
              └─▶ Domain event dispatched
                      └─▶ Event subscriber
                              └─▶ Messenger bus  [async message]
                                      └─▶ Message handler
                                              ├─▶ Read from source-of-truth DB
                                              └─▶ Serialise once  (single normaliser run)
                                                      ├─▶ AlgoliaTalentSearchGateway
                                                      │       └─▶ Algolia  (frontend: full-text, facets, geo-ranking)
                                                      └─▶ DoctrineTalentSearchGateway
                                                              └─▶ PostgreSQL
                                                                      ├─▶ data  (JSONB)
                                                                      └─▶ generated columns
                                                                            updated_at, path_id, country …
                                                                            (backend: complex joins, point-in-box)
  ```

  The single serialisation pass is central to the design: both backends receive the same document. There is no risk of the Algolia document and the database document diverging
  in structure — only in timing, since the database write happens within the same ORM flush while the Algolia write is a separate HTTP call made in the same handler.

  ---

  ## Summary

  | Concern | Solution | Key trade-off |
  |---|---|---|
  | Frontend search (full-text, facets, geo-ranking) | Algolia | External dependency, cost |
  | Backend complex queries (cross-domain joins, containment) | PostgreSQL `search_talent` table | Data duplication |
  | Avoiding dual serialisation | Single normaliser, fanned out via Composite gateway | Both backends must accept the same document shape |
  | Efficient scalar filtering on JSON fields | PostgreSQL generated stored columns | JSON key names become part of the DB schema contract |
  | Geographic containment queries | PostgreSQL `BOX` type + `@>` operator in a child table | PostgreSQL-specific; requires a custom Doctrine DBAL type |
  | Safe re-indexation without expiration side effects | `skipUpdatedAt` flag | Small leakage of infrastructure concerns into the domain entity |
  | Idempotent writes | Upsert (update-or-insert) | Slightly higher read cost on the insert path |
