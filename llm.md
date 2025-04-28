# UNBODY DOCUMENTATION

**Current Version: 0.0.14**

## COMMON ERROR CODES

- `401`: Authentication failed - check API keys
- `403`: Permission denied - insufficient access rights
- `404`: Resource not found - check IDs and paths
- `422`: Validation error - check request parameters
- `429`: Rate limit exceeded - implement backoff strategy
- `500`: Server error - retry with exponential backoff

## ADMIN API

### Authentication

```typescript
// Admin Keys (Basic Auth)
const admin = new UnbodyAdmin({
  auth: {
    username: "[admin-key-id]",
    password: "[admin-key-secret]",
  },
});

// Access Token (Bearer)
const admin = new UnbodyAdmin({
  auth: {
    accessToken: "Bearer [your-access-token]",
  },
});
```

### Creating and Managing Projects

```typescript
// Create Project
interface ProjectResponse {
  id: string;
  name: string;
  createdAt: string;
  state: string;
}

// Create Project
const settings = new ProjectSettings().set(
  new TextVectorizer(TextVectorizer.Transformers.Default),
);
const project = admin.projects.ref({
  name: "New Project",
  settings,
});

try {
  const savedProject: ProjectResponse = await project.save();
  console.log(`Project created with ID: ${savedProject.id}`);
} catch (error) {
  console.error("Failed to create project:", error.message);
}

// List Projects
interface ProjectListResponse {
  projects: ProjectResponse[];
  pagination: {
    count: number;
    total: number;
    offset: number;
    limit: number;
  };
}

const { projects, pagination }: ProjectListResponse = await admin.projects.list(
  {
    limit: 10,
    offset: 0,
    sort: { field: "createdAt", order: "desc" },
    filter: { state: "initialized" },
  },
);

// Delete Project
await admin.projects.delete({ id: "[project-id]" });
```

### Managing Data Sources

```typescript
// Create Source
interface SourceResponse {
  id: string;
  name: string;
  type: string;
  state: string;
  createdAt: string;
}

const source = project.sources.ref({
  name: "[source name]",
  type: SourceTypes.PushApi, // Enum value, not string
});

try {
  const savedSource: SourceResponse = await source.save();
  console.log(`Source created with ID: ${savedSource.id}`);
} catch (error) {
  if (error.status === 422) {
    console.error("Invalid source configuration:", error.message);
  }
}

// List Sources
interface SourceListResponse {
  sources: SourceResponse[];
  pagination: {
    count: number;
    total: number;
    offset: number;
    limit: number;
  };
}

const { sources, pagination }: SourceListResponse = await project.sources.list({
  limit: 10,
  filter: { type: SourceTypes.GoogleDrive },
});

// Initialize/Rebuild Source
try {
  await project.sources.initialize({ id: "[source-id]" });
  console.log("Source initialization started");
} catch (error) {
  console.error("Failed to initialize source:", error.message);
}

await project.sources.rebuild({ id: "[source-id]" });

// See also: Data Ingestion section for working with initialized sources
```

### Project Settings

Available settings include:

- TextVectorizer: Converts text to vectors (see Vectorizers section)
- ImageVectorizer: Converts images to vectors
- QnA: Configures question-answering capabilities
- Generative: Enables RAG capabilities
- Reranker: Reorders search results for relevance
- Spellcheck: Configures spellchecking for search
- AutoSummary: Generates summaries automatically
- AutoKeywords: Extracts keywords from content
- AutoEntities: Identifies named entities
- AutoTopics: Extracts topics from content
- AutoVision: Analyzes images and generates captions
- PdfParser: Configures PDF processing
- CustomSchema: Defines custom data structures (see Custom Schemas section)
- Enhancement: Configures enhancement pipelines (see Enhancement Pipelines section)

## CONTENT API

### Document Types and Query Types

Unbody organizes content into various document types, each with specific structure and capabilities:

- **TextDocument** - General text documents with rich content
- **ImageBlock** - Image content with metadata
- **AudioFile** - Audio content with metadata
- **VideoFile** - Video content with metadata
- **TextBlock** - Blocks of text within larger documents
- **GoogleDoc**, **GoogleCalendarEvent** - Google Workspace content
- **DiscordMessage** - Messages from Discord
- **GithubThread**, **GithubComment** - GitHub content
- **SpreadsheetDocument** - Documents from spreadsheets
- **Website**, **WebPage** - Web content

There are two main types of queries:

- **Get** - Retrieve instances of documents
- **Aggregate** - Get aggregate information and statistics about documents

### Authentication

```typescript
// API client setup
import { Unbody } from "unbody";

const unbody = new Unbody({
  apiKey: "YOUR_API_KEY", // From project settings
  projectId: "YOUR_PROJECT_ID",
  // Optional: custom base URL
  // baseUrl: "https://custom-api.unbody.io",
  // Optional: data transformers
  // transformers: {
  //   TextDocument: {
  //     customData: (data: string) => JSON.parse(data)
  //   }
  // }
});

// For raw HTTP requests:
// Include in request headers
// Authorization: Your API key
// X-Project-Id: Your project ID
```

### Basic Data Retrieval

```typescript
// Using SDK
interface QueryResponse<T> {
  data: {
    payload: T[];
    meta?: {
      count: number;
      total: number;
    };
    errors?: any[];
  };
}

// Get a list of documents
try {
  const { data }: QueryResponse<TextDocument> = await unbody.get
    .collection("TextDocument")
    .select("id", "title", "content")
    .exec();

  console.log(`Retrieved ${data.payload.length} documents`);

  // Process results
  data.payload.forEach((doc) => {
    console.log(`${doc.id}: ${doc.title}`);
  });
} catch (error) {
  console.error("Failed to fetch documents:", error.message);
}

// Working with nested fields
// To access properties of nested objects, always use dot notation
const { data } = await unbody.get
  .collection("Product")
  .select(
    "id",
    "name",
    "price",
    "details.manufacturer", // Accessing a nested property
    "details.dimensions.width", // Accessing a deeply nested property
    "details.dimensions.height",
    "features.0.name", // Accessing array elements by index
    "variants.color", // Accessing property of objects in an array
    "metadata.created.timestamp", // Multiple levels of nesting
  )
  .exec();

// Common pattern for nested object access
// Vehicle with nested fuelEconomy object
const { data } = await unbody.get
  .collection("Car")
  .select(
    "id",
    "make",
    "model",
    "year",
    "fuelEconomy.city", // Select specific nested fields
    "fuelEconomy.highway", // Not "fuelEconomy" by itself
  )
  .exec();

// Type definition for proper type inference with nested objects
interface Car {
  id: string;
  make: string;
  model: string;
  // Define nested structure
  fuelEconomy: {
    city: number;
    highway: number;
  };
}

// Access with correct typing
const cars = data.payload as Car[];
console.log(`City MPG: ${cars[0].fuelEconomy.city}`);

// Using GraphQL
const query = `
query {
  Get {
    TextDocument {
      id
      title
      content
    }
  }
}
`;
```

### Searching for Content

#### Semantic Search

```typescript
// Define enhanced type for search results
interface SearchDocument extends TextDocument {
  _additional?: {
    certainty?: number;
    distance?: number;
    score?: number;
  };
}

// Find documents about a concept with proper typing
const { data }: QueryResponse<SearchDocument> =
  await unbody.get.textDocument.search
    .about("artificial intelligence")
    .select("id", "title", "text") // Select only needed fields
    .exec();

// With certainty threshold
const { data } = await unbody.get.textDocument.search
  .about("artificial intelligence", {
    certainty: 0.7, // Higher means more strict matching
  })
  .exec();

// Accessing certainty scores with type safety and optional chaining
const documents = data.payload;
const sortedByRelevance = [...documents].sort(
  (a, b) => (b._additional?.certainty ?? 0) - (a._additional?.certainty ?? 0),
);

// Response handling
if (documents.length === 0) {
  console.log("No relevant documents found");
} else {
  console.log(`Found ${documents.length} relevant documents`);
  console.log(
    `Top match: ${sortedByRelevance[0].title} with ${sortedByRelevance[0]._additional?.certainty?.toFixed(2) ?? "unknown"} certainty`,
  );
}
```

#### Keyword Search

```typescript
// Find exact keyword matches
const { data }: QueryResponse<TextDocument> =
  await unbody.get.textDocument.search
    .match("machine learning")
    .select("id", "title", "text")
    .exec();

// Search in specific properties
const { data } = await unbody.get.textDocument.search
  .match("machine learning", {
    properties: ["title", "summary"],
  })
  .exec();
```

#### Hybrid Search

```typescript
// Combine semantic and keyword matching
const { data }: QueryResponse<TextDocument> =
  await unbody.get.textDocument.search
    .find("AI in healthcare", {
      alpha: 0.7, // Balance between semantic (1.0) and keyword (0.0)
      fusionType: "rankedFusion",
    })
    .exec();

// With boosted properties
const { data } = await unbody.get.textDocument.search
  .find("AI in healthcare", {
    properties: {
      title: 2.0, // Boost title matches
      text: 1.0,
    },
  })
  .exec();
```

#### Vector Search

```typescript
// Search using direct vector input
const { data } = await unbody.get
  .textDocument
  .nearVector({
    vector: [0.1, 0.2, 0.3, ...], // Your normalized vector
    distance: 0.8 // Maximum distance threshold
  })
  .exec();

// Search with a vector from another record
const { data } = await unbody.get
  .textDocument
  .nearVector({
    id: "document-123", // Use vector from this document
    property: "text", // Optional: specific field to use
    distance: 0.3 // Lower means more similar
  })
  .exec();
```

#### Record Similarity

```typescript
// Find similar records
const { data }: QueryResponse<TextDocument> =
  await unbody.get.textDocument.search
    .similar({ id: "document-123" })
    .select("id", "title", "text")
    .exec();

// With distance threshold
const { data } = await unbody.get.textDocument.search
  .similar({
    id: "document-123",
    distance: 0.3, // Lower means more similar
  })
  .exec();
```

#### Visual Similarity

```typescript
// Find similar images
const { data }: QueryResponse<ImageBlock> = await unbody.get.imageBlock.search
  .similar({ id: "image-123" })
  .select("id", "url", "caption")
  .exec();

// Process results
const similarImages = data.payload;
similarImages.forEach((img) => {
  console.log(`Similar image: ${img.url}`);
});
```

### Filtering Results

```typescript
// Basic filtering
const { data }: QueryResponse<TextDocument> = await unbody.get.textDocument
  .where({
    title: "My Document",
    createdAt: { $gt: new Date("2023-01-01") },
  })
  .select("id", "title", "text")
  .exec();

// Complex conditions
const { data } = await unbody.get.textDocument
  .where(({ Equal, And, GreaterThan }) =>
    And({ title: Equal("Report") }, { size: GreaterThan(100) }),
  )
  .exec();

// Combining search and filters
const { data } = await unbody.get.textDocument.search
  .about("climate change")
  .where({
    createdAt: { $gt: new Date("2023-01-01") },
  })
  .exec();
```

### Pagination and Sorting

```typescript
// Paginated results
const { data }: QueryResponse<TextDocument> = await unbody.get.textDocument
  .limit(10)
  .offset(20) // Skip first 20 results
  .exec();

// Handling pagination metadata
console.log(`Showing ${data.payload.length} of ${data.meta.total} results`);

// Sorting results
const { data } = await unbody.get.textDocument
  .sort("createdAt", "desc")
  .limit(10)
  .exec();

// Multiple sort criteria
const { data } = await unbody.get.textDocument
  .sort([
    { field: "createdAt", order: "desc" },
    { field: "title", order: "asc" },
  ])
  .exec();
```

### Advanced Query Features

#### Group By

```typescript
// Group results by a field
const { data } = await unbody.get.textDocument
  .groupBy("tags", 5) // Group by tags field, up to 5 groups
  .exec();
```

#### Reranking

```typescript
// Rerank search results for better relevance
const { data } = await unbody.get.textDocument.search
  .about("quantum computing")
  .rerank("advanced quantum algorithms", "text")
  .exec();
```

#### Spell Check

```typescript
// Enable spell checking for search queries
const { data } = await unbody.get.textDocument
  .nearText({
    concepts: ["quantom computing"], // Misspelled
    autocorrect: true,
  })
  .spellCheck()
  .exec();

// Get spelling corrections
const corrections = data.payload.map(
  (doc) => doc._additional?.spellCheck?.changes,
);
```

### Generative Search

#### Generate from One

`generate.fromOne()` generates content for each individual search result. In Unbody version 0.0.14, this method accepts either a string prompt or a configuration object, but not a combination of both.

**Method Signatures:**

```typescript
// Simple string prompt
generate.fromOne(prompt: string): Query

// With prompt and options
generate.fromOne(config: {
  prompt: string,
  options?: {
    temperature?: number,
    maxTokens?: number,
    model?: string,
    // other LLM options
  }
}): Query

// With messages format
generate.fromOne(config: {
  messages: Array<{ role: string, content: string }>,
  options?: {
    temperature?: number,
    model?: string,
    // other LLM options
  }
}): Query
```

**Response type:**

```typescript
interface GenerateFromOneResponse<T> {
  data: {
    payload: T[];
    generate: Array<{
      result: string;
      error?: string;
      from: T & {
        _additional?: {
          certainty?: number;
          distance?: number;
        };
      };
      metadata: {
        generationTimeMs?: number;
      };
    }>;
  };
}
```

**Examples:**

```typescript
// Generate content for each result - simple prompt
const { data }: GenerateFromOneResponse<TextDocument> =
  await unbody.get.textDocument.search
    .about("climate change")
    .select("id", "title", "text")
    .generate.fromOne("Summarize the key points from {title}: {text}")
    .exec();

// Process generated results
data.generate.forEach((item) => {
  console.log(`Document: ${item.from.title}`);
  console.log(`Summary: ${item.result}`);
  console.log(`Match score: ${item.from._additional?.certainty || 0}`);
});

// With generation options (using object format)
const { data } = await unbody.get.textDocument.search
  .about("climate change")
  .generate.fromOne({
    prompt: "Summarize {text}",
    options: {
      temperature: 0.7,
      maxTokens: 200,
      model: "gpt-4",
    },
  })
  .exec();

// Using messages format for more control
const { data } = await unbody.get.textDocument.search
  .about("climate change")
  .generate.fromOne({
    messages: [
      { role: "system", content: "You are a scientific summarizer." },
      {
        role: "user",
        content: "Summarize this document in an academic style.",
      },
    ],
    options: { model: "gpt-4" },
  })
  .exec();
```

#### Generate from Many

`generate.fromMany()` creates a single response that incorporates all search results together. In Unbody version 0.0.14, be careful with the parameter format - each method signature shown below has a specific format that must be followed.

**Method Signatures:**

```typescript
// Basic usage with task and properties
generate.fromMany(
  task: string,
  properties: string[],
  options?: {
    temperature?: number,
    maxTokens?: number,
    model?: string
    // other LLM options
  }
): Query

// Object syntax for more control
generate.fromMany(config: {
  task: string,
  properties: string[],
  options?: {
    temperature?: number,
    maxTokens?: number,
    model?: string
    // other LLM options
  }
}): Query

// Messages format
generate.fromMany(config: {
  messages: Array<{ role: string, content: string }>,
  options?: {
    temperature?: number,
    model?: string
    // other LLM options
  }
}): Query
```

**Response type:**

```typescript
interface GenerateFromManyResponse<T> {
  data: {
    payload: T[];
    generate: {
      result: string;
    };
  };
}
```

**Examples:**

```typescript
// Generate content from multiple results - basic usage
const { data }: GenerateFromManyResponse<TextDocument> =
  await unbody.get.textDocument.search
    .about("climate change")
    .limit(5)
    .select("id", "title", "text")
    .generate.fromMany(
      "Create a comprehensive summary of these documents about climate change",
      ["title", "text"],
    )
    .exec();

console.log("Generated summary:", data.generate.result);

// With model specification
const { data } = await unbody.get.textDocument.search
  .about("climate change")
  .generate.fromMany("Summarize these documents", ["text"], {
    model: "gpt-4",
    temperature: 0.5,
    maxTokens: 1000,
  })
  .exec();

// Using a task object with options
const { data } = await unbody.get.textDocument.search
  .about("climate change")
  .generate.fromMany({
    task: "Create an executive summary of all documents",
    properties: ["text", "title"],
    options: {
      temperature: 0.5,
      model: "gpt-4",
      maxTokens: 1000,
    },
  })
  .exec();

// Using messages format for more complex prompting
const { data } = await unbody.get.textDocument.search
  .about("climate change")
  .generate.fromMany({
    messages: [
      {
        role: "system",
        content: "You are creating a cohesive report from multiple documents.",
      },
      {
        role: "user",
        content: "Synthesize these documents into one coherent report.",
      },
    ],
    options: { model: "gpt-4" },
  })
  .exec();
```

### Q&A

```typescript
interface QnAResponse<T> {
  data: {
    payload: T[];
    ask: {
      answer: string;
      sources: Array<{
        object: T;
      }>;
    };
  };
}

// Ask questions about content
const { data }: QnAResponse<TextDocument> = await unbody.get.textDocument.search
  .about("renewable energy")
  .select("id", "title", "text")
  .ask("What are the main challenges for solar power adoption?")
  .exec();

console.log("Answer:", data.ask.answer);
console.log(
  "Sources:",
  data.ask.sources.map((s) => s.object.title),
);

// With model options
const { data } = await unbody.get.textDocument.search
  .about("renewable energy")
  .ask("What are the main challenges for solar power adoption?", {
    model: "gpt-4",
    temperature: 0.3,
  })
  .exec();
```

### Multiple Queries

```typescript
// Execute multiple queries at once
const [docsResult, imagesResult] = await unbody.exec(
  unbody.get.textDocument.limit(5),
  unbody.get.imageBlock.where({ alt: "Quantum computer" }),
);

const documents = docsResult.data.payload;
const images = imagesResult.data.payload;
```

### Working with Results

Response objects in Unbody follow a consistent structure. Understanding this structure is essential for properly accessing your data:

```typescript
// Standard response structure
const result = await unbody.get.textDocument.exec();

// The main data payload - array of documents
const documents = result.data.payload;

// Additional metadata for each document (if available)
// Note: Always use optional chaining when accessing _additional properties
const certaintyScore = documents[0]?._additional?.certainty ?? 0;
const distance = documents[0]?._additional?.distance;

// Meta information like counts (if available)
const metaInfo = result.data.meta;
const totalResults = metaInfo?.total ?? 0;

// Error information (if any)
const errors = result.data.errors;
```

All query responses follow this pattern, making it consistent to work with different types of data. When using TypeScript, you can leverage the enhanced `QueryResponse<T>` interface for better type safety:

```typescript
// Base response type
interface QueryResponse<T> {
  data: {
    payload: T[];
    meta?: {
      count: number;
      total: number;
    };
    errors?: any[];
  };
}

// Enhanced type for search results that includes _additional metadata
interface SearchResult<T>
  extends QueryResponse<
    T & {
      _additional?: {
        certainty?: number;
        distance?: number;
        score?: number;
        // Other metadata fields that may be present
      };
    }
  > {}

// Usage with basic type
const { data }: QueryResponse<TextDocument> = await unbody.get.textDocument
  .select("id", "title", "content")
  .exec();

// Usage with search results and _additional metadata
const { data }: SearchResult<TextDocument> =
  await unbody.get.textDocument.search
    .about("climate change")
    .select("id", "title", "content")
    .exec();

// Now you can safely work with the typed payload and metadata
const documents = data.payload;
const topMatch = documents[0];
const matchScore = topMatch?._additional?.certainty;
```

## GENERATIVE API

### Text Generation

```typescript
interface TextGenerationResponse {
  data: {
    payload: string;
    usage: {
      promptTokens: number;
      completionTokens: number;
      totalTokens: number;
    };
  };
}

// Basic generation
try {
  const { data }: TextGenerationResponse = await unbody.generate.text(
    "Explain AI's impact on healthcare.",
  );

  console.log("Generated text:", data.payload);
  console.log("Tokens used:", data.usage.totalTokens);
} catch (error) {
  console.error("Text generation failed:", error.message);
}

// With options
const { data } = await unbody.generate.text(
  "Write a report on climate change.",
  {
    model: "gpt-4",
    temperature: 0.7,
    maxTokens: 1000,
    presencePenalty: 0.2,
    frequencyPenalty: 0.2,
  },
);

// Message-based
const { data } = await unbody.generate.text(
  [
    { role: "system", content: "You are an AI specialist." },
    { role: "user", content: "What are key AI trends in 2024?" },
  ],
  { model: "gpt-4" },
);
```

### JSON Generation

```typescript
interface JsonGenerationResponse<T> {
  data: {
    payload: T;
    usage: {
      promptTokens: number;
      completionTokens: number;
      totalTokens: number;
    };
  };
}

// With plain schema
interface UserSchema {
  name: string;
  age: number;
  address: string;
}

try {
  const { data }: JsonGenerationResponse<UserSchema> =
    await unbody.generate.json("Provide user details:", {
      schema: {
        type: "object",
        properties: {
          name: { type: "string" },
          age: { type: "number" },
          address: { type: "string" },
        },
      },
    });

  console.log("Generated JSON:", data.payload);
  console.log("User name:", data.payload.name);
} catch (error) {
  if (error.message.includes("schema validation")) {
    console.error("JSON generation schema validation failed");
  }
}

// With Zod schema
import { z } from "zod";

const productSchema = z.object({
  productName: z.string(),
  price: z.number().positive(),
  inStock: z.boolean(),
});

type Product = z.infer<typeof productSchema>;

const { data }: JsonGenerationResponse<Product> = await unbody.generate.json(
  "Create a product entry:",
  { schema: productSchema },
);

console.log("Product name:", data.payload.productName);
console.log("Price:", data.payload.price);
```

## DATA INGESTION

### Prebuilt Integrations

- Google Drive: Documents, sheets, images, video files
- Discord: Messages and attachments
- GitHub: Repositories, issues, comments
- Google Calendar: Events and attachments

See also: Provider-specific sections for configuration details

### Push API for Custom Data

#### Setting Up Push API

```typescript
import { UnbodyPushAPI } from "unbody/push";

interface PushApiConfig {
  projectId: string;
  sourceId: string;
  auth: {
    apiKey: string;
  };
}

const pushApi = new UnbodyPushAPI({
  projectId: "[project-id]",
  sourceId: "[source-id]",
  auth: { apiKey: "pk_..." },
});

// Error handling
try {
  const pushApi = new UnbodyPushAPI({
    /*...*/
  });
} catch (error) {
  console.error("Push API setup failed:", error.message);
}
```

#### Uploading Files

```typescript
interface UploadResponse {
  data: {
    id: string;
    collection: string;
    contentType: string;
  };
}

// Upload file with form data
const formData = new FormData();
formData.append("id", "file-123");
formData.append("file", fileObject);
formData.append(
  "payload",
  JSON.stringify({
    customField: "value",
  }),
);

try {
  const { data }: UploadResponse = await pushApi.files.upload({
    form: formData,
  });

  console.log(`File uploaded as ${data.collection} with ID: ${data.id}`);
} catch (error) {
  if (error.status === 413) {
    console.error("File too large");
  }
}

// Upload with direct parameters
const { data } = await pushApi.files.upload({
  id: "file-uuid",
  file: fileObject,
  payload: { customField: "value" },
});

// Upload from URL
const { data } = await pushApi.files.upload({
  id: "file-from-url",
  filename: "image.jpg",
  url: "https://example.com/image.jpg",
});
```

#### Managing Custom Records

```typescript
interface RecordResponse {
  data: {
    id: string;
    collection: string;
    type: "record" | "file";
  };
}

// Create custom records
try {
  const { data }: RecordResponse = await pushApi.records.create({
    id: "record-id",
    collection: "CustomCollection",
    payload: {
      field1: "value1",
      field2: "value2",
    },
  });

  console.log(`Record created in ${data.collection} with ID: ${data.id}`);
} catch (error) {
  if (error.status === 422) {
    console.error("Invalid record data:", error.message);
  }
}

// Update records (replace payload)
await pushApi.records.update({
  id: "record-id",
  collection: "CustomCollection",
  payload: { updatedField: "newValue" },
});

// Patch records (partial update)
await pushApi.records.patch({
  id: "record-id",
  collection: "CustomCollection",
  payload: { onlyThisField: "gets updated" },
});

// Create cross-references
await pushApi.records.patch({
  id: "record-id",
  collection: "CustomCollection",
  payload: {
    relatedImages: [
      {
        id: "image-id",
        collection: "ImageBlock",
      },
    ],
  },
});

// Delete records
await pushApi.records.delete({
  id: "record-id",
  collection: "CustomCollection",
});
```

#### Listing and Querying Push API Data

```typescript
interface ListFilesResponse {
  data: {
    files: Array<{
      id: string;
      collection: string;
      type: "file";
    }>;
    cursor: string;
    errors?: any[];
  };
}

// List files with pagination
const {
  data: { files, cursor },
}: ListFilesResponse = await pushApi.files.list({
  limit: 10,
  cursor: previousCursor,
});

console.log(`Retrieved ${files.length} files`);

// List records for a collection
interface ListRecordsResponse {
  data: {
    records: Array<{
      id: string;
      collection: string;
      type: "record" | "file";
    }>;
    cursor: string;
    errors?: any[];
  };
}

const {
  data: { records, cursor },
}: ListRecordsResponse = await pushApi.records.list({
  collection: "CustomCollection",
  limit: 20,
});

console.log(`Retrieved ${records.length} records`);

// See also: Content API section for querying the imported data
```

## CUSTOM SCHEMAS

### Defining Custom Schemas

```typescript
import { CustomSchema } from "unbody/admin";

// Create custom collection
const customSchema = new CustomSchema();
const customCollection = new CustomSchema.Collection("CustomCollection");

// Define fields with constraints
customCollection.add(new CustomSchema.Field.Text("name", "Name"));
customCollection.add(new CustomSchema.Field.Number("price", "Price"));
customCollection.add(new CustomSchema.Field.Boolean("inStock", "In Stock"));
customCollection.add(new CustomSchema.Field.Date("createdAt", "Creation Date"));
customCollection.add(
  new CustomSchema.Field.Text(
    "description",
    "Description",
    false, // not an array
    "word", // tokenization strategy
  ),
);

// Add collection to schema
customSchema.add(customCollection);

// Apply to project
project.settings.set(customSchema);
```

### Extending Built-in Types

```typescript
// Extend built-in schema
const videoFileCollection = new CustomSchema.Collection("VideoFile").add(
  new CustomSchema.Field.Text("xChapters", "Extracted Chapters", true),
);

customSchema.extend(videoFileCollection);

// Note: Extended fields for built-in types must start with 'x'
```

### Complex Field Types

```typescript
// Object fields
const productCollection = new CustomSchema.Collection("ProductCollection");

// Nested object
const addressField = new CustomSchema.Field.Object("address", "Address");
addressField
  .add(new CustomSchema.Field.Text("street", "Street"))
  .add(new CustomSchema.Field.Text("city", "City"))
  .add(new CustomSchema.Field.Text("postalCode", "Postal Code"));

productCollection.add(addressField);

// Cross-references between collections
const authorCollection = new CustomSchema.Collection("AuthorCollection");
const bookCollection = new CustomSchema.Collection("BookCollection");

// Add a reference from books to authors
const authorRefField = new CustomSchema.Field.Cref("author", "Book Author");
authorRefField.add("AuthorCollection", "id");
bookCollection.add(authorRefField);

// Add both collections
customSchema.add(authorCollection).add(bookCollection);
```

### Using Custom Schema Types in SDK

```typescript
// Define TypeScript interfaces for your collections
interface Product {
  id: string;
  name: string;
  price: number;
  inStock: boolean;
  address: {
    street: string;
    city: string;
    postalCode: string;
  };
}

// Query with type safety
const { data } = await unbody.get
  .collection<Product>("ProductCollection")
  .select("id", "name", "price", "inStock", "address")
  .exec();

// Process response with type checking
data.payload.forEach((product) => {
  console.log(`${product.name}: $${product.price}`);
  console.log(`Location: ${product.address.city}`);
});
```

## ENHANCEMENT PIPELINES

### Creating Enhancement Pipelines

```typescript
import { Enhancement } from "unbody/admin";

// Define pipeline for a specific collection
const pipeline = new Enhancement.Pipeline("enrich_document", "TextDocument", {
  // Optional: Only run if condition is true
  if: (ctx) => ctx.record.text && ctx.record.text.length > 100,
});

// Create step with action
const extractKeywords = new Enhancement.Step(
  "extract_keywords",
  new Enhancement.Action.StructuredGenerator({
    model: "openai-gpt-4o",
    prompt: (ctx) => `Extract keywords from: ${ctx.record.text}`,
    schema: (ctx, { z }) =>
      z.object({
        keywords: z.array(z.string()),
      }),
  }),
  {
    // Map results to fields
    output: {
      xKeywords: (ctx) => ctx.result.json.keywords,
    },
    // Optional: Skip step conditionally
    if: (ctx) => ctx.record.text.length > 100,
    // Error handling
    onFailure: "continue", // or 'stop'
  },
);

// Add step to pipeline
pipeline.add(extractKeywords);

// Add more steps as needed
const summarizeContent = new Enhancement.Step(
  "summarize",
  new Enhancement.Action.TextGenerator({
    model: "openai-gpt-4o",
    prompt: (ctx) => `Summarize this text: ${ctx.record.text}`,
    temperature: 0.5,
  }),
  {
    output: {
      xSummary: (ctx) => ctx.result.content,
    },
  },
);

pipeline.add(summarizeContent);

// Apply to project
const enhancement = new Enhancement();
enhancement.add(pipeline);
project.settings.set(enhancement);
```

### Available Enhancement Actions

```typescript
// Text generation
new Enhancement.Action.TextGenerator({
  model: "openai-gpt-4o", // or function: (ctx) => ctx.vars.modelName
  prompt: (ctx) => `Generate from: ${ctx.record.content}`,
  temperature: 0.7,
  maxTokens: 500,
});

// Structured JSON generation
new Enhancement.Action.StructuredGenerator({
  model: "openai-gpt-4o",
  prompt: (ctx) => `Extract data from: ${ctx.record.content}`,
  schema: (ctx, { z }) =>
    z.object({
      title: z.string(),
      topics: z.array(z.string()),
      sentiment: z.enum(["positive", "neutral", "negative"]),
    }),
});

// Long text summarization
new Enhancement.Action.Summarizer({
  model: "openai-gpt-4o",
  text: (ctx) => ctx.record.content,
  metadata: (ctx) => `Title: ${ctx.record.title}`,
});

// See also: Custom Schemas section for defining fields to store enhancement results
```

## MEDIA APIS

### Image API

Powered by Imgix, allows image transformations through URL parameters:

```
// Basic usage
image.cdn.unbody.io/[path]?w=300&h=200&crop=faces

// In code (example with React)
<img
  src={`${imageUrl}?w=300&h=200&crop=faces&auto=format,compress`}
  alt="Profile picture"
/>

// Error handling
<img
  src={`${imageUrl}?w=300&h=200`}
  alt="Profile picture"
  onError={(e) => {
    e.target.src = "fallback-image.jpg";
    console.error("Image failed to load");
  }}
/>
```

Common parameters:

- Resize: `w=300`, `h=200`
- Crop: `crop=faces`, `crop=focalpoint`
- Format: `fm=jpg`, `fm=png`, `fm=webp`
- Quality: `q=80` (lower values = smaller file size)
- Auto: `auto=format,compress`
- Face detection: `fit=facearea`, `facepad=2.0`
- Blur: `blur=50`

### Video API

Powered by Mux, provides:

- Streaming via HLS
- Automated transcription
- Thumbnail generation
- MP4 renditions

```typescript
// Get video playback URL
const { data } = await unbody.get
  .videoFile
  .select("playbackId", "hlsUrl", "thumbnailUrl")
  .exec();

const videoData = data.payload[0];

// Using with a player (React example)
<MuxPlayer
  playbackId={videoData.playbackId}
  metadata={{
    video_title: "My Video"
  }}
/>

// Custom player with HLS.js
const videoElement = document.getElementById('video-player');
const hls = new Hls();
hls.loadSource(videoData.hlsUrl);
hls.attachMedia(videoElement);

// Get video transcript
const { data } = await unbody.get
  .videoFile
  .select(
    "subtitles.SubtitleFile.entries.SubtitleEntry.start",
    "subtitles.SubtitleFile.entries.SubtitleEntry.end",
    "subtitles.SubtitleFile.entries.SubtitleEntry.text"
  )
  .exec();

// Get video thumbnail
const thumbnailUrl = `${videoData.thumbnailUrl}?time=15`;
```

## DATA TYPES

### Core Interface Properties

Most data types share these common properties:

```typescript
interface BaseObject {
  id: string;
  createdAt: string; // ISO datetime
  updatedAt: string; // ISO datetime
  sourceId: string;
  remoteId: string;
}
```

### Main Data Types

```typescript
// Text Documents
interface TextDocument extends BaseObject {
  title: string;
  text: string;
  html?: string;
  path: string[];
  pathString: string;
  mimeType: string;
  originalName: string;
  authors?: string[];
  tags?: string[];
  blocks?: (TextBlock | ImageBlock)[];
  // Auto-generated fields if enabled:
  autoSummary?: string;
  autoKeywords?: string[];
  autoEntities?: string[];
  autoTopics?: string[];
}

// Images
interface ImageBlock extends BaseObject {
  url: string;
  width: number;
  height: number;
  size: number;
  mimeType: string;
  caption?: string;
  alt?: string;
}

// Videos
interface VideoFile extends BaseObject {
  url: string;
  width: number;
  height: number;
  duration: number;
  size: number;
  mimeType: string;
  playbackId: string;
  hlsUrl: string;
  thumbnailUrl: {
    png: string;
    jpeg: string;
    webp: string;
  };
  animatedImageUrl: {
    gif: string;
    webp: string;
  };
  subtitles?: SubtitleFile[];
}

// See also: Data Types section in documentation for complete type definitions
```

## BEST PRACTICES

### Query Optimization

1. **Filter Early**: Apply filters first to reduce the dataset before applying complex operations.

   ```typescript
   // Better performance - filter early
   const result = await unbody.get.textDocument
     .where({ category: "important" }) // Apply filter first
     .search.about("complex topic") // Then search within filtered set
     .exec();
   ```

2. **Select Only Needed Fields**: Use `.select()` to retrieve only the fields you need.

   ```typescript
   // Only retrieve necessary fields
   const result = await unbody.get.textDocument
     .select("id", "title", "summary") // Don't fetch entire content
     .exec();
   ```

3. **Pagination**: Always use `.limit()` for large datasets.
   ```typescript
   // Paginate results
   const result = await unbody.get.textDocument
     .limit(20)
     .offset(page * 20)
     .exec();
   ```

### Type Safety

Use TypeScript interfaces to ensure type safety for your queries and responses:

```typescript
interface MyDocument extends TextDocument {
  customField: string;
}

const result = await unbody.get
  .collection<MyDocument>("TextDocument")
  .where({ customField: "value" })
  .exec();
```

### Error Handling

1. Implement proper error handling with specific responses for different error codes.
2. Use retry mechanisms with exponential backoff for transient errors (429, 500).

### Response Handling

Consistently unwrap data from response objects to ensure clean, maintainable code:

```typescript
// Standard pattern for accessing data from responses
const result = await unbody.get.textDocument.exec();

// Always access the payload through the data property
const documents = result.data.payload;

// Use destructuring for cleaner code with TypeScript support
const { data }: QueryResponse<TextDocument> =
  await unbody.get.textDocument.exec();

// Then access payload directly
const documentsData = data.payload;

// Check for errors
if (data.errors && data.errors.length > 0) {
  console.error("Query returned errors:", data.errors);
}
```

### Query Reuse

For complex queries that you'll use multiple times, build the query once and reuse it:

```typescript
// Define the query once
const recentDocumentsQuery = unbody.get.textDocument
  .where({
    createdAt: { $gt: new Date(Date.now() - 86400000) }, // Last 24 hours
  })
  .sort("createdAt", "desc");

// Execute in different places
const result1 = await recentDocumentsQuery.exec();
const result2 = await recentDocumentsQuery.limit(5).exec();
```

## USAGE PATTERNS

### RAG (Retrieval Augmented Generation)

```typescript
// Basic RAG workflow
try {
  const { data } = await unbody.get
    .textDocument
    .search.about("user query")
    .limit(5)
    .select("title", "text")
    .generate.fromMany(
      "Using the provided documents, answer: user question",
      ["title", "text"],
      { temperature: 0.7 }
    )
    .exec();
```
