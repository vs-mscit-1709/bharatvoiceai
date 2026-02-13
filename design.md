# Design Document: Government Schemes Voice Assistant

## Overview

The Government Schemes Voice Assistant is a web-based application that enables Indian citizens to access government scheme information through natural voice interaction in their regional language. The system follows a pipeline architecture where user speech flows through recognition, language detection, AI-powered query processing, database retrieval, response simplification, translation, and speech synthesis.

The design prioritizes accessibility, supporting users with varying literacy levels and language preferences. It leverages modern web APIs for speech processing, cloud-based AI services for natural language understanding, and a structured database for scheme information.

### Key Design Principles

1. **Language-First Design**: Every component must handle multilingual content natively
2. **Graceful Degradation**: System provides fallbacks when components fail
3. **Simplicity Over Complexity**: Interface and responses optimized for non-technical users
4. **Modular Architecture**: Components can be independently updated or replaced
5. **Performance on Limited Bandwidth**: Optimized for rural connectivity scenarios

## Architecture

The system follows a client-server architecture with the following high-level components:

```
┌─────────────────────────────────────────────────────────────┐
│                        Web Client                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   UI Layer   │  │ Audio Input  │  │ Audio Output │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                            │
                    HTTPS/WebSocket
                            │
┌─────────────────────────────────────────────────────────────┐
│                      Backend Server                          │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              API Gateway / Router                     │   │
│  └──────────────────────────────────────────────────────┘   │
│                            │                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Speech     │  │   Language   │  │    Query     │      │
│  │  Recognizer  │  │   Detector   │  │  Processor   │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                            │                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Scheme     │  │   Response   │  │    Speech    │      │
│  │   Retrieval  │  │  Simplifier  │  │ Synthesizer  │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                            │                                 │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Scheme Database                         │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                            │
                    External Services
                            │
┌─────────────────────────────────────────────────────────────┐
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Speech API  │  │   AI/LLM     │  │ Translation  │      │
│  │  (Google/    │  │  Service     │  │   Service    │      │
│  │   Azure)     │  │              │  │              │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

### Data Flow

1. **User speaks** → Audio captured via Web Audio API
2. **Audio sent to backend** → Transmitted over WebSocket for real-time processing
3. **Speech Recognition** → Audio converted to text using cloud speech API
4. **Language Detection** → Text analyzed to identify language
5. **Query Processing** → AI extracts intent and entities from query
6. **Scheme Retrieval** → Database queried for matching schemes
7. **Response Simplification** → Complex information simplified using AI
8. **Translation** → Response translated to user's language if needed
9. **Speech Synthesis** → Text converted to audio in target language
10. **Audio playback** → Response played to user with controls

## Components and Interfaces

### 1. Web Client

**Responsibilities:**
- Capture audio from user's microphone
- Display visual feedback (listening, processing, speaking states)
- Play audio responses
- Provide playback controls (pause, replay, speed adjustment)
- Handle offline scenarios with cached responses

**Key Interfaces:**
```typescript
interface VoiceClient {
  startListening(): Promise<void>
  stopListening(): Promise<AudioBlob>
  playResponse(audio: AudioBlob): Promise<void>
  displayState(state: 'idle' | 'listening' | 'processing' | 'speaking'): void
  showError(message: string, language: string): void
}
```

**Technology Choices:**
- Web Audio API for audio capture
- MediaRecorder API for audio encoding
- HTML5 Audio element for playback
- WebSocket for real-time communication
- Service Worker for offline caching

### 2. API Gateway

**Responsibilities:**
- Route incoming requests to appropriate services
- Handle authentication and rate limiting
- Manage WebSocket connections
- Aggregate responses from multiple services
- Implement circuit breaker pattern for external service failures

**Key Interfaces:**
```typescript
interface APIGateway {
  handleVoiceQuery(audio: AudioBlob, sessionId: string): Promise<VoiceResponse>
  getSchemeDetails(schemeId: string, language: string): Promise<SchemeInfo>
  healthCheck(): Promise<ServiceStatus>
}

interface VoiceResponse {
  audio: AudioBlob
  text: string
  language: string
  schemes: SchemeInfo[]
  processingTime: number
}
```

### 3. Speech Recognizer

**Responsibilities:**
- Convert audio to text using external speech API
- Handle audio format conversion
- Implement retry logic for failed recognitions
- Filter background noise
- Detect silence and speech boundaries

**Key Interfaces:**
```typescript
interface SpeechRecognizer {
  recognize(audio: AudioBlob): Promise<RecognitionResult>
  validateAudioQuality(audio: AudioBlob): AudioQualityScore
}

interface RecognitionResult {
  text: string
  confidence: number
  alternatives: string[]
  language: string | null
}
```

**Implementation Approach:**
- Use Google Cloud Speech-to-Text or Azure Speech Services
- Support multiple audio formats (WebM, WAV, MP3)
- Implement audio preprocessing for noise reduction
- Cache recognition results for identical audio inputs

### 4. Language Detector

**Responsibilities:**
- Identify language from text input
- Provide confidence scores
- Handle code-mixed text (multiple languages in one query)
- Support all 10 target languages

**Key Interfaces:**
```typescript
interface LanguageDetector {
  detect(text: string): Promise<LanguageResult>
  getSupportedLanguages(): string[]
}

interface LanguageResult {
  language: string
  confidence: number
  alternatives: Array<{language: string, confidence: number}>
}
```

**Implementation Approach:**
- Use language detection library (e.g., franc, langdetect)
- Implement character-based n-gram analysis for Indian languages
- Fallback to script detection (Devanagari → Hindi, Tamil script → Tamil)
- Confidence threshold of 0.7 for automatic acceptance

### 5. Query Processor

**Responsibilities:**
- Extract intent from user query (search schemes, get details, ask eligibility)
- Identify entities (location, user category, scheme type)
- Handle ambiguous queries with clarification
- Maintain conversation context for follow-up questions

**Key Interfaces:**
```typescript
interface QueryProcessor {
  process(text: string, language: string, context: ConversationContext): Promise<ProcessedQuery>
  clarify(query: ProcessedQuery, userResponse: string): Promise<ProcessedQuery>
}

interface ProcessedQuery {
  intent: 'search' | 'details' | 'eligibility' | 'application' | 'clarification_needed'
  entities: {
    location?: string
    category?: string[]
    schemeType?: string
    keywords?: string[]
  }
  clarificationQuestion?: string
  confidence: number
}

interface ConversationContext {
  sessionId: string
  previousQueries: ProcessedQuery[]
  userProfile?: {
    state?: string
    category?: string[]
  }
}
```

**Implementation Approach:**
- Use LLM (GPT-4, Claude, or Gemini) for intent extraction
- Implement prompt engineering for multilingual understanding
- Use few-shot examples for Indian context
- Extract entities using NER (Named Entity Recognition)
- Maintain session state in Redis for conversation context

### 6. Scheme Retrieval Service

**Responsibilities:**
- Query database for matching schemes
- Rank results by relevance
- Filter by eligibility criteria
- Handle partial matches and suggestions

**Key Interfaces:**
```typescript
interface SchemeRetrieval {
  search(query: ProcessedQuery): Promise<SchemeSearchResult[]>
  getById(schemeId: string): Promise<SchemeInfo>
  getSuggestions(query: ProcessedQuery): Promise<SchemeInfo[]>
}

interface SchemeSearchResult {
  scheme: SchemeInfo
  relevanceScore: number
  matchedCriteria: string[]
}

interface SchemeInfo {
  id: string
  name: string
  description: string
  eligibility: string[]
  benefits: string[]
  requiredDocuments: string[]
  applicationProcess: string
  deadlines: Date[]
  state?: string
  category: string[]
  officialLink: string
  lastUpdated: Date
}
```

**Implementation Approach:**
- Use PostgreSQL with full-text search (tsvector)
- Implement vector similarity search for semantic matching
- Create indexes on category, state, and keywords
- Use ranking algorithm: exact match > category match > keyword match
- Cache frequently accessed schemes in Redis

### 7. Response Simplifier

**Responsibilities:**
- Convert complex government language to simple terms
- Maintain accuracy while simplifying
- Structure response in clear sections
- Limit response length for voice delivery
- Add explanations for technical terms

**Key Interfaces:**
```typescript
interface ResponseSimplifier {
  simplify(schemes: SchemeInfo[], query: ProcessedQuery, language: string): Promise<SimplifiedResponse>
  validateAccuracy(original: SchemeInfo, simplified: string): Promise<boolean>
}

interface SimplifiedResponse {
  summary: string
  sections: {
    schemeName: string
    whoCanApply: string
    benefits: string
    documentsNeeded: string
    howToApply: string
    importantDates: string
  }
  wordCount: number
  readingLevel: string
}
```

**Implementation Approach:**
- Use LLM with specific prompts for simplification
- Implement templates for consistent structure
- Set max word count constraint (200 words)
- Use readability metrics (Flesch-Kincaid adapted for Indian languages)
- Validate that key information (dates, amounts, eligibility) is preserved

### 8. Speech Synthesizer

**Responsibilities:**
- Convert text to natural speech in target language
- Support multiple Indian language voices
- Control speech rate and pitch
- Generate audio in web-compatible format
- Cache generated audio for repeated responses

**Key Interfaces:**
```typescript
interface SpeechSynthesizer {
  synthesize(text: string, language: string, options: SynthesisOptions): Promise<AudioBlob>
  getAvailableVoices(language: string): Promise<Voice[]>
}

interface SynthesisOptions {
  rate: number // 0.5 to 2.0, default 1.0
  pitch: number // 0.5 to 2.0, default 1.0
  voice?: string
}

interface Voice {
  id: string
  language: string
  gender: 'male' | 'female' | 'neutral'
  name: string
}
```

**Implementation Approach:**
- Use Google Cloud Text-to-Speech or Azure Speech Services
- Select neural voices for natural pronunciation
- Default speech rate: 0.85 (slightly slower for comprehension)
- Cache audio files in CDN for repeated responses
- Fallback to browser Web Speech API if cloud service fails

## Data Models

### Scheme Database Schema

```sql
-- Schemes table
CREATE TABLE schemes (
  id UUID PRIMARY KEY,
  name_en TEXT NOT NULL,
  name_hi TEXT,
  name_regional JSONB, -- {gu: "...", ta: "...", etc}
  description_en TEXT NOT NULL,
  description_regional JSONB,
  category TEXT[] NOT NULL, -- ['farmer', 'student', 'woman', etc]
  state TEXT, -- NULL for central schemes
  eligibility JSONB NOT NULL, -- structured eligibility criteria
  benefits JSONB NOT NULL,
  required_documents JSONB NOT NULL,
  application_process_en TEXT NOT NULL,
  application_process_regional JSONB,
  deadlines JSONB, -- [{type: 'application', date: '2024-12-31'}, ...]
  official_link TEXT,
  keywords TEXT[], -- for search optimization
  search_vector tsvector, -- for full-text search
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  last_verified_at TIMESTAMP
);

-- Indexes
CREATE INDEX idx_schemes_category ON schemes USING GIN(category);
CREATE INDEX idx_schemes_state ON schemes(state);
CREATE INDEX idx_schemes_search ON schemes USING GIN(search_vector);
CREATE INDEX idx_schemes_keywords ON schemes USING GIN(keywords);

-- User sessions table (for conversation context)
CREATE TABLE user_sessions (
  session_id UUID PRIMARY KEY,
  user_profile JSONB, -- {state: 'Gujarat', category: ['farmer']}
  conversation_history JSONB[], -- array of queries and responses
  created_at TIMESTAMP DEFAULT NOW(),
  last_activity TIMESTAMP DEFAULT NOW(),
  expires_at TIMESTAMP
);

-- Query logs (for analytics and improvement)
CREATE TABLE query_logs (
  id UUID PRIMARY KEY,
  session_id UUID REFERENCES user_sessions(session_id),
  query_text TEXT NOT NULL,
  detected_language TEXT,
  intent TEXT,
  entities JSONB,
  matched_schemes UUID[],
  response_time_ms INTEGER,
  user_feedback TEXT, -- 'helpful', 'not_helpful', NULL
  created_at TIMESTAMP DEFAULT NOW()
);

-- Audio cache (for frequently requested responses)
CREATE TABLE audio_cache (
  id UUID PRIMARY KEY,
  text_hash TEXT UNIQUE NOT NULL, -- hash of text + language + voice
  audio_url TEXT NOT NULL,
  language TEXT NOT NULL,
  duration_seconds FLOAT,
  access_count INTEGER DEFAULT 0,
  last_accessed TIMESTAMP DEFAULT NOW(),
  created_at TIMESTAMP DEFAULT NOW()
);
```

### Application State Models

```typescript
// Client-side state
interface AppState {
  currentState: 'idle' | 'listening' | 'processing' | 'speaking' | 'error'
  sessionId: string
  conversationHistory: ConversationTurn[]
  currentAudio: AudioBlob | null
  userPreferences: {
    language?: string
    speechRate: number
  }
  cachedResponses: Map<string, CachedResponse>
}

interface ConversationTurn {
  timestamp: Date
  userQuery: string
  detectedLanguage: string
  systemResponse: string
  schemes: SchemeInfo[]
  audioUrl?: string
}

interface CachedResponse {
  text: string
  audio: AudioBlob
  timestamp: Date
  expiresAt: Date
}
```

## Correctness Properties


A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.

### Core Processing Properties

**Property 1: Speech Recognition Produces Text**
*For any* valid audio input containing speech, the Speech_Recognizer should produce non-empty text output.
**Validates: Requirements 1.1**

**Property 2: Language Detection Accuracy**
*For any* text input in a supported language (Gujarati, Hindi, Tamil, Bengali, Marathi, Telugu, Kannada, Malayalam, Punjabi, English), the Language_Detector should correctly identify that language with confidence >= 0.7.
**Validates: Requirements 2.1**

**Property 3: Entity Extraction Completeness**
*For any* query containing location, category, or scheme type entities, the Query_Processor should extract all present entities and include them in the ProcessedQuery.
**Validates: Requirements 3.1, 3.2, 3.3**

**Property 4: Intent Consistency Across Phrasings**
*For any* set of semantically equivalent queries (same meaning, different wording), the Query_Processor should produce the same intent classification.
**Validates: Requirements 3.5**

### Retrieval and Ranking Properties

**Property 5: Relevance-Based Ranking**
*For any* query that matches multiple schemes, the returned results should be ordered by descending relevance score, where relevance_score[i] >= relevance_score[i+1] for all i.
**Validates: Requirements 4.2**

**Property 6: Fallback Suggestions for Empty Results**
*For any* query that produces no exact matches, the system should return at least one suggested scheme or category.
**Validates: Requirements 4.3**

**Property 7: Scheme Completeness**
*For any* scheme returned by the retrieval service, it should contain all required fields: name, eligibility, benefits, requiredDocuments, applicationProcess, and deadlines.
**Validates: Requirements 4.4**

**Property 8: Outdated Scheme Flagging**
*For any* scheme where (current_date - last_updated) > 90 days, the response should include a verification notice.
**Validates: Requirements 4.5**

### Simplification and Translation Properties

**Property 9: Response Simplification Reduces Complexity**
*For any* scheme information, the simplified version should have lower readability complexity score than the original while maintaining the same key facts.
**Validates: Requirements 5.1, 5.2**

**Property 10: Critical Data Preservation Through Transformation**
*For any* scheme with numerical values, dates, or eligibility criteria, these should remain identical through both simplification and translation transformations.
**Validates: Requirements 5.2, 6.4**

**Property 11: Response Structure Consistency**
*For any* simplified response, it should contain all required sections (schemeName, whoCanApply, benefits, documentsNeeded, howToApply, importantDates) in both the original and translated versions.
**Validates: Requirements 5.4, 6.2**

**Property 12: Word Count Constraint**
*For any* simplified response, the word count should be <= 200 words.
**Validates: Requirements 5.5**

**Property 13: Technical Term Explanation**
*For any* simplified response containing technical terms (identified by domain-specific vocabulary), those terms should be followed by explanations in parentheses.
**Validates: Requirements 5.3**

**Property 14: Language Consistency (Round-Trip)**
*For any* query in language L, the response should also be in language L, maintaining the same language throughout the interaction.
**Validates: Requirements 6.1**

### Speech Synthesis Properties

**Property 15: Text-to-Speech Conversion**
*For any* text response in a supported language, the Speech_Synthesizer should produce audio output with duration > 0.
**Validates: Requirements 7.1**

**Property 16: Speech Rate Consistency**
*For any* text response with word count W, the generated audio duration should be approximately W / 150 minutes (±10% tolerance), corresponding to 150 words per minute.
**Validates: Requirements 7.3**

### Caching and Performance Properties

**Property 17: Response Caching (Idempotence)**
*For any* query Q, executing it twice should result in the second execution using cached data, with identical response content but faster retrieval time.
**Validates: Requirements 8.4, 11.5**

**Property 18: Temporary Data Cleanup**
*For any* audio or text data marked as temporary, it should not exist in storage after 24 hours from creation.
**Validates: Requirements 10.3**

**Property 19: Audio Processing Without Permanent Storage**
*For any* audio input processed by the system, no raw audio file should exist in permanent storage after processing completes.
**Validates: Requirements 10.1**

### Database Management Properties

**Property 20: Update Timestamp Accuracy**
*For any* scheme update operation, the updated_at timestamp should be set to the current time (within 1 second tolerance).
**Validates: Requirements 12.2**

**Property 21: Bulk Import Correctness**
*For any* valid CSV or JSON file containing N scheme records, importing it should create exactly N scheme entries in the database with matching data.
**Validates: Requirements 12.3**

**Property 22: Scheme Validation Enforcement**
*For any* scheme data, if it lacks required fields (name, eligibility, benefits, documents, process), it should be rejected; if complete, it should be accepted.
**Validates: Requirements 12.4**

**Property 23: Version History Tracking**
*For any* scheme update, a new version history record should be created containing the previous state and timestamp.
**Validates: Requirements 12.5**

**Property 24: Cross-Language Response Consistency**
*For any* query about scheme S, responses in different languages should contain the same core information (eligibility criteria, benefit amounts, deadlines) when translated back to a common language.
**Validates: Requirements 13.4**

## Error Handling

The system implements multiple layers of error handling to ensure graceful degradation:

### Client-Side Error Handling

1. **Microphone Access Denied**: Display clear message with instructions to enable microphone permissions
2. **Network Timeout**: Show offline indicator and offer to retry with cached data
3. **Audio Playback Failure**: Fall back to displaying text response
4. **Browser Compatibility**: Detect unsupported browsers and suggest alternatives

### Server-Side Error Handling

1. **Speech Recognition Failure**:
   - Retry with adjusted audio parameters (noise reduction, normalization)
   - After 3 failures, provide spoken error message in Hindi and English
   - Log failure for analysis

2. **Language Detection Failure**:
   - If confidence < 0.7, ask user to confirm language
   - Default to Hindi if user doesn't respond
   - Log ambiguous cases for model improvement

3. **AI Service Unavailability**:
   - Implement circuit breaker pattern (open after 5 consecutive failures)
   - Fall back to keyword-based search
   - Provide helpline numbers and official website links

4. **Database Connection Loss**:
   - Retry with exponential backoff (1s, 2s, 4s)
   - Serve cached responses if available
   - Display maintenance message after 3 retries

5. **Translation Service Failure**:
   - Fall back to English response with apology message
   - Cache successful translations to reduce dependency

### Error Response Format

```typescript
interface ErrorResponse {
  error: {
    code: string // 'SPEECH_RECOGNITION_FAILED', 'DATABASE_UNAVAILABLE', etc.
    message: string // User-friendly message in detected language
    messageHindi: string // Fallback Hindi message
    messageEnglish: string // Fallback English message
    retryable: boolean
    alternativeResources?: {
      helpline: string[]
      websites: string[]
    }
  }
  timestamp: Date
  requestId: string
}
```

### Logging and Monitoring

- All errors logged with context (user session, query, component)
- Critical errors trigger alerts (database down, AI service unavailable)
- Error rates tracked per component for SLA monitoring
- User-facing errors include request ID for support tracking

## Testing Strategy

The testing strategy employs a dual approach combining property-based testing for universal correctness guarantees and unit testing for specific examples and edge cases.

### Property-Based Testing

Property-based tests validate that the correctness properties defined above hold across a wide range of generated inputs. Each property test will run a minimum of 100 iterations with randomly generated data.

**Testing Library**: Use `fast-check` for JavaScript/TypeScript or `Hypothesis` for Python

**Test Configuration**:
- Minimum 100 iterations per property test
- Each test tagged with: `Feature: government-schemes-voice-assistant, Property {N}: {property_text}`
- Seed-based randomization for reproducibility
- Shrinking enabled to find minimal failing cases

**Property Test Examples**:

```typescript
// Property 2: Language Detection Accuracy
test('Property 2: Language Detection Accuracy', async () => {
  // Feature: government-schemes-voice-assistant, Property 2
  await fc.assert(
    fc.asyncProperty(
      fc.record({
        text: fc.oneof(
          generateGujaratiText(),
          generateHindiText(),
          generateTamilText(),
          // ... other languages
        ),
        expectedLanguage: fc.string()
      }),
      async ({text, expectedLanguage}) => {
        const result = await languageDetector.detect(text);
        expect(result.language).toBe(expectedLanguage);
        expect(result.confidence).toBeGreaterThanOrEqual(0.7);
      }
    ),
    { numRuns: 100 }
  );
});

// Property 12: Word Count Constraint
test('Property 12: Word Count Constraint', async () => {
  // Feature: government-schemes-voice-assistant, Property 12
  await fc.assert(
    fc.asyncProperty(
      generateSchemeInfo(),
      generateProcessedQuery(),
      fc.constantFrom('hi', 'gu', 'ta', 'bn', 'mr'),
      async (scheme, query, language) => {
        const simplified = await responseSimplifier.simplify([scheme], query, language);
        const wordCount = simplified.summary.split(/\s+/).length;
        expect(wordCount).toBeLessThanOrEqual(200);
      }
    ),
    { numRuns: 100 }
  );
});

// Property 17: Response Caching (Idempotence)
test('Property 17: Response Caching', async () => {
  // Feature: government-schemes-voice-assistant, Property 17
  await fc.assert(
    fc.asyncProperty(
      generateProcessedQuery(),
      async (query) => {
        const startTime1 = Date.now();
        const response1 = await schemeRetrieval.search(query);
        const duration1 = Date.now() - startTime1;
        
        const startTime2 = Date.now();
        const response2 = await schemeRetrieval.search(query);
        const duration2 = Date.now() - startTime2;
        
        // Same content
        expect(response2).toEqual(response1);
        // Faster on second call (cached)
        expect(duration2).toBeLessThan(duration1);
      }
    ),
    { numRuns: 100 }
  );
});
```

### Unit Testing

Unit tests focus on specific examples, edge cases, and integration points between components.

**Test Coverage Areas**:

1. **Speech Recognition Edge Cases**:
   - Very short audio (< 1 second)
   - Very long audio (> 60 seconds)
   - High background noise
   - Multiple speakers
   - Silence detection

2. **Language Detection Edge Cases**:
   - Code-mixed text (Hindi + English)
   - Very short text (< 5 words)
   - Text with numbers and special characters
   - Unsupported languages (French, Spanish)

3. **Query Processing Examples**:
   - "What schemes are available for farmers in Gujarat?"
   - "मुझे छात्रवृत्ति के बारे में बताएं" (Tell me about scholarships)
   - "ગુજરાતમાં મહિલાઓ માટે યોજનાઓ" (Schemes for women in Gujarat)
   - Ambiguous query: "Tell me about schemes"

4. **Error Handling Examples**:
   - Speech recognition failure → verify Hindi/English error message
   - Database unavailable → verify retry and fallback
   - AI service timeout → verify alternative resources provided
   - Synthesis failure → verify text fallback displayed

5. **Integration Tests**:
   - End-to-end flow: audio input → text response
   - Multi-turn conversation with context
   - Language switching mid-conversation
   - Offline mode with cached responses

**Test Organization**:
```
tests/
├── unit/
│   ├── speech-recognizer.test.ts
│   ├── language-detector.test.ts
│   ├── query-processor.test.ts
│   ├── scheme-retrieval.test.ts
│   ├── response-simplifier.test.ts
│   └── speech-synthesizer.test.ts
├── property/
│   ├── language-detection.property.test.ts
│   ├── query-processing.property.test.ts
│   ├── retrieval-ranking.property.test.ts
│   ├── simplification.property.test.ts
│   └── caching.property.test.ts
├── integration/
│   ├── end-to-end.test.ts
│   ├── conversation-flow.test.ts
│   └── error-scenarios.test.ts
└── fixtures/
    ├── audio-samples/
    ├── test-queries.json
    └── mock-schemes.json
```

### Test Data Generation

**For Property Tests**:
- Generate random text in each supported language using language-specific character sets
- Generate random scheme data with valid structure
- Generate random queries with various intents and entities
- Use realistic distributions (e.g., more common queries weighted higher)

**For Unit Tests**:
- Curated audio samples in each language
- Real government scheme data (anonymized/simplified)
- Common user queries collected from user research
- Edge cases identified from production logs

### Continuous Testing

- Run unit tests on every commit
- Run property tests nightly (due to longer execution time)
- Integration tests run on staging environment before deployment
- Monitor production for property violations (e.g., response time, word count)
- A/B test simplification quality with user feedback

### Success Criteria

- All property tests pass with 100 iterations
- Unit test coverage > 80% for core components
- Integration tests cover all critical user journeys
- Zero critical bugs in production for 30 days
- User satisfaction score > 4.0/5.0 for response quality
