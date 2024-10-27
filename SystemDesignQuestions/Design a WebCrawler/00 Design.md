## High Level Design

![image](https://github.com/user-attachments/assets/76ca050b-1d27-45d9-91e6-00533c768962)

So, at a high level if we want to break the process of crawling into steps, it will look something like this: -

- Pick a URL from the list.
- Fetch the Ip address of the host & establish a connection to download the corresponding document.
- While parsing the contents of this document look for new URL’s.
- Add the new URLs to the URLs list.
- Store contents, or index the document, process it as needed.
- Go back to step 1.

Now, assuming there are billions of pages to be crawled, here are the most important components we need in our service.

- URL Queue, To store the list of URLs to download.
- HTML Fetcher, To retrieve a web page from the server.
- Link Extractor, from HTML documents.
- Duplicate Resolver, to avoid extracting same data twice.
- Data-store, To store contents, metadata…etc

### 1. URL Queue
A distributed queue is essential for high scalability. For a large-scale setup, you might use:

Message Queue (e.g., Kafka, RabbitMQ): These can handle millions of messages and support parallel consumers, ideal for distributing URL processing among multiple crawler instances.
Redis or DynamoDB: Used as a key-value store to manage URLs with fast access times and TTL features to handle re-crawling.
In Go:

Use a library like segmentio/kafka-go for Kafka or streadway/amqp for RabbitMQ.
Maintain multiple consumers to scale URL processing across crawler instances.

### 2. HTML Fetcher
The HTML fetcher retrieves pages for parsing. Since network requests can be slow, we use concurrency to handle multiple fetches in parallel:

Concurrency: Use goroutines and a worker pool to manage parallel HTTP requests.
HTTP Client: Use a rate-limited HTTP client to avoid overloading servers and implement retry logic with backoff for transient errors.
Error Handling: Use a circuit breaker pattern to skip URLs if a server is overloaded or unresponsive.

```go
Copy code
import (
    "net/http"
    "time"
)

// Limited HTTP client with timeout and retry capabilities
func NewHTTPClient() *http.Client {
    return &http.Client{
        Timeout: 10 * time.Second,
    }
}
```
### 3. Link Extractor
The link extractor parses the HTML content and extracts new URLs:

Parsing: Use libraries like golang.org/x/net/html to parse HTML documents.
URL Normalization: Normalize URLs to avoid duplicates (e.g., stripping trailing slashes, handling relative URLs).
Filter and Sanitize: Only keep valid URLs within the targeted domains or constraints.
In Go:

```go
import (
    "golang.org/x/net/html"
    "strings"
)

// Parse HTML and extract links
func ExtractLinks(htmlContent string) []string {
    var links []string
    doc, _ := html.Parse(strings.NewReader(htmlContent))
    // Traverse and extract links from doc
    return links
}
```
### 4. Duplicate Resolver
To prevent re-crawling the same page, a duplicate resolver is necessary:

Bloom Filters: To check if a URL has been seen before, use a distributed Bloom filter for approximate set membership (like RedisBloom).
Datastore with Hashing: Store crawled URLs in a datastore (like Redis, DynamoDB) where each URL is hashed for uniqueness.
TTL Management: Set TTL on URLs if re-crawling is required after a certain period.
In Go:

Use a Redis-backed Bloom filter library or maintain a simple in-memory cache for smaller crawls.

```go
import "github.com/bits-and-blooms/bloom/v3"

var bf *bloom.BloomFilter

func InitBloomFilter() {
    bf = bloom.New(1000000, 5) // Adjust size and hashes for scale
}

func IsDuplicate(url string) bool {
    return bf.Test([]byte(url))
}

func AddURL(url string) {
    bf.Add([]byte(url))
}
```
### 5. Data Store
To store the page contents and metadata, a highly available database is essential:

Document Store (e.g., MongoDB, Elasticsearch): Store each page’s HTML content, metadata, and timestamps. Both MongoDB and Elasticsearch are ideal for flexible document storage and full-text search.
Relational Database (e.g., PostgreSQL): Useful if you need to enforce strong consistency or structure around metadata and relationships between data points.
In Go:

Use a Go client for MongoDB (e.g., go.mongodb.org/mongo-driver/mongo) or Elasticsearch (github.com/elastic/go-elasticsearch) to connect and store data.

```go
import "go.mongodb.org/mongo-driver/mongo"

func StorePageData(url string, content string, metadata map[string]string) {
    // Connect to MongoDB or another data store
    // Insert or update document with content and metadata
}
```
Putting It All Together
Here’s a basic flow combining these components:

- Queue Management: Read URLs from the URL queue and distribute them to workers.
- HTML Fetching: Each worker retrieves the HTML content for a URL using the HTTP client.
- Link Extraction and Duplication Check: Extract new links from the HTML, check for duplicates, and enqueue any unique links.
- Storage: Store the HTML content and metadata of each processed URL.
- Observer Notifications: Notify observers (logging, monitoring) when a URL is processed, including errors or successes.

## Design Patterns can be used 

- Singleton Pattern: URLQueue uses the singleton pattern to ensure only one instance manages URLs across the application.
- Observer Pattern: ConsoleObserver implements CrawlerObserver and prints each processed URL.
  In a real-time crawler setup, you might have the following observers:

        -- ConsoleObserver: Logs URLs as they’re processed.
        -- ErrorObserver: Monitors failed URL fetches and triggers retries or alerts.
        -- AnalyticsObserver: Collects data for analysis and performance tracking.
        -- RateLimiterObserver: Dynamically adjusts request rates based on observed server responses.
- Command Pattern: CrawlCommand encapsulates the logic for crawling a URL, making requests, parsing results, and notifying observers.
- Factory Pattern: A NewHTTPClient function creates an HTTPClient with predefined settings.
- Politeness Policy: The main function enforces a delay between requests using time.Sleep(1 * time.Second).
- Parsing Strategy: ParserStrategy is implemented by HTMLParserStrategy to parse URLs.

## LLD details 

- Singleton Pattern for URLQueue.
- Factory Pattern for HTTPClientFactory.
- Strategy Pattern for parsers.
- Observer Pattern to notify on URL processing.
- Command Pattern for crawl requests.
- BFS traversal for crawling strategy.
- Politeness Policy for request throttling.

### 1. Singleton URL queue

```go
package main

import "sync"

// URLQueue is a singleton queue for URLs to be processed.
type URLQueue struct {
    urls  chan string
    mutex sync.Mutex
}

var queueInstance *URLQueue
var once sync.Once

// GetURLQueue returns the singleton instance of the URLQueue.
func GetURLQueue() *URLQueue {
    once.Do(func() {
        queueInstance = &URLQueue{urls: make(chan string, 100)}
    })
    return queueInstance
}

func (q *URLQueue) Enqueue(url string) {
    q.urls <- url
}

func (q *URLQueue) Dequeue() (string, bool) {
    url, open := <-q.urls
    return url, open
}
```

### 2. Factory Pattern for HTTPClientFactory

Suppose we have different needs for HTTP clients:

- StandardClient: A default client for typical crawling.
- ThrottledClient: A client that limits request rates to respect politeness policies.
- CustomHeaderClient: A client with custom headers for sites requiring specific headers (like user-agent strings).

```go
package main

import (
	"fmt"
	"net/http"
	"time"
)

// HTTPClient is the interface that all concrete clients must implement.
type HTTPClient interface {
	DoRequest(url string) (*http.Response, error)
}
```

```golang
// StandardClient is a default HTTP client with basic settings.
type StandardClient struct {
	client *http.Client
}

func (c *StandardClient) DoRequest(url string) (*http.Response, error) {
	return c.client.Get(url)
}

// ThrottledClient enforces a delay between requests to respect politeness policy.
type ThrottledClient struct {
	client *http.Client
	delay  time.Duration
}

func (c *ThrottledClient) DoRequest(url string) (*http.Response, error) {
	time.Sleep(c.delay) // Enforce delay
	return c.client.Get(url)
}

// CustomHeaderClient allows setting custom headers for specific requirements.
type CustomHeaderClient struct {
	client  *http.Client
	headers map[string]string
}

func (c *CustomHeaderClient) DoRequest(url string) (*http.Response, error) {
	req, err := http.NewRequest("GET", url, nil)
	if err != nil {
		return nil, err
	}
	for key, value := range c.headers {
		req.Header.Set(key, value)
	}
	return c.client.Do(req)
}
```

```golang
type HTTPClientFactory struct{}

// NewHTTPClient creates and returns an HTTPClient based on the specified type.
func (f *HTTPClientFactory) NewHTTPClient(clientType string, params ...interface{}) HTTPClient {
	switch clientType {
	case "standard":
		return &StandardClient{
			client: &http.Client{Timeout: 10 * time.Second},
		}
	case "throttled":
		delay, ok := params[0].(time.Duration)
		if !ok {
			delay = 2 * time.Second // Default delay
		}
		return &ThrottledClient{
			client: &http.Client{Timeout: 10 * time.Second},
			delay:  delay,
		}
	case "customHeader":
		headers, ok := params[0].(map[string]string)
		if !ok {
			headers = map[string]string{"User-Agent": "MyCrawlerBot"}
		}
		return &CustomHeaderClient{
			client:  &http.Client{Timeout: 10 * time.Second},
			headers: headers,
		}
	default:
		return &StandardClient{
			client: &http.Client{Timeout: 10 * time.Second},
		}
	}
}
```

```golang
func main() {
	factory := &HTTPClientFactory{}

	// Create a standard client
	client := factory.NewHTTPClient("standard")
	resp, _ := client.DoRequest("http://example.com")
	fmt.Printf("Standard Client: %v\n", resp.Status)

	// Create a throttled client with a 3-second delay
	throttledClient := factory.NewHTTPClient("throttled", 3*time.Second)
	resp, _ = throttledClient.DoRequest("http://example.com")
	fmt.Printf("Throttled Client: %v\n", resp.Status)

	// Create a custom header client
	headers := map[string]string{"User-Agent": "MyCustomBot"}
	customClient := factory.NewHTTPClient("customHeader", headers)
	resp, _ = customClient.DoRequest("http://example.com")
	fmt.Printf("Custom Header Client: %v\n", resp.Status)
}
```




