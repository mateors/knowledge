# Comprehensive Guide to Bloom Filters in Go (Golang)

A **Bloom filter** is a highly space-efficient, probabilistic data structure used to test whether an element is a member of a set. It is engineered to tell you lightning-fast whether an item is **definitely not** in a set, or if it **might be** in the set.

---

## 1. What is a Bloom Filter?

Invented by Burton Howard Bloom in 1970, a Bloom filter trades absolute mathematical certainty for dramatic space savings. 

### Core Properties:
*   **Zero False Negatives:** If the filter tells you an item is **not** in the set, it is 100% correct. The item is absolutely absent.
*   **Possible False Positives:** If the filter tells you an item **is** in the set, there is a small, configurable probability that it is lying. The item *might* be there, or another combination of items could have flipped the same tracking bits.
*   **No Data Storage:** It never saves the actual string, integer, or object you insert. It only records a bitwise footprint.
*   **No Deletions:** In a standard Bloom filter, you cannot delete an element. Clearing bits to remove one item might accidentally clear bits shared by other items.

### The Trade-off Matrix:
*   **Time Complexity:** $O(k)$ for both inserts and lookups (where $k$ is the number of hash functions). This scales as a blazing-fast constant time operation, completely independent of how many millions of items are stored.
*   **Space Complexity:** Completely decoupled from the size of the original data payload. It requires only a tiny fraction of the memory that a standard Hash Set or SQL Index would demand.

---

## 2. How It Works Under the Hood

A Bloom filter consists of an underlying **Bit Array** of size $m$ (initially filled with `0`s) and a set of $k$ independent, uniform **Hash Functions**.

```
Initial State (Bit Array of size m=8, all 0s):
[ 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 ]
  0   1   2   3   4   5   6   7
```

### The Insertion Workflow:
1. An element arrives (e.g., `"user@example.com"`).
2. The element is passed through all $k$ independent hash functions.
3. Each hash function generates an integer index matching the bounds of the array ($hash \pmod m$).
4. The bits located at all of those calculated positions are flipped to `1`.

```
Example: Inserting "user@example.com" with k=3 hash functions:
- Hash1("user@example.com") % 8 = 1
- Hash2("user@example.com") % 8 = 4
- Hash3("user@example.com") % 8 = 7

Resulting Bit Array:
[ 0 | 1 | 0 | 0 | 1 | 0 | 0 | 1 ]
  0   1   2   3   4   5   6   7
```

### The Querying Workflow:
1. When checking if an element exists, it is pushed through the exact same $k$ hash functions.
2. The system checks the values at those specific array indexes:
    *   If **ANY** of the bits are `0`, the element **definitely does not exist** (if it had been inserted, all those bits would be `1`).
    *   If **ALL** of the bits are `1`, the element **might exist**.

```
Example Query A: Checking "new_user@test.com"
- Hashes point to indexes: 2, 4, 5
- Index 2 is '0'. 
- Conclusion: DEFINITELY NOT in the set. (100% accurate)

Example Query B: Checking "attacker@spam.com" (Never actually inserted!)
- Hashes point to indexes: 1, 4, 7
- Index 1, 4, and 7 are all '1' (due to the footprint left by "user@example.com").
- Conclusion: MIGHT BE in the set. This is a False Positive.
```

---

## 3. Why and Where Should You Use It?

Bloom filters function as an ultra-fast, low-cost **"shield"** or **"gatekeeper"** placed upstream of expensive operations (like disk I/O, heavy network requests, or costly API calls).

### Primary Production Use Cases:
*   **Database Read Protection (LSM Trees):** Systems like **Apache Cassandra, RocksDB, and Google Bigtable** build Bloom filters for SSTables. Before performing a slow, multi-millisecond disk seek to find a row key, the database checks an in-memory Bloom filter. If it returns false, the database skips the disk read entirely.
*   **Web Cache Protection (CDN Optimization):** Services like **Akamai** use Bloom filters to prevent "one-hit wonders" from polluting web caches. They track requested URLs; a page is only cached if it is requested a second time (detected via the filter).
*   **Malicious URL/Spam Filtering:** Web browsers (like Google Chrome's Safe Browsing feature) store a local, compact Bloom filter of millions of known malicious URLs. If a user visits a link and the local filter says "not malicious," the page loads instantly. If it flags a potential match, the browser makes a secure API network call to verify the full threat database.
*   **Crypto and Blockchain Nodes:** Simplified Payment Verification (SPV) clients in Bitcoin use Bloom filters to download only transactions relevant to their specific wallet address without syncing the entire heavy blockchain ledger.

---

## 4. Production Go Implementation

Below is a complete, modular implementation of a Bloom filter in Go using the highly efficient **MurmurHash3** algorithm (non-cryptographic, optimized for speed and uniform bit distribution).

### Step 1: Install Dependencies
We use a high-performance native Go implementation of MurmurHash3:
```bash
go get github.com/spaolacci/murmur3
```

### Step 2: Write the Core Engine (`bloom.go`)

```go
package main

import (
	"encoding/binary"
	"fmt"
	"math"
	"strings"

	"github.com/spaolacci/murmur3"
)

// BloomFilter represents the memory-efficient probabilistic data structure
type BloomFilter struct {
	bitArray []bool // Underlying bit array (using bool for simplicity; use bit-shifting uint64 for raw memory optimization)
	m        uint32 // Size of the bit array
	k        uint32 // Number of hash functions
}

// NewBloomFilter calculates the optimal size and builds the filter based on target metrics
// n: Expected number of items to insert
// p: Desired false positive probability rate (e.g., 0.01 for 1%)
func NewBloomFilter(n uint32, p float64) *BloomFilter {
	// Formula for optimal bit array size: m = - (n * ln(p)) / (ln(2)^2)
	m := uint32(math.Ceil(-float64(n) * math.Log(p) / math.Pow(math.Log(2), 2)))

	// Formula for optimal number of hash functions: k = (m / n) * ln(2)
	k := uint32(math.Ceil(float64(m) / float64(n) * math.Log(2)))

	// Ensure boundary minimums
	if m == 0 { m = 1 }
	if k == 0 { k = 1 }

	return &BloomFilter{
		bitArray: make([]bool, m),
		m:        m,
		k:        k,
	}
}

// getHashes uses a single MurmurHash3 execution to generate 'k' unique index locations 
// via the Kirsch-Mitzenmacher optimization technique: Hash(i) = HashA + i * HashB
func (bf *BloomFilter) getHashes(data []byte) []uint32 {
	hasher := murmur3.New128()
	hasher.Write(data)
	hashA64, hashB64 := hasher.Sum128()

	// Extract two 32-bit components from the 128-bit hash output
	hashA := uint32(hashA64 & 0xFFFFFFFF)
	hashB := uint32(hashB64 & 0xFFFFFFFF)

	positions := make([]uint32, bf.k)
	for i := uint32(0); i < bf.k; i++ {
		// Kirsch-Mitzenmacher sequence mapped inside array boundaries
		positions[i] = (hashA + i*hashB) % bf.m
	}
	return positions
}

// Add inserts an item data footprint into the filter
func (bf *BloomFilter) Add(item string) {
	cleanItem := []byte(strings.ToLower(strings.TrimSpace(item)))
	positions := bf.getHashes(cleanItem)

	for _, pos := range positions {
		bf.bitArray[pos] = true
	}
}

// Contains checks if the item footprint exists within the filter
// Returns false (100% true absence) or true (probabilistic presence)
func (bf *BloomFilter) Contains(item string) bool {
	cleanItem := []byte(strings.ToLower(strings.TrimSpace(item)))
	positions := bf.getHashes(cleanItem)

	for _, pos := range positions {
		// If even ONE bit is missing (false), the item definitely was never added
		if !bf.bitArray[pos] {
			return false
		}
	}
	return true
}

func main() {
	// Real-world scenario setup:
	// We expect to store 10,000 bad spam emails, and want a low 1% (0.01) false positive rate.
	expectedItems := uint32(10000)
	falsePositiveRate := 0.01

	filter := NewBloomFilter(expectedItems, falsePositiveRate)
	fmt.Printf("--- Bloom Filter Initialized ---\n")
	fmt.Printf("Bit Array Size (m): %d bits\n", filter.m)
	fmt.Printf("Hash Functions (k): %d distinct steps\n\n", filter.k)

	// Use Case: Populate the filter with flagged spam/malicious email addresses
	spamDatabase := []string{
		"spammer@malicious.com",
		"phishing-attack@scam-site.net",
		"fake-invoice@wire-transfer.org",
		"win-free-cash@jackpot.xyz",
	}

	fmt.Println("Populating Bloom Filter shield with known spam senders...")
	for _, email := range spamDatabase {
		filter.Add(email)
	}
	fmt.Println("Population complete.\n")

	// --- TEST CASE 1: Querying a known threat ---
	testEmail1 := "phishing-attack@scam-site.net"
	fmt.Printf("Testing Item 1: '%s'\n", testEmail1)
	if filter.Contains(testEmail1) {
		fmt.Println("--> Result: MIGHT BE spam. (Rerouting to comprehensive database check...)")
	} else {
		fmt.Println("--> Result: DEFINITELY NOT spam. (Safe to proceed instantly!)")
	}
	fmt.Println()

	// --- TEST CASE 2: Querying a completely clean, new user ---
	testEmail2 := "clean.developer@mateors.com"
	fmt.Printf("Testing Item 2: '%s'\n", testEmail2)
	if filter.Contains(testEmail2) {
		fmt.Println("--> Result: MIGHT BE spam. (Rerouting to comprehensive database check...)")
	} else {
		fmt.Println("--> Result: DEFINITELY NOT spam. (Safe to proceed instantly!)")
	}
}
```

---

## 5. Architectural Considerations

When deploying a Bloom filter into a real system, you must design around these fundamental mathematical laws:

1.  **Sizing is Fixed:** You cannot dynamically grow a standard Bloom filter. If you under-size the bit array ($m$) and insert more items than expected ($n$), the array fills up with `1`s. As density increases, the false positive rate rapidly climbs toward 100%.
2.  **Choosing the Right Hashing Algorithm:** Cryptographic hashes like SHA-256 or MD5 are unnecessarily CPU-heavy for this use case. Instead, choose fast, non-cryptographic hashes like **MurmurHash3, CityHash, or FNV-1a**.
3.  **The Deletion Fix (Counting Bloom Filters):** If your application *must* support deletions, look into a **Counting Bloom Filter**. Instead of a simple boolean array of single bits (`0` or `1`), it uses an array of small integer counters (e.g., 4 bits per bucket). When an item is added, the counters are incremented; when deleted, they are decremented.
