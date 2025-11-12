# Elasticsearch & Search - Interview Answers

## Question 1: How would you design a search system using Elasticsearch for a large e-commerce platform?

### Conceptual Explanation

Designing an e-commerce search system requires balancing relevance, performance, and user experience. Key components include: **data indexing strategy** (which fields to index, how to structure documents), **search relevance** (scoring, boosting, filters), **performance optimization** (caching, query optimization), and **data synchronization** (keeping Elasticsearch in sync with primary database).

The architecture typically involves: PostgreSQL/MongoDB as source of truth, a sync mechanism (Change Data Capture, event streaming, or scheduled indexing), Elasticsearch cluster for search, and API layer handling search requests with fallback mechanisms. Consider product catalog search (full-text, filters, facets), autocomplete, spell correction, and personalized results.

Design decisions depend on scale: small catalogs (<100K products) can use simple sync jobs, large catalogs (millions of products) need streaming updates via Kafka and optimized sharding strategies. Always implement caching (Redis) for popular searches and analytics tracking for search behavior.

### Code Example

```javascript
// Elasticsearch index design for e-commerce products
const productIndexMapping = {
  settings: {
    number_of_shards: 5,
    number_of_replicas: 2,
    analysis: {
      analyzer: {
        product_name_analyzer: {
          type: 'custom',
          tokenizer: 'standard',
          filter: ['lowercase', 'asciifolding', 'product_synonym', 'edge_ngram_filter']
        },
        autocomplete_analyzer: {
          type: 'custom',
          tokenizer: 'standard',
          filter: ['lowercase', 'edge_ngram_filter']
        }
      },
      filter: {
        edge_ngram_filter: {
          type: 'edge_ngram',
          min_gram: 2,
          max_gram: 20
        },
        product_synonym: {
          type: 'synonym',
          synonyms: [
            'laptop, notebook, computer',
            'phone, mobile, smartphone',
            'tv, television'
          ]
        }
      }
    }
  },
  mappings: {
    properties: {
      id: { type: 'keyword' },
      name: {
        type: 'text',
        analyzer: 'product_name_analyzer',
        fields: {
          keyword: { type: 'keyword' },
          autocomplete: { type: 'text', analyzer: 'autocomplete_analyzer' }
        }
      },
      description: {
        type: 'text',
        analyzer: 'standard'
      },
      category: {
        type: 'keyword'
      },
      brand: {
        type: 'keyword'
      },
      price: {
        type: 'float'
      },
      rating: {
        type: 'float'
      },
      reviews_count: {
        type: 'integer'
      },
      in_stock: {
        type: 'boolean'
      },
      tags: {
        type: 'keyword'
      },
      attributes: {
        type: 'nested',
        properties: {
          name: { type: 'keyword' },
          value: { type: 'keyword' }
        }
      },
      created_at: {
        type: 'date'
      },
      popularity_score: {
        type: 'float'
      }
    }
  }
};

// Create index
const { Client } = require('@elastic/elasticsearch');
const client = new Client({ node: 'http://localhost:9200' });

await client.indices.create({
  index: 'products',
  body: productIndexMapping
});

// Search API with multiple features
class ProductSearchService {
  constructor(esClient, redisClient) {
    this.es = esClient;
    this.redis = redisClient;
  }
  
  async search(query, filters = {}) {
    const { category, brand, minPrice, maxPrice, inStock, page = 1, pageSize = 20 } = filters;
    
    // Check cache for popular searches
    const cacheKey = `search:${query}:${JSON.stringify(filters)}`;
    const cached = await this.redis.get(cacheKey);
    if (cached) {
      return JSON.parse(cached);
    }
    
    // Build Elasticsearch query
    const must = [];
    const filter = [];
    
    // Main search query with boosting
    if (query) {
      must.push({
        multi_match: {
          query: query,
          fields: [
            'name^3',              // Boost name matches
            'name.autocomplete^2', // Boost autocomplete
            'description',
            'brand^2',
            'category',
            'tags'
          ],
          type: 'best_fields',
          fuzziness: 'AUTO'        // Handle typos
        }
      });
    }
    
    // Filters (don't affect scoring)
    if (category) {
      filter.push({ term: { category } });
    }
    
    if (brand) {
      filter.push({ term: { brand } });
    }
    
    if (minPrice || maxPrice) {
      filter.push({
        range: {
          price: {
            gte: minPrice || 0,
            lte: maxPrice || 999999
          }
        }
      });
    }
    
    if (inStock) {
      filter.push({ term: { in_stock: true } });
    }
    
    const searchBody = {
      from: (page - 1) * pageSize,
      size: pageSize,
      query: {
        function_score: {
          query: {
            bool: {
              must: must.length > 0 ? must : [{ match_all: {} }],
              filter
            }
          },
          functions: [
            // Boost by popularity
            {
              field_value_factor: {
                field: 'popularity_score',
                factor: 1.2,
                modifier: 'log1p',
                missing: 1
              }
            },
            // Boost by rating
            {
              field_value_factor: {
                field: 'rating',
                factor: 0.5,
                modifier: 'sqrt',
                missing: 1
              }
            },
            // Boost by reviews count
            {
              field_value_factor: {
                field: 'reviews_count',
                factor: 0.1,
                modifier: 'log1p',
                missing: 1
              }
            }
          ],
          score_mode: 'sum',
          boost_mode: 'multiply'
        }
      },
      // Aggregations for faceted search
      aggs: {
        categories: {
          terms: { field: 'category', size: 20 }
        },
        brands: {
          terms: { field: 'brand', size: 50 }
        },
        price_ranges: {
          range: {
            field: 'price',
            ranges: [
              { to: 50 },
              { from: 50, to: 100 },
              { from: 100, to: 500 },
              { from: 500 }
            ]
          }
        }
      },
      highlight: {
        fields: {
          name: {},
          description: {}
        }
      }
    };
    
    const result = await this.es.search({
      index: 'products',
      body: searchBody
    });
    
    const response = {
      total: result.hits.total.value,
      products: result.hits.hits.map(hit => ({
        ...hit._source,
        score: hit._score,
        highlights: hit.highlight
      })),
      facets: {
        categories: result.aggregations.categories.buckets,
        brands: result.aggregations.brands.buckets,
        priceRanges: result.aggregations.price_ranges.buckets
      },
      page,
      pageSize
    };
    
    // Cache for 5 minutes
    await this.redis.setex(cacheKey, 300, JSON.stringify(response));
    
    return response;
  }
  
  // Autocomplete endpoint
  async autocomplete(prefix) {
    const result = await this.es.search({
      index: 'products',
      body: {
        size: 10,
        query: {
          match: {
            'name.autocomplete': {
              query: prefix,
              operator: 'and'
            }
          }
        },
        _source: ['name', 'category'],
        aggs: {
          suggestions: {
            terms: {
              field: 'name.keyword',
              size: 10
            }
          }
        }
      }
    });
    
    return result.hits.hits.map(hit => hit._source.name);
  }
}

// Data synchronization from PostgreSQL
class ProductSyncService {
  constructor(db, esClient) {
    this.db = db;
    this.es = esClient;
  }
  
  // Full reindex (initial sync or recovery)
  async fullReindex() {
    console.log('Starting full reindex...');
    
    let offset = 0;
    const batchSize = 1000;
    
    while (true) {
      const products = await this.db.query(
        `SELECT * FROM products ORDER BY id LIMIT $1 OFFSET $2`,
        [batchSize, offset]
      );
      
      if (products.rows.length === 0) break;
      
      // Bulk index to Elasticsearch
      const body = products.rows.flatMap(product => [
        { index: { _index: 'products', _id: product.id } },
        this.transformProduct(product)
      ]);
      
      await this.es.bulk({ body, refresh: false });
      
      offset += batchSize;
      console.log(`Indexed ${offset} products`);
    }
    
    await this.es.indices.refresh({ index: 'products' });
    console.log('Full reindex complete');
  }
  
  // Incremental sync (via change events)
  async handleProductChange(event) {
    const { operation, productId, data } = event;
    
    try {
      if (operation === 'INSERT' || operation === 'UPDATE') {
        await this.es.index({
          index: 'products',
          id: productId,
          body: this.transformProduct(data)
        });
      } else if (operation === 'DELETE') {
        await this.es.delete({
          index: 'products',
          id: productId
        });
      }
      
      console.log(`Synced product ${productId} (${operation})`);
    } catch (error) {
      console.error(`Failed to sync product ${productId}:`, error);
      // Push to dead letter queue for retry
      await this.deadLetterQueue.add({ event, error: error.message });
    }
  }
  
  transformProduct(product) {
    return {
      id: product.id,
      name: product.name,
      description: product.description,
      category: product.category,
      brand: product.brand,
      price: parseFloat(product.price),
      rating: product.rating || 0,
      reviews_count: product.reviews_count || 0,
      in_stock: product.inventory_count > 0,
      tags: product.tags ? product.tags.split(',') : [],
      attributes: product.attributes || [],
      created_at: product.created_at,
      popularity_score: this.calculatePopularityScore(product)
    };
  }
  
  calculatePopularityScore(product) {
    // Custom scoring based on views, sales, etc.
    const views = product.view_count || 0;
    const sales = product.sales_count || 0;
    return Math.log(1 + views * 0.1 + sales * 10);
  }
}

// Express API endpoints
const searchService = new ProductSearchService(esClient, redisClient);

app.get('/api/search', async (req, res) => {
  try {
    const { q, category, brand, minPrice, maxPrice, inStock, page } = req.query;
    
    const results = await searchService.search(q, {
      category,
      brand,
      minPrice: minPrice ? parseFloat(minPrice) : undefined,
      maxPrice: maxPrice ? parseFloat(maxPrice) : undefined,
      inStock: inStock === 'true',
      page: page ? parseInt(page) : 1
    });
    
    res.json(results);
  } catch (error) {
    console.error('Search error:', error);
    res.status(500).json({ error: 'Search failed' });
  }
});

app.get('/api/autocomplete', async (req, res) => {
  try {
    const { q } = req.query;
    if (!q || q.length < 2) {
      return res.json([]);
    }
    
    const suggestions = await searchService.autocomplete(q);
    res.json(suggestions);
  } catch (error) {
    res.status(500).json({ error: 'Autocomplete failed' });
  }
});
```

### Best Practices

- **Design for relevance**: Use boosting, function scores, and custom analyzers to ensure best results appear first. Test with real user queries.
- **Implement faceted search**: Aggregations allow users to filter by category, brand, price range. Essential for e-commerce UX.
- **Use appropriate analyzers**: Custom analyzers with synonyms, stemming, and edge n-grams improve search quality. Test different configurations.
- **Cache popular queries**: Redis caching significantly reduces Elasticsearch load. Cache search results for 1-5 minutes.
- **Monitor query performance**: Track slow queries using Elasticsearch slow log. Optimize or cache problematic queries.
- **Implement autocomplete**: Edge n-gram tokenizer enables prefix matching. Consider separate suggestions index for better performance.
- **Handle data sync carefully**: Use event streaming (Kafka) for real-time sync or scheduled jobs for eventual consistency. Always have full reindex capability.

---

## Question 2: Explain the difference between term queries and match queries in Elasticsearch.

### Conceptual Explanation

**Term queries** are exact match queries that work on the inverted index without analysis. They search for the exact term as-is in the inverted index, making them fast and suitable for structured data like IDs, statuses, categories, or keyword fields. Term queries are case-sensitive and don't apply text analysis (tokenization, lowercasing, stemming).

**Match queries** are full-text queries that analyze the query string before searching, applying the same analyzer used during indexing. They tokenize, lowercase, remove stop words, and stem the query, making them ideal for human-readable text like product descriptions or blog posts. Match queries are more forgiving with user input and handle typos through fuzziness.

The key difference: term queries look for exact tokens in the index, while match queries process the input through analyzers first. Use term for filters (category = "Electronics"), use match for search (find products matching "running shoes").

### Code Example

```javascript
// Term Query Examples (exact match, no analysis)

// 1. Exact match on keyword field
const termQuery1 = {
  query: {
    term: {
      status: 'active'  // Searches for exact term "active"
    }
  }
};

// 2. Multiple terms (OR)
const termsQuery = {
  query: {
    terms: {
      category: ['electronics', 'computers', 'phones']  // Match any of these
    }
  }
};

// 3. Case sensitivity issue with term query
const termQuery2 = {
  query: {
    term: {
      brand: 'Nike'  // Won't match "nike" or "NIKE" if indexed differently
    }
  }
};

// 4. Term query on keyword subfield
const termQuery3 = {
  query: {
    term: {
      'name.keyword': 'iPhone 15 Pro'  // Exact match on entire string
    }
  }
};

// Match Query Examples (analyzed, full-text)

// 1. Basic match query
const matchQuery1 = {
  query: {
    match: {
      description: 'wireless bluetooth headphones'  
      // Tokenizes to: ["wireless", "bluetooth", "headphones"]
      // Matches documents containing any of these words
    }
  }
};

// 2. Match with operator
const matchQuery2 = {
  query: {
    match: {
      description: {
        query: 'wireless bluetooth headphones',
        operator: 'and'  // Document must contain all words
      }
    }
  }
};

// 3. Match with fuzziness (handles typos)
const matchQuery3 = {
  query: {
    match: {
      name: {
        query: 'iphne',  // Typo in "iphone"
        fuzziness: 'AUTO'  // Will still match "iphone"
      }
    }
  }
};

// 4. Match phrase (words in order)
const matchPhraseQuery = {
  query: {
    match_phrase: {
      description: 'noise cancelling technology'
      // Matches only if words appear in this exact order
    }
  }
};

// Practical comparison with same data

// Example document indexed
const product = {
  id: 'prod-123',
  name: 'Sony WH-1000XM5 Wireless Headphones',
  category: 'Electronics',
  brand: 'Sony',
  status: 'active',
  description: 'Premium wireless headphones with noise cancelling technology'
};

// TERM QUERIES on this document

// ✅ Works - exact match on keyword
await client.search({
  index: 'products',
  body: {
    query: { term: { status: 'active' } }
  }
});

// ✅ Works - exact match on keyword field
await client.search({
  index: 'products',
  body: {
    query: { term: { category: 'Electronics' } }
  }
});

// ❌ Doesn't work - term query on analyzed text field
await client.search({
  index: 'products',
  body: {
    query: { 
      term: { 
        description: 'noise cancelling'  // Won't match, needs individual tokens
      } 
    }
  }
});

// ✅ Works - term on individual token
await client.search({
  index: 'products',
  body: {
    query: { 
      term: { 
        description: 'wireless'  // Matches single token
      } 
    }
  }
});

// MATCH QUERIES on this document

// ✅ Works - analyzed and matches
await client.search({
  index: 'products',
  body: {
    query: { 
      match: { 
        description: 'wireless noise cancelling'  // Finds document
      } 
    }
  }
});

// ✅ Works - handles typos with fuzziness
await client.search({
  index: 'products',
  body: {
    query: { 
      match: { 
        name: {
          query: 'Soni Headphnes',  // Typos
          fuzziness: 'AUTO'
        }
      } 
    }
  }
});

// ❌ Doesn't work well - match on keyword field (wastes analysis)
await client.search({
  index: 'products',
  body: {
    query: { 
      match: { 
        category: 'Electronics'  // Works but term query is better
      } 
    }
  }
});

// Combining both in bool query (common pattern)
const combinedQuery = {
  query: {
    bool: {
      must: [
        // Full-text search with match
        {
          match: {
            description: 'wireless headphones'
          }
        }
      ],
      filter: [
        // Exact filters with term
        { term: { category: 'Electronics' } },
        { term: { status: 'active' } },
        { range: { price: { lte: 500 } } }
      ]
    }
  }
};

// Why this matters: performance and correctness
// Filter context (term queries) is cached and doesn't affect scoring
// Must context (match queries) affects relevance scoring

const efficientSearchQuery = {
  query: {
    bool: {
      must: [
        // Score this
        {
          multi_match: {
            query: 'noise cancelling headphones',
            fields: ['name^3', 'description'],
            fuzziness: 'AUTO'
          }
        }
      ],
      filter: [
        // Don't score, just filter (faster, cacheable)
        { term: { status: 'active' } },
        { term: { in_stock: true } },
        { terms: { category: ['Electronics', 'Audio'] } },
        { range: { rating: { gte: 4.0 } } }
      ]
    }
  }
};
```

### Best Practices

- **Use term for structured data**: IDs, statuses, categories, boolean flags, dates. Term queries are faster and more predictable for exact matches.
- **Use match for user input**: Product names, descriptions, reviews. Match queries handle natural language, typos, and variations.
- **Combine in bool queries**: Use match in must clause for scoring, term in filter clause for exact filtering. Filters are cached and don't affect scoring.
- **Mind the field type**: Use keyword type for term queries, text type for match queries. Map fields appropriately during index creation.
- **Understand case sensitivity**: Term queries are case-sensitive. If you need case-insensitive exact matching, ensure field is lowercase-analyzed or use keyword normalizer.

---

## Question 3: How do you sync data between PostgreSQL and Elasticsearch efficiently?

### Conceptual Explanation

Syncing data between PostgreSQL (source of truth) and Elasticsearch (search index) requires handling: initial bulk load, real-time updates, handling failures, and ensuring consistency. Strategies include: **scheduled batch sync** (simple, eventual consistency), **Change Data Capture (CDC)** with Debezium (near real-time), **event-driven sync** via Kafka (real-time, scalable), and **trigger-based sync** (simple but couples systems).

The best approach depends on consistency requirements and scale. For small datasets (<1M records), scheduled jobs work fine. For large, frequently-changing data, use CDC or event streaming. Always implement: idempotency (handle duplicate syncs), error handling with retry, full reindex capability, and monitoring/alerting.

Key considerations: PostgreSQL remains authoritative, Elasticsearch can be rebuilt from PostgreSQL, handle schema changes carefully, and monitor sync lag to ensure searches reflect recent data.

### Code Example

```javascript
// Approach 1: Scheduled Batch Sync (Simple, eventual consistency)

class BatchSyncService {
  constructor(db, esClient) {
    this.db = db;
    this.es = esClient;
    this.batchSize = 1000;
  }
  
  async syncProducts() {
    console.log('Starting batch sync...');
    
    // Get last sync timestamp
    const lastSync = await this.getLastSyncTimestamp();
    
    // Query products updated since last sync
    const products = await this.db.query(`
      SELECT * FROM products 
      WHERE updated_at > $1 
      ORDER BY updated_at ASC
    `, [lastSync]);
    
    if (products.rows.length === 0) {
      console.log('No products to sync');
      return;
    }
    
    // Bulk index to Elasticsearch
    const body = products.rows.flatMap(product => [
      { 
        index: { 
          _index: 'products', 
          _id: product.id 
        } 
      },
      {
        id: product.id,
        name: product.name,
        description: product.description,
        price: parseFloat(product.price),
        category: product.category,
        updated_at: product.updated_at
      }
    ]);
    
    const result = await this.es.bulk({ body });
    
    if (result.errors) {
      const errors = result.items.filter(item => item.index.error);
      console.error('Bulk index errors:', errors);
    }
    
    // Update last sync timestamp
    const latestUpdate = products.rows[products.rows.length - 1].updated_at;
    await this.setLastSyncTimestamp(latestUpdate);
    
    console.log(`Synced ${products.rows.length} products`);
  }
  
  // Run every 5 minutes
  startScheduledSync() {
    setInterval(() => {
      this.syncProducts().catch(console.error);
    }, 5 * 60 * 1000);
  }
  
  async getLastSyncTimestamp() {
    const result = await this.db.query(
      'SELECT last_sync FROM sync_state WHERE entity = $1',
      ['products']
    );
    return result.rows[0]?.last_sync || new Date(0);
  }
  
  async setLastSyncTimestamp(timestamp) {
    await this.db.query(`
      INSERT INTO sync_state (entity, last_sync) 
      VALUES ($1, $2)
      ON CONFLICT (entity) DO UPDATE SET last_sync = $2
    `, ['products', timestamp]);
  }
}

// Approach 2: Event-Driven Sync via Kafka (Real-time, scalable)

// PostgreSQL trigger publishes to notification channel
/*
CREATE OR REPLACE FUNCTION notify_product_change()
RETURNS trigger AS $$
DECLARE
  payload JSON;
BEGIN
  IF TG_OP = 'DELETE' THEN
    payload = json_build_object('operation', 'DELETE', 'id', OLD.id);
  ELSE
    payload = json_build_object(
      'operation', TG_OP,
      'id', NEW.id,
      'data', row_to_json(NEW)
    );
  END IF;
  
  PERFORM pg_notify('product_changes', payload::text);
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER product_change_trigger
AFTER INSERT OR UPDATE OR DELETE ON products
FOR EACH ROW EXECUTE FUNCTION notify_product_change();
*/

// Node.js service listens to notifications
const { Client } = require('pg');

class PostgresEventListener {
  constructor(db, kafka) {
    this.db = db;
    this.kafka = kafka;
    this.producer = kafka.producer();
  }
  
  async start() {
    await this.producer.connect();
    
    const client = new Client({
      host: process.env.DB_HOST,
      database: process.env.DB_NAME,
      user: process.env.DB_USER,
      password: process.env.DB_PASSWORD
    });
    
    await client.connect();
    await client.query('LISTEN product_changes');
    
    client.on('notification', async (msg) => {
      if (msg.channel === 'product_changes') {
        const event = JSON.parse(msg.payload);
        
        // Publish to Kafka
        await this.producer.send({
          topic: 'product-changes',
          messages: [{
            key: event.id,
            value: JSON.stringify(event)
          }]
        });
        
        console.log(`Published event for product ${event.id}`);
      }
    });
    
    console.log('Listening for product changes...');
  }
}

// Kafka consumer syncs to Elasticsearch
const { Kafka } = require('kafkajs');

class ElasticsearchSyncConsumer {
  constructor(kafka, esClient) {
    this.kafka = kafka;
    this.es = esClient;
    this.consumer = kafka.consumer({ groupId: 'es-sync-group' });
  }
  
  async start() {
    await this.consumer.connect();
    await this.consumer.subscribe({ topic: 'product-changes' });
    
    await this.consumer.run({
      eachMessage: async ({ message }) => {
        const event = JSON.parse(message.value.toString());
        
        try {
          await this.syncToElasticsearch(event);
          console.log(`Synced product ${event.id} to Elasticsearch`);
        } catch (error) {
          console.error(`Failed to sync product ${event.id}:`, error);
          // Send to dead letter queue for retry
          await this.handleSyncError(event, error);
        }
      }
    });
  }
  
  async syncToElasticsearch(event) {
    const { operation, id, data } = event;
    
    if (operation === 'DELETE') {
      await this.es.delete({
        index: 'products',
        id: id,
        refresh: false  // Don't refresh immediately for better performance
      });
    } else {
      // For INSERT and UPDATE
      await this.es.index({
        index: 'products',
        id: id,
        body: {
          id: data.id,
          name: data.name,
          description: data.description,
          price: parseFloat(data.price),
          category: data.category,
          brand: data.brand,
          in_stock: data.inventory_count > 0,
          rating: data.rating || 0,
          updated_at: data.updated_at
        },
        refresh: false
      });
    }
  }
  
  async handleSyncError(event, error) {
    // Store failed events for retry
    await this.db.query(`
      INSERT INTO sync_errors (entity, entity_id, event_data, error, created_at)
      VALUES ($1, $2, $3, $4, NOW())
    `, ['product', event.id, JSON.stringify(event), error.message]);
  }
}

// Approach 3: CDC with Debezium (Production-grade)

/*
// Debezium connector configuration (JSON)
{
  "name": "postgres-products-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres-host",
    "database.port": "5432",
    "database.user": "debezium",
    "database.password": "secret",
    "database.dbname": "mydb",
    "database.server.name": "postgres",
    "table.include.list": "public.products",
    "plugin.name": "pgoutput",
    "publication.name": "debezium_publication",
    "slot.name": "debezium_slot"
  }
}
*/

// Consumer for Debezium CDC events
class DebeziumElasticsearchSync {
  constructor(kafka, esClient) {
    this.kafka = kafka;
    this.es = esClient;
    this.consumer = kafka.consumer({ groupId: 'debezium-es-sync' });
  }
  
  async start() {
    await this.consumer.connect();
    await this.consumer.subscribe({ 
      topic: 'postgres.public.products'  // Debezium topic format
    });
    
    await this.consumer.run({
      eachBatch: async ({ batch }) => {
        // Process in batches for better performance
        const operations = [];
        
        for (const message of batch.messages) {
          const event = JSON.parse(message.value.toString());
          operations.push(this.convertToEsOperation(event));
        }
        
        // Bulk sync to Elasticsearch
        const body = operations.flat();
        const result = await this.es.bulk({ body, refresh: false });
        
        if (result.errors) {
          console.error('Bulk sync errors:', result.items.filter(i => i.error));
        }
      }
    });
  }
  
  convertToEsOperation(cdcEvent) {
    const { op, before, after } = cdcEvent.payload;
    
    // op: 'c' (create), 'u' (update), 'd' (delete), 'r' (read/snapshot)
    if (op === 'd') {
      return [
        { delete: { _index: 'products', _id: before.id } }
      ];
    } else {
      const data = after || before;
      return [
        { index: { _index: 'products', _id: data.id } },
        {
          id: data.id,
          name: data.name,
          description: data.description,
          price: parseFloat(data.price),
          category: data.category,
          updated_at: data.updated_at
        }
      ];
    }
  }
}

// Full reindex capability (recovery mechanism)
class FullReindexService {
  constructor(db, esClient) {
    this.db = db;
    this.es = esClient;
  }
  
  async reindex() {
    console.log('Starting full reindex...');
    
    // Create new index with timestamp
    const newIndex = `products_${Date.now()}`;
    await this.es.indices.create({
      index: newIndex,
      body: productIndexMapping  // Reuse mapping from earlier
    });
    
    // Bulk load all products
    let offset = 0;
    const batchSize = 5000;
    let totalIndexed = 0;
    
    while (true) {
      const products = await this.db.query(`
        SELECT * FROM products 
        ORDER BY id 
        LIMIT $1 OFFSET $2
      `, [batchSize, offset]);
      
      if (products.rows.length === 0) break;
      
      const body = products.rows.flatMap(p => [
        { index: { _index: newIndex, _id: p.id } },
        this.transformProduct(p)
      ]);
      
      await this.es.bulk({ body, refresh: false });
      
      offset += batchSize;
      totalIndexed += products.rows.length;
      console.log(`Indexed ${totalIndexed} products`);
    }
    
    // Refresh new index
    await this.es.indices.refresh({ index: newIndex });
    
    // Swap alias atomically
    await this.es.indices.updateAliases({
      body: {
        actions: [
          { remove: { index: 'products_*', alias: 'products' } },
          { add: { index: newIndex, alias: 'products' } }
        ]
      }
    });
    
    console.log(`Reindex complete. Total: ${totalIndexed} products`);
    
    // Delete old indices (keep last 2 for rollback)
    await this.cleanupOldIndices();
  }
  
  transformProduct(product) {
    return {
      id: product.id,
      name: product.name,
      description: product.description,
      price: parseFloat(product.price),
      category: product.category,
      brand: product.brand,
      in_stock: product.inventory_count > 0,
      rating: product.rating || 0,
      reviews_count: product.reviews_count || 0,
      created_at: product.created_at,
      updated_at: product.updated_at
    };
  }
}
```

### Best Practices

- **Choose sync strategy based on requirements**: Scheduled batch for eventual consistency, CDC/Kafka for near real-time. Don't over-engineer for low-volume use cases.
- **Implement full reindex**: Always have ability to rebuild Elasticsearch from PostgreSQL. Use for recovery and major schema changes.
- **Handle failures gracefully**: Implement dead letter queues for failed syncs. Monitor and retry failed operations.
- **Use bulk operations**: Single document operations are slow. Batch 1000-5000 documents per bulk request for optimal performance.
- **Monitor sync lag**: Track time difference between PostgreSQL updates and Elasticsearch index. Alert if lag exceeds threshold.
- **PostgreSQL is source of truth**: Never write directly to Elasticsearch. All writes go through PostgreSQL to maintain consistency.
- **Test failover scenarios**: Ensure system handles Elasticsearch downtime. Queue updates and replay when service recovers.

---

## Question 4: How would you handle relevance scoring and custom ranking in search results?

### Conceptual Explanation

Relevance scoring determines how well documents match a query and in what order results appear. Elasticsearch uses **BM25 algorithm** by default, which scores based on term frequency, inverse document frequency, and field length. Custom ranking overlays business logic on top: boosting popular products, promoting in-stock items, or personalizing based on user behavior.

Techniques include: **field boosting** (name^3 gives name 3x weight), **function_score** (modify scores with functions), **script_score** (custom JavaScript-like scoring), **boosting query** (demote rather than exclude), and **rescore** (re-rank top N results with expensive computation). Balance relevance (text match quality) with business goals (revenue, user satisfaction).

Key considerations: avoid over-optimization (keyword stuffing), A/B test ranking changes, monitor metrics (CTR, conversion rate, null search rate), and iterate based on user behavior analytics.

### Code Example

```javascript
// Basic field boosting
const basicBoostQuery = {
  query: {
    multi_match: {
      query: 'wireless headphones',
      fields: [
        'name^3',        // Name is most important
        'brand^2',       // Brand somewhat important
        'description',   // Description baseline weight
        'category'
      ]
    }
  }
};

// Function score for business logic
const functionScoreQuery = {
  query: {
    function_score: {
      query: {
        multi_match: {
          query: 'laptop',
          fields: ['name^3', 'description']
        }
      },
      functions: [
        // Boost by popularity/sales
        {
          field_value_factor: {
            field: 'sales_count',
            factor: 1.2,
            modifier: 'log1p',  // log(1 + value)
            missing: 1
          }
        },
        // Boost by rating
        {
          field_value_factor: {
            field: 'rating',
            factor: 0.5,
            modifier: 'sqrt',
            missing: 1
          }
        },
        // Boost in-stock items
        {
          filter: { term: { in_stock: true } },
          weight: 1.5
        },
        // Boost recent products
        {
          gauss: {
            created_at: {
              origin: 'now',
              scale: '30d',    // Products from last 30 days
              decay: 0.5       // Decay rate
            }
          }
        },
        // Boost by price range (promote mid-range)
        {
          gauss: {
            price: {
              origin: 500,     // Optimal price point
              scale: 200,      // Range around origin
              decay: 0.5
            }
          }
        }
      ],
      score_mode: 'sum',       // How to combine function scores
      boost_mode: 'multiply',  // How to combine with query score
      max_boost: 10,           // Cap the boost
      min_score: 1             // Filter out low scores
    }
  }
};

// Personalized ranking based on user preferences
class PersonalizedSearchService {
  async search(userId, query, filters = {}) {
    // Get user's preferred categories from history
    const userPrefs = await this.getUserPreferences(userId);
    
    const searchQuery = {
      query: {
        function_score: {
          query: {
            bool: {
              must: [
                {
                  multi_match: {
                    query: query,
                    fields: ['name^3', 'description'],
                    fuzziness: 'AUTO'
                  }
                }
              ],
              filter: filters
            }
          },
          functions: [
            // Boost user's preferred categories
            ...userPrefs.categories.map(cat => ({
              filter: { term: { category: cat } },
              weight: 2.0
            })),
            // Boost user's preferred brands
            ...userPrefs.brands.map(brand => ({
              filter: { term: { brand: brand } },
              weight: 1.5
            })),
            // Boost products similar to past purchases
            {
              more_like_this: {
                fields: ['name', 'description', 'category'],
                like: userPrefs.purchasedProductIds.map(id => ({
                  _index: 'products',
                  _id: id
                })),
                min_term_freq: 1,
                max_query_terms: 12
              }
            },
            // General popularity
            {
              field_value_factor: {
                field: 'popularity_score',
                modifier: 'log1p',
                factor: 1.0
              }
            }
          ],
          score_mode: 'sum',
          boost_mode: 'multiply'
        }
      },
      // Re-score top 100 with more expensive calculation
      rescore: {
        window_size: 100,
        query: {
          rescore_query: {
            script_score: {
              script: {
                source: `
                  double baseScore = _score;
                  double recencyBoost = 1.0;
                  
                  // Boost very recent products more
                  long daysSinceCreated = (new Date().getTime() - doc['created_at'].value.getMillis()) / 86400000;
                  if (daysSinceCreated < 7) {
                    recencyBoost = 1.5;
                  }
                  
                  // Boost based on conversion rate
                  double conversionRate = doc['orders_count'].value / Math.max(doc['views_count'].value, 1.0);
                  double conversionBoost = 1.0 + (conversionRate * 2.0);
                  
                  return baseScore * recencyBoost * conversionBoost;
                `
              }
            }
          },
          query_weight: 0.7,
          rescore_query_weight: 0.3
        }
      }
    };
    
    return await this.es.search({
      index: 'products',
      body: searchQuery
    });
  }
  
  async getUserPreferences(userId) {
    // Fetch from cache or database
    const cached = await redis.get(`user:prefs:${userId}`);
    if (cached) return JSON.parse(cached);
    
    // Aggregate user's past behavior
    const prefs = await this.db.query(`
      SELECT 
        ARRAY_AGG(DISTINCT p.category) as categories,
        ARRAY_AGG(DISTINCT p.brand) as brands,
        ARRAY_AGG(DISTINCT o.product_id) as purchased_product_ids
      FROM orders o
      JOIN products p ON o.product_id = p.id
      WHERE o.user_id = $1
        AND o.created_at > NOW() - INTERVAL '90 days'
    `, [userId]);
    
    const preferences = {
      categories: prefs.rows[0].categories || [],
      brands: prefs.rows[0].brands || [],
      purchasedProductIds: prefs.rows[0].purchased_product_ids || []
    };
    
    await redis.setex(`user:prefs:${userId}`, 3600, JSON.stringify(preferences));
    return preferences;
  }
}

// Learning to Rank (LTR) with click-through data
class LTRSearchService {
  async search(query, filters = {}) {
    // Get initial results
    const initialResults = await this.es.search({
      index: 'products',
      body: {
        query: {
          multi_match: {
            query: query,
            fields: ['name^3', 'description']
          }
        },
        size: 100
      }
    });
    
    // Extract features for each result
    const rankedResults = await Promise.all(
      initialResults.hits.hits.map(async hit => {
        const features = await this.extractFeatures(hit, query);
        const ltrScore = this.calculateLTRScore(features);
        
        return {
          ...hit._source,
          originalScore: hit._score,
          ltrScore: ltrScore,
          finalScore: hit._score * 0.3 + ltrScore * 0.7  // Blend scores
        };
      })
    );
    
    // Sort by final score
    rankedResults.sort((a, b) => b.finalScore - a.finalScore);
    
    return rankedResults;
  }
  
  extractFeatures(doc, query) {
    // Features for ML model
    return {
      textRelevance: doc._score,
      rating: doc._source.rating || 0,
      reviewsCount: doc._source.reviews_count || 0,
      price: doc._source.price,
      inStock: doc._source.in_stock ? 1 : 0,
      salesCount: doc._source.sales_count || 0,
      ctr: doc._source.click_through_rate || 0,
      conversionRate: doc._source.conversion_rate || 0,
      queryProductMatch: this.calculateQueryMatch(query, doc._source.name),
      recency: this.calculateRecency(doc._source.created_at)
    };
  }
  
  calculateLTRScore(features) {
    // Trained model weights (from ML training on click data)
    const weights = {
      textRelevance: 0.3,
      rating: 0.15,
      reviewsCount: 0.05,
      inStock: 0.1,
      salesCount: 0.1,
      ctr: 0.15,
      conversionRate: 0.15
    };
    
    let score = 0;
    for (const [feature, weight] of Object.entries(weights)) {
      score += (features[feature] || 0) * weight;
    }
    
    return score;
  }
  
  calculateQueryMatch(query, productName) {
    // Simple Jaccard similarity
    const queryTokens = new Set(query.toLowerCase().split(' '));
    const nameTokens = new Set(productName.toLowerCase().split(' '));
    
    const intersection = new Set([...queryTokens].filter(t => nameTokens.has(t)));
    const union = new Set([...queryTokens, ...nameTokens]);
    
    return intersection.size / union.size;
  }
  
  calculateRecency(createdAt) {
    const daysSince = (Date.now() - new Date(createdAt).getTime()) / (1000 * 60 * 60 * 24);
    return Math.max(0, 1 - (daysSince / 365));  // Decay over a year
  }
}

// Boosting query (demote, don't exclude)
const boostingQuery = {
  query: {
    boosting: {
      positive: {
        match: {
          description: 'laptop'
        }
      },
      negative: {
        term: {
          category: 'refurbished'  // Demote refurbished products
        }
      },
      negative_boost: 0.3  // Multiply score by 0.3 for negative matches
    }
  }
};

// A/B testing different ranking strategies
class ABTestingService {
  async search(userId, query, filters = {}) {
    // Assign user to variant
    const variant = this.getUserVariant(userId);
    
    let results;
    if (variant === 'control') {
      results = await this.controlRanking(query, filters);
    } else if (variant === 'variant_a') {
      results = await this.variantARanking(query, filters);
    } else {
      results = await this.variantBRanking(query, filters);
    }
    
    // Log for analysis
    await this.logSearchEvent({
      userId,
      query,
      variant,
      resultCount: results.length
    });
    
    return results;
  }
  
  getUserVariant(userId) {
    // Consistent hashing for stable assignment
    const hash = this.hashString(userId);
    const bucket = hash % 100;
    
    if (bucket < 33) return 'control';
    if (bucket < 66) return 'variant_a';
    return 'variant_b';
  }
}
```

### Best Practices

- **Start simple, iterate**: Begin with basic text relevance and field boosting. Add complexity based on metrics showing need for improvement.
- **Use function_score for business logic**: Combine text relevance with popularity, ratings, inventory status. Test different scoring functions and weights.
- **A/B test ranking changes**: Never deploy ranking changes without testing impact on CTR, conversion rate, and revenue. What seems logical may hurt metrics.
- **Monitor search quality metrics**: Track CTR (click-through rate), zero result rate, abandonment rate, and conversion rate per query.
- **Implement learning to rank**: For mature systems, use ML models trained on click/conversion data. Requires significant data and infrastructure.
- **Balance relevance and business goals**: Don't over-promote low-quality products for short-term revenue. User trust depends on relevant results.
- **Use rescore for expensive calculations**: Apply complex scoring only to top N results. Saves computation on full result set.
- **Document scoring decisions**: Keep ADRs explaining why certain boosts/weights were chosen. Helps with future tuning and debugging.

---

## Question 5: What's your approach to scaling and managing large Elasticsearch clusters?

### Conceptual Explanation

Scaling Elasticsearch involves **horizontal scaling** (add more nodes), **proper sharding** (distribute data across nodes), **index lifecycle management** (move old data to cheaper storage), and **resource optimization** (memory, CPU, disk tuning). Large clusters face challenges: shard management, cluster health, query performance, and operational complexity.

Key concepts: **shards** are the unit of scaling (primary + replicas), **nodes** have roles (master, data, ingest, coordinating), **indices** should have appropriate shard count (5-50GB per shard is optimal), and **ILM** automates data retention (hot/warm/cold/delete phases). Monitor cluster health (green/yellow/red), search/indexing rates, and resource utilization.

For very large deployments (100+ nodes, PB+ data): use dedicated node roles, separate hot/warm/cold architectures, implement curator for old data cleanup, and employ cluster-level configurations for stability.

### Code Example

```javascript
// Optimal index settings for scale
const scalableIndexSettings = {
  settings: {
    number_of_shards: 5,  // Based on data volume and query load
    number_of_replicas: 1, // One replica for HA
    refresh_interval: '30s', // Reduce refresh frequency (default 1s)
    
    // Memory optimization
    index: {
      codec: 'best_compression', // Trade CPU for disk space
      max_result_window: 10000,  // Limit deep pagination
    },
    
    // Indexing performance
    index: {
      translog: {
        durability: 'async',  // Faster indexing, slight risk
        sync_interval: '30s',
        flush_threshold_size: '1gb'
      }
    },
    
    // Merge policy for write-heavy indices
    merge: {
      policy: {
        max_merged_segment: '5gb',
        segments_per_tier: 10
      }
    }
  },
  mappings: {
    // ... field mappings
  }
};

// Index Lifecycle Management (ILM) policy
const ilmPolicy = {
  policy: {
    phases: {
      hot: {
        min_age: '0ms',
        actions: {
          rollover: {
            max_size: '50gb',
            max_age: '7d',
            max_docs: 50000000
          },
          set_priority: {
            priority: 100
          }
        }
      },
      warm: {
        min_age: '7d',
        actions: {
          shrink: {
            number_of_shards: 1  // Reduce shards for old data
          },
          forcemerge: {
            max_num_segments: 1  // Optimize for read
          },
          allocate: {
            require: {
              data: 'warm'  // Move to warm nodes
            }
          },
          set_priority: {
            priority: 50
          }
        }
      },
      cold: {
        min_age: '30d',
        actions: {
          allocate: {
            require: {
              data: 'cold'
            }
          },
          freeze: {},  // Reduce memory footprint
          set_priority: {
            priority: 0
          }
        }
      },
      delete: {
        min_age: '90d',
        actions: {
          delete: {}
        }
      }
    }
  }
};

// Create ILM policy
await client.ilm.putLifecycle({
  policy: 'products-policy',
  body: ilmPolicy
});

// Create index template with ILM
await client.indices.putIndexTemplate({
  name: 'products-template',
  body: {
    index_patterns: ['products-*'],
    template: {
      settings: {
        'index.lifecycle.name': 'products-policy',
        'index.lifecycle.rollover_alias': 'products',
        ...scalableIndexSettings.settings
      },
      mappings: scalableIndexSettings.mappings
    }
  }
});

// Node roles in large cluster
/*
# Master nodes (cluster management)
node.roles: [ master ]
node.master: true
node.data: false
node.ingest: false

# Hot data nodes (recent data, high performance)
node.roles: [ data_hot, data_content ]
node.attr.data: hot

# Warm data nodes (older data, moderate performance)
node.roles: [ data_warm ]
node.attr.data: warm

# Cold data nodes (old data, cheap storage)
node.roles: [ data_cold ]
node.attr.data: cold

# Coordinating nodes (query routing, no data)
node.roles: []

# Ingest nodes (preprocessing)
node.roles: [ ingest ]
*/

// Cluster management service
class ElasticsearchClusterManager {
  constructor(client) {
    this.client = client;
  }
  
  async getClusterHealth() {
    const health = await this.client.cluster.health();
    
    return {
      status: health.status,  // green, yellow, red
      nodes: health.number_of_nodes,
      dataNodes: health.number_of_data_nodes,
      activeShards: health.active_shards,
      relocatingShards: health.relocating_shards,
      initializingShards: health.initializing_shards,
      unassignedShards: health.unassigned_shards,
      activeShardsPercent: health.active_shards_percent_as_number
    };
  }
  
  async getClusterStats() {
    const stats = await this.client.cluster.stats();
    
    return {
      indices: {
        count: stats.indices.count,
        docsCount: stats.indices.docs.count,
        storeSize: stats.indices.store.size_in_bytes,
        shards: stats.indices.shards.total
      },
      nodes: {
        count: stats.nodes.count.total,
        memory: {
          total: stats.nodes.os.mem.total_in_bytes,
          free: stats.nodes.os.mem.free_in_bytes,
          used: stats.nodes.os.mem.used_in_bytes
        },
        jvm: {
          heapUsed: stats.nodes.jvm.mem.heap_used_in_bytes,
          heapMax: stats.nodes.jvm.mem.heap_max_in_bytes
        }
      }
    };
  }
  
  async getHotThreads() {
    // Identify performance bottlenecks
    const threads = await this.client.nodes.hotThreads({
      threads: 10,
      type: 'cpu'
    });
    return threads;
  }
  
  async rebalanceShards() {
    // Enable/disable shard allocation
    await this.client.cluster.putSettings({
      body: {
        transient: {
          'cluster.routing.allocation.enable': 'all'
        }
      }
    });
  }
  
  async getSlowQueries() {
    // Analyze slow query log
    const response = await this.client.search({
      index: '.monitoring-es-*',
      body: {
        query: {
          range: {
            'search.took_time': { gte: 1000 }  // > 1 second
          }
        },
        size: 100,
        sort: [{ 'search.took_time': 'desc' }]
      }
    });
    
    return response.hits.hits;
  }
  
  async optimizeIndices() {
    // Force merge old indices (read-only)
    const indices = await this.getOldIndices();
    
    for (const index of indices) {
      console.log(`Force merging ${index}...`);
      
      await this.client.indices.forcemerge({
        index: index,
        max_num_segments: 1,
        wait_for_completion: false
      });
    }
  }
  
  async getOldIndices() {
    const response = await this.client.cat.indices({
      format: 'json',
      h: 'index,creation.date'
    });
    
    const thirtyDaysAgo = Date.now() - (30 * 24 * 60 * 60 * 1000);
    
    return response
      .filter(idx => parseInt(idx['creation.date']) < thirtyDaysAgo)
      .map(idx => idx.index);
  }
  
  async monitorSearchPerformance() {
    const stats = await this.client.nodes.stats({
      metric: ['indices'],
      index_metric: ['search']
    });
    
    for (const [nodeId, nodeStats] of Object.entries(stats.nodes)) {
      const search = nodeStats.indices.search;
      
      console.log(`Node ${nodeId}:`);
      console.log(`  Search total: ${search.query_total}`);
      console.log(`  Search time: ${search.query_time_in_millis}ms`);
      console.log(`  Avg query time: ${search.query_time_in_millis / search.query_total}ms`);
      console.log(`  Fetch total: ${search.fetch_total}`);
      console.log(`  Fetch time: ${search.fetch_time_in_millis}ms`);
    }
  }
}

// Shard allocation awareness (for multi-AZ deployment)
const multiAZSettings = {
  'cluster.routing.allocation.awareness.attributes': 'zone',
  'cluster.routing.allocation.awareness.force.zone.values': 'us-east-1a,us-east-1b,us-east-1c'
};

// Circuit breaker settings (prevent OOM)
const circuitBreakerSettings = {
  'indices.breaker.total.limit': '70%',  // Total memory for all breakers
  'indices.breaker.request.limit': '40%', // Request circuit breaker
  'indices.breaker.fielddata.limit': '40%' // Fielddata circuit breaker
};

// Monitoring with Prometheus
class ElasticsearchMetricsExporter {
  async getMetrics() {
    const health = await this.client.cluster.health();
    const stats = await this.client.cluster.stats();
    
    return {
      // Cluster health
      'es_cluster_status': health.status === 'green' ? 0 : health.status === 'yellow' ? 1 : 2,
      'es_cluster_nodes': health.number_of_nodes,
      'es_cluster_shards_active': health.active_shards,
      'es_cluster_shards_unassigned': health.unassigned_shards,
      
      // Performance
      'es_indices_count': stats.indices.count,
      'es_documents_count': stats.indices.docs.count,
      'es_store_size_bytes': stats.indices.store.size_in_bytes,
      
      // Resources
      'es_jvm_heap_used_percent': (stats.nodes.jvm.mem.heap_used_in_bytes / stats.nodes.jvm.mem.heap_max_in_bytes) * 100,
      'es_jvm_gc_time_ms': stats.nodes.jvm.gc.collectors.young.collection_time_in_millis
    };
  }
}

// Backup and restore
class ElasticsearchBackupService {
  async createSnapshot(repository, snapshotName) {
    await this.client.snapshot.create({
      repository: repository,
      snapshot: snapshotName,
      body: {
        indices: 'products-*',
        ignore_unavailable: true,
        include_global_state: false
      },
      wait_for_completion: false
    });
    
    console.log(`Snapshot ${snapshotName} started`);
  }
  
  async restoreSnapshot(repository, snapshotName) {
    await this.client.snapshot.restore({
      repository: repository,
      snapshot: snapshotName,
      body: {
        indices: 'products-*',
        ignore_unavailable: true,
        include_global_state: false,
        rename_pattern: 'products-(.*)',
        rename_replacement: 'restored-products-$1'
      }
    });
    
    console.log(`Restore from ${snapshotName} started`);
  }
}
```

### Best Practices

- **Right-size shards**: Aim for 20-50GB per shard. Too many small shards hurt performance, too few large shards limit scaling. Adjust based on workload.
- **Use dedicated master nodes**: 3 master-eligible nodes prevent split-brain. They don't store data, just manage cluster state.
- **Implement ILM**: Automatically move old data through hot→warm→cold→delete phases. Saves significant infrastructure costs.
- **Monitor heap usage**: Keep JVM heap under 50% usage. Set Xms and Xmx to same value (typically 50% of RAM, max 31GB).
- **Separate node roles**: In large clusters, dedicate nodes for specific roles (master, hot data, warm data, coordinating). Improves stability.
- **Use index templates**: Define consistent settings and mappings for all new indices. Easier management at scale.
- **Regular snapshots**: Daily snapshots to S3/GCS. Test restore procedures regularly. Elasticsearch is not a primary data store.
- **Monitor and alert**: Track cluster status, shard allocation, query performance, and resource usage. Alert on yellow/red status immediately.

